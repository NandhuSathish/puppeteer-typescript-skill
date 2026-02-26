# Testing Patterns Reference

## Puppeteer for E2E Testing

Puppeteer is great for:

- Visual regression testing (screenshot comparison)
- E2E flows (login → do thing → verify result)
- Testing JavaScript-heavy SPAs that unit tests can't reach
- Form submission flows
- File download testing

> For full test suites, consider **Jest + Puppeteer** or **Playwright** (has built-in test runner).
> Puppeteer alone is best for targeted automation tests.

---

## Jest + Puppeteer Setup

```bash
npm install --save-dev jest jest-puppeteer @types/jest @types/puppeteer-jest
```

```javascript
// jest.config.js
module.exports = {
    preset: 'jest-puppeteer',
    testMatch: ['**/*.e2e.ts'],
    transform: { '^.+\\.ts$': 'ts-jest' },
};
```

```javascript
// jest-puppeteer.config.js
module.exports = {
    launch: {
        headless: true,
        args: ['--no-sandbox', '--disable-setuid-sandbox'],
    },
    server: {
        command: 'npm run dev', // Start your dev server
        port: 3000,
        launchTimeout: 30_000,
    },
};
```

---

## Page Object Pattern (POM)

Avoid brittle selectors scattered everywhere — centralize them:

```typescript
// pages/LoginPage.ts
import { Page } from 'puppeteer';

export class LoginPage {
    private readonly selectors = {
        emailInput: '#email',
        passwordInput: '#password',
        submitButton: '[type="submit"]',
        errorMessage: '.login-error',
        successRedirect: '/dashboard',
    } as const;

    constructor(private readonly page: Page) {}

    async navigate(): Promise<void> {
        await this.page.goto('https://example.com/login', {
            waitUntil: 'domcontentloaded',
        });
    }

    async login(email: string, password: string): Promise<void> {
        await this.page.type(this.selectors.emailInput, email);
        await this.page.type(this.selectors.passwordInput, password);
        await Promise.all([this.page.waitForNavigation({ waitUntil: 'networkidle0' }), this.page.click(this.selectors.submitButton)]);
    }

    async getErrorMessage(): Promise<string | null> {
        const el = await this.page.$(this.selectors.errorMessage);
        if (!el) return null;
        return el.evaluate((e) => e.textContent?.trim() ?? null);
    }

    async isLoggedIn(): Promise<boolean> {
        return this.page.url().includes(this.selectors.successRedirect);
    }
}

// Test using POM
describe('Login', () => {
    let loginPage: LoginPage;

    beforeEach(async () => {
        loginPage = new LoginPage(page); // `page` provided by jest-puppeteer
        await loginPage.navigate();
    });

    it('should log in with valid credentials', async () => {
        await loginPage.login('user@example.com', 'correctpassword');
        expect(await loginPage.isLoggedIn()).toBe(true);
    });

    it('should show error with invalid credentials', async () => {
        await loginPage.login('user@example.com', 'wrongpassword');
        const error = await loginPage.getErrorMessage();
        expect(error).toContain('Invalid credentials');
    });
});
```

---

## Visual Regression Testing

```typescript
import * as fs from 'fs/promises';
import * as path from 'path';

const SNAPSHOTS_DIR = path.join(__dirname, '__snapshots__');

async function captureSnapshot(page: Page, name: string): Promise<Buffer> {
    return (await page.screenshot({ fullPage: true, type: 'png' })) as Buffer;
}

async function compareWithBaseline(name: string, current: Buffer): Promise<{ matches: boolean; diffPath?: string }> {
    const baselinePath = path.join(SNAPSHOTS_DIR, `${name}.png`);

    try {
        const baseline = await fs.readFile(baselinePath);
        const matches = baseline.equals(current);

        if (!matches) {
            const diffPath = path.join(SNAPSHOTS_DIR, `${name}.diff.png`);
            await fs.writeFile(diffPath, current);
            return { matches: false, diffPath };
        }
        return { matches: true };
    } catch {
        // Baseline doesn't exist — create it
        await fs.mkdir(SNAPSHOTS_DIR, { recursive: true });
        await fs.writeFile(baselinePath, current);
        console.log(`Created baseline snapshot: ${name}`);
        return { matches: true };
    }
}

// Usage in test
it('homepage should match visual snapshot', async () => {
    await page.goto('https://example.com', { waitUntil: 'networkidle0' });
    const screenshot = await captureSnapshot(page, 'homepage');
    const result = await compareWithBaseline('homepage', screenshot);
    expect(result.matches).toBe(true);
});
```

---

## Testing File Downloads

```typescript
import * as path from 'path';
import * as fs from 'fs/promises';

async function testFileDownload(page: Page, downloadTriggerSelector: string, expectedFilename: string): Promise<string> {
    const downloadDir = path.join(__dirname, 'downloads');
    await fs.mkdir(downloadDir, { recursive: true });

    // Set download behavior
    const client = await page.target().createCDPSession();
    await client.send('Page.setDownloadBehavior', {
        behavior: 'allow',
        downloadPath: downloadDir,
    });

    await page.click(downloadTriggerSelector);

    // Wait for file to appear
    const filePath = path.join(downloadDir, expectedFilename);
    await waitForFile(filePath, 15_000);

    return filePath;
}

async function waitForFile(filePath: string, timeoutMs: number): Promise<void> {
    const start = Date.now();
    while (Date.now() - start < timeoutMs) {
        try {
            await fs.access(filePath);
            return; // File exists
        } catch {
            await new Promise((r) => setTimeout(r, 500));
        }
    }
    throw new Error(`File not downloaded in ${timeoutMs}ms: ${filePath}`);
}
```

---

## Network Mocking for Tests

```typescript
// Mock API responses during tests
async function mockApiResponse(page: Page, urlPattern: string, mockData: unknown): Promise<void> {
    await page.setRequestInterception(true);

    page.on('request', (request) => {
        if (request.url().includes(urlPattern)) {
            request.respond({
                status: 200,
                contentType: 'application/json',
                body: JSON.stringify(mockData),
            });
        } else {
            request.continue();
        }
    });
}

// Usage in test
it('should display products from API', async () => {
    await mockApiResponse(page, '/api/products', [{ id: 1, name: 'Test Product', price: 9.99 }]);

    await page.goto('http://localhost:3000/products');
    await page.waitForSelector('.product-card');

    const count = await page.$$eval('.product-card', (els) => els.length);
    expect(count).toBe(1);
});
```

---

## Performance Benchmarking

```typescript
// Measure Core Web Vitals using Puppeteer
async function measurePerformance(url: string) {
    const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
    const page = await browser.newPage();

    try {
        await page.goto(url, { waitUntil: 'networkidle0' });

        const metrics = await page.metrics();
        const timing = await page.evaluate(() => {
            const t = performance.timing;
            return {
                TTFB: t.responseStart - t.fetchStart,
                DOMContentLoaded: t.domContentLoadedEventEnd - t.fetchStart,
                PageLoad: t.loadEventEnd - t.fetchStart,
                FCP: performance.getEntriesByName('first-contentful-paint')[0]?.startTime ?? null,
            };
        });

        return { ...timing, JSHeapUsedMB: metrics.JSHeapUsedSize / 1024 / 1024 };
    } finally {
        await page.close();
        await browser.close();
    }
}
```
