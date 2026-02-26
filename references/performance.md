# Performance & Browser Pooling Reference

## Resource Consumption Reality Check

- Single Chromium instance: **200–400MB RAM**
- Each additional page: **~50–150MB RAM**
- Browser startup time: **500ms–2s**
- Rule of thumb: **max 5 browser instances on a 4-core machine**

---

## Request Interception (Single Biggest Speed Win)

Blocking images/fonts/media can cut page load time by **50–80%** for scraping tasks:

```typescript
import { HTTPRequest, Page } from 'puppeteer';

type ResourceType = 'image' | 'stylesheet' | 'font' | 'media' | 'other';

// Scraping mode — block everything non-essential
async function enableScrapingMode(page: Page): Promise<void> {
    await page.setRequestInterception(true);

    const BLOCKED: ResourceType[] = ['image', 'stylesheet', 'font', 'media'];

    page.on('request', (request: HTTPRequest) => {
        if (BLOCKED.includes(request.resourceType() as ResourceType)) {
            request.abort();
        } else {
            request.continue();
        }
    });
}

// Selective mode — block ads and trackers but allow images
async function enableSelectiveMode(page: Page): Promise<void> {
    await page.setRequestInterception(true);

    const BLOCKED_DOMAINS = ['google-analytics.com', 'googletagmanager.com', 'facebook.net', 'doubleclick.net', 'hotjar.com', 'intercom.io'];

    page.on('request', (request: HTTPRequest) => {
        const url = request.url();
        const isBlocked = BLOCKED_DOMAINS.some((domain) => url.includes(domain));
        isBlocked ? request.abort() : request.continue();
    });
}

// PDF/Screenshot mode — allow everything (need styles and images)
async function enableRenderMode(page: Page): Promise<void> {
    // Don't intercept at all — let everything load for accurate rendering
    await page.setRequestInterception(false);
}
```

---

## Browser Reuse (Single Browser, Multiple Pages)

For sequential tasks — the simplest performance upgrade:

```typescript
import puppeteer, { Browser, Page } from 'puppeteer';

class BrowserSession {
    private browser: Browser | null = null;

    async start(): Promise<void> {
        this.browser = await puppeteer.launch({
            headless: true,
            args: ['--no-sandbox', '--disable-setuid-sandbox', '--disable-dev-shm-usage'],
        });
    }

    async runOnPage<T>(fn: (page: Page) => Promise<T>): Promise<T> {
        if (!this.browser) throw new Error('Browser not started. Call start() first.');
        const page = await this.browser.newPage();
        try {
            return await fn(page);
        } finally {
            await page.close();
        }
    }

    async close(): Promise<void> {
        await this.browser?.close();
        this.browser = null;
    }
}

// Usage — one browser startup for many pages
const session = new BrowserSession();
await session.start();

try {
    const results = await Promise.all(
        urls.map((url) =>
            session.runOnPage(async (page) => {
                await page.goto(url);
                return page.title();
            }),
        ),
    );
} finally {
    await session.close();
}
```

---

## Puppeteer Cluster (Parallel Scraping)

For high-volume tasks requiring parallelism:

