# Error Handling & Retries Reference

## Common Puppeteer Errors & Their Fixes

| Error                                       | Cause                            | Fix                                                 |
| ------------------------------------------- | -------------------------------- | --------------------------------------------------- |
| `TimeoutError: waiting for selector`        | Element didn't appear in time    | Increase timeout, check selector, verify page state |
| `Error: Node is detached from document`     | DOM changed after getting handle | Re-query the element                                |
| `ProtocolError: Target closed`              | Browser crashed or page closed   | Check for crashes, use try/finally                  |
| `Error: net::ERR_ABORTED`                   | Request was blocked/aborted      | Check request interception logic                    |
| `Error: Navigation timeout exceeded`        | Page load too slow               | Increase timeout, use `domcontentloaded`            |
| `UnhandledPromiseRejection: Protocol error` | CDP connection dropped           | Restart browser, use reconnect logic                |

---

## Core Error Handling Pattern

```typescript
import puppeteer, { Browser, Page, TimeoutError } from 'puppeteer';

// Typed error categories
class ScraperError extends Error {
    constructor(
        message: string,
        public readonly url: string,
        public readonly cause?: Error,
    ) {
        super(message);
        this.name = 'ScraperError';
    }
}

class NavigationError extends ScraperError {
    constructor(url: string, cause?: Error) {
        super(`Failed to navigate to ${url}`, url, cause);
        this.name = 'NavigationError';
    }
}

class ExtractionError extends ScraperError {
    constructor(url: string, selector: string, cause?: Error) {
        super(`Failed to extract data from ${selector} on ${url}`, url, cause);
        this.name = 'ExtractionError';
    }
}
```

---

## Retry with Exponential Backoff

```typescript
interface RetryOptions {
    maxAttempts?: number;
    initialDelayMs?: number;
    backoffMultiplier?: number;
    maxDelayMs?: number;
    retryOn?: (error: Error) => boolean;
}

async function withRetry<T>(fn: () => Promise<T>, options: RetryOptions = {}): Promise<T> {
    const {
        maxAttempts = 3,
        initialDelayMs = 1000,
        backoffMultiplier = 2,
        maxDelayMs = 30_000,
        retryOn = () => true, // retry on all errors by default
    } = options;

    let lastError: Error | undefined;

    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
            return await fn();
        } catch (error) {
            lastError = error instanceof Error ? error : new Error(String(error));

            const shouldRetry = retryOn(lastError);
            const isLastAttempt = attempt === maxAttempts;

            if (!shouldRetry || isLastAttempt) {
                throw lastError;
            }

            const delay = Math.min(initialDelayMs * Math.pow(backoffMultiplier, attempt - 1), maxDelayMs);

            console.warn(`Attempt ${attempt}/${maxAttempts} failed: ${lastError.message}. Retrying in ${delay}ms...`);
            await new Promise((r) => setTimeout(r, delay));
        }
    }

    throw lastError;
}

// Usage
const data = await withRetry(() => scrapePage(url), {
    maxAttempts: 3,
    initialDelayMs: 2000,
    retryOn: (err) => {
        // Don't retry on 404s or auth errors
        if (err.message.includes('ERR_NAME_NOT_RESOLVED')) return false;
        if (err.message.includes('403')) return false;
        return true; // retry on timeouts, network errors, etc.
    },
});
```

---

## Page-Level Error Handling

```typescript
async function safeScrape(page: Page, url: string): Promise<string | null> {
    try {
        // Catch page crashes
        page.on('error', (err) => {
            console.error('Page crashed:', err);
        });

        // Catch unhandled page errors (JS errors in browser context)
        page.on('pageerror', (err) => {
            console.warn('Page JS error (non-fatal):', err.message);
        });

        const response = await page.goto(url, {
            waitUntil: 'domcontentloaded',
            timeout: 30_000,
        });

        if (!response) {
            throw new NavigationError(url, new Error('No response received'));
        }

        const status = response.status();
        if (status === 404) {
            console.warn(`Page not found: ${url}`);
            return null;
        }
        if (status >= 400) {
            throw new NavigationError(url, new Error(`HTTP ${status}`));
        }

        // Safe selector access with fallback
        const title = await page
            .waitForSelector('h1', { timeout: 5_000 })
            .then((el) => el?.evaluate((e) => e.textContent?.trim() ?? ''))
            .catch(() => null); // Return null if h1 not found, don't throw

        return title;
    } catch (error) {
        if (error instanceof TimeoutError) {
            throw new NavigationError(url, error);
        }
        throw error;
    }
}
```

---

## Browser Crash Recovery

```typescript
class ResilientBrowser {
    private browser: Browser | null = null;
    private readonly maxRestarts = 3;
    private restartCount = 0;

    async getBrowser(): Promise<Browser> {
        if (!this.browser || !this.browser.connected) {
            if (this.restartCount >= this.maxRestarts) {
                throw new Error(`Browser restarted too many times (${this.maxRestarts})`);
            }
            this.browser = await puppeteer.launch({
                headless: true,
                args: ['--no-sandbox', '--disable-dev-shm-usage'],
            });
            this.restartCount++;

            // Auto-restart on browser disconnection
            this.browser.on('disconnected', () => {
                console.warn('Browser disconnected, will restart on next request');
                this.browser = null;
            });
        }
        return this.browser;
    }

    async close(): Promise<void> {
        await this.browser?.close();
        this.browser = null;
    }
}

// Usage
const resilientBrowser = new ResilientBrowser();

async function scrapeWithRecovery(url: string): Promise<string> {
    return withRetry(async () => {
        const browser = await resilientBrowser.getBrowser();
        const page = await browser.newPage();
        try {
            await page.goto(url, { waitUntil: 'domcontentloaded' });
            return await page.title();
        } finally {
            await page.close();
        }
    });
}
```

---

## Timeout Configuration

```typescript
// Per-operation timeouts — don't rely on defaults (30s is often wrong)
const TIMEOUTS = {
    navigation: 30_000, // page.goto()
    elementVisible: 10_000, // waitForSelector
    networkIdle: 15_000, // waitForNetworkIdle
    evaluation: 5_000, // page.evaluate
} as const;

// Set default timeout for the page (overrides Puppeteer's 30s default)
page.setDefaultTimeout(TIMEOUTS.navigation);
page.setDefaultNavigationTimeout(TIMEOUTS.navigation);

// Override per-call
await page.waitForSelector('.modal', { timeout: 5_000 });
```

---

## Graceful Shutdown

```typescript
// Always handle process signals to cleanup browsers
process.on('SIGINT', async () => {
    console.log('Shutting down...');
    await browser?.close();
    process.exit(0);
});

process.on('SIGTERM', async () => {
    await browser?.close();
    process.exit(0);
});

process.on('uncaughtException', async (error) => {
    console.error('Uncaught exception:', error);
    await browser?.close();
    process.exit(1);
});
```