```typescript
import { Cluster } from 'puppeteer-cluster';

interface ScrapeTask {
    url: string;
}

interface ScrapeResult {
    url: string;
    title: string;
    links: string[];
}

async function runCluster(urls: string[]): Promise<ScrapeResult[]> {
    const cluster = await Cluster.launch({
        concurrency: Cluster.CONCURRENCY_PAGE, // Reuse browser, new page per task
        maxConcurrency: 5, // Max 5 parallel pages
        timeout: 60_000,
        retryLimit: 2,
        retryDelay: 1000,
        puppeteerOptions: {
            headless: true,
            args: ['--no-sandbox', '--disable-setuid-sandbox', '--disable-dev-shm-usage', '--disable-gpu'],
        },
        monitor: process.env.DEBUG === 'cluster', // Log progress in debug mode
    });

    // Define the task handler once
    await cluster.task(async ({ page, data: { url } }: { page: any; data: ScrapeTask }) => {
        await page.setRequestInterception(true);
        page.on('request', (req: any) => {
            ['image', 'font', 'media'].includes(req.resourceType()) ? req.abort() : req.continue();
        });

        await page.goto(url, { waitUntil: 'domcontentloaded', timeout: 30_000 });

        return page.evaluate<ScrapeResult>(
            (pageUrl: string) => ({
                url: pageUrl,
                title: document.title,
                links: Array.from(document.querySelectorAll('a[href]'))
                    .map((a) => (a as HTMLAnchorElement).href)
                    .slice(0, 20), // Limit to 20 links
            }),
            url,
        );
    });

    // Queue all URLs
    const results: ScrapeResult[] = [];
    for (const url of urls) {
        cluster.queue({ url }, async (result: ScrapeResult) => {
            results.push(result);
        });
    }

    await cluster.idle();
    await cluster.close();

    return results;
}
```

---

## Concurrency Mode Comparison

```typescript
// Cluster.CONCURRENCY_PAGE
// - One browser, multiple pages (tabs)
// - Best for: similar sites, moderate concurrency (2-10)
// - Memory: lowest (one browser process)
// - Isolation: pages share same browser (cookies/cache can leak)

// Cluster.CONCURRENCY_CONTEXT
// - One browser, multiple incognito contexts
// - Best for: different user sessions, account switching
// - Memory: low (contexts are lighter than browsers)
// - Isolation: complete cookie/storage isolation between contexts

// Cluster.CONCURRENCY_BROWSER
// - Multiple browsers
// - Best for: strict isolation, higher concurrency, anti-detection
// - Memory: highest (one Chromium process per task)
// - Isolation: complete process isolation
```

---

## Page Performance Optimization

```typescript
async function optimizePage(page: Page): Promise<void> {
    // 1. Set viewport to reduce rendering surface
    await page.setViewport({ width: 1280, height: 720, deviceScaleFactor: 1 });

    // 2. Disable JavaScript if you only need static HTML
    // await page.setJavaScriptEnabled(false); // Only for fully static pages

    // 3. Set aggressive timeouts
    page.setDefaultTimeout(15_000);
    page.setDefaultNavigationTimeout(20_000);

    // 4. Block unnecessary resources
    await enableScrapingMode(page); // See above

    // 5. Use faster waitUntil for static pages
    // await page.goto(url, { waitUntil: 'domcontentloaded' }); // faster than 'load'
}
```

---

## Memory Monitoring

```typescript
async function monitorMemory(page: Page): Promise<void> {
    const metrics = await page.metrics();
    const heapUsedMB = metrics.JSHeapUsedSize / 1024 / 1024;
    const heapTotalMB = metrics.JSHeapTotalSize / 1024 / 1024;

    console.log(`Memory: ${heapUsedMB.toFixed(1)}MB used / ${heapTotalMB.toFixed(1)}MB total`);

    // Force garbage collection if available (requires --expose-gc flag in Node)
    if (heapUsedMB > 500) {
        await page.evaluate(() => {
            if ('gc' in window) (window as any).gc();
        });
    }
}

// Launch with GC exposed
const browser = await puppeteer.launch({
    args: ['--no-sandbox', '--js-flags=--expose-gc'],
});
```

---

## Caching Sessions (Skip Login Every Time)

```typescript
const USER_DATA_DIR = './browser-cache';

// First run: login and cache session
const browser = await puppeteer.launch({
    headless: true,
    args: ['--no-sandbox'],
    userDataDir: USER_DATA_DIR, // Cache cookies, localStorage, etc.
});

// Subsequent runs: session is restored from disk automatically
const browser2 = await puppeteer.launch({
    headless: true,
    args: ['--no-sandbox'],
    userDataDir: USER_DATA_DIR, // Already logged in!
});
```
