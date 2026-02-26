# Web Scraping & Data Extraction Reference

## The Golden Rules of Puppeteer Scraping

1. **Never return DOM elements from `evaluate()`** — they don't serialize across the JS bridge
2. **Always type your `evaluate()` calls** — `page.evaluate<ReturnType>(() => ...)`
3. **Use `waitForSelector` before accessing elements** — the page might not be ready
4. **Block unnecessary resources** — images/fonts/media slow down scraping massively
5. **Close pages in finally blocks** — prevent memory leaks on errors

---

## Basic Page Scraping Pattern

```typescript
import puppeteer, { Page } from 'puppeteer';

interface ProductData {
    title: string;
    price: string;
    rating: number | null;
    imageUrl: string | null;
}

async function scrapeProduct(url: string): Promise<ProductData> {
    const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
    const page = await browser.newPage();

    try {
        // Block images for speed (remove if you need image URLs from DOM inspection)
        await page.setRequestInterception(true);
        page.on('request', (req) => {
            if (['image', 'font', 'media', 'stylesheet'].includes(req.resourceType())) {
                req.abort();
            } else {
                req.continue();
            }
        });

        await page.goto(url, { waitUntil: 'domcontentloaded', timeout: 30_000 });
        await page.waitForSelector('.product-title', { visible: true, timeout: 10_000 });

        // Extract all data in a single evaluate call (reduces bridge overhead)
        const data = await page.evaluate<ProductData>(() => {
            const getText = (selector: string): string => document.querySelector(selector)?.textContent?.trim() ?? '';

            const ratingText = document.querySelector('.rating')?.getAttribute('data-value');

            return {
                title: getText('.product-title'),
                price: getText('.product-price'),
                rating: ratingText ? parseFloat(ratingText) : null,
                imageUrl: document.querySelector<HTMLImageElement>('.product-image')?.src ?? null,
            };
        });

        return data;
    } finally {
        await page.close();
        await browser.close();
    }
}
```

---

## Scraping Multiple Pages (Pagination)

```typescript
async function scrapeAllPages(baseUrl: string): Promise<ProductData[]> {
    const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
    const page = await browser.newPage();
    const allResults: ProductData[] = [];

    try {
        let currentPage = 1;
        let hasNextPage = true;

        while (hasNextPage) {
            await page.goto(`${baseUrl}?page=${currentPage}`, {
                waitUntil: 'networkidle2',
                timeout: 30_000,
            });

            // Scrape current page items
            const items = await page.evaluate<ProductData[]>(() => {
                return Array.from(document.querySelectorAll('.product-card')).map((el) => ({
                    title: el.querySelector('.title')?.textContent?.trim() ?? '',
                    price: el.querySelector('.price')?.textContent?.trim() ?? '',
                    rating: null,
                    imageUrl: el.querySelector<HTMLImageElement>('img')?.src ?? null,
                }));
            });

            allResults.push(...items);

            // Check if next page exists
            const nextButton = await page.$('.pagination-next:not(.disabled)');
            hasNextPage = nextButton !== null;
            currentPage++;

            // Polite delay between pages
            if (hasNextPage) {
                await new Promise((r) => setTimeout(r, 1000 + Math.random() * 1000));
            }
        }
    } finally {
        await page.close();
        await browser.close();
    }

    return allResults;
}
```

---

## Handling Dynamic/SPA Content

```typescript
async function scrapeSPA(url: string) {
    const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
    const page = await browser.newPage();

    try {
        // Intercept API responses for direct data extraction
        const apiData: unknown[] = [];

        page.on('response', async (response) => {
            const url = response.url();
            if (url.includes('/api/products') && response.status() === 200) {
                try {
                    const json = await response.json();
                    apiData.push(json);
                } catch {
                    // Not JSON, ignore
                }
            }
        });

        await page.goto(url, { waitUntil: 'networkidle0', timeout: 30_000 });

        // Wait for dynamic content to render
        await page.waitForFunction(() => document.querySelectorAll('.product-item').length > 0, { timeout: 15_000 });

        // Scroll to trigger lazy loading
        await autoScroll(page);

        const items = await page.evaluate<string[]>(() => Array.from(document.querySelectorAll('.product-item h2')).map((el) => el.textContent?.trim() ?? ''));

        return { domItems: items, apiItems: apiData };
    } finally {
        await page.close();
        await browser.close();
    }
}

// Auto-scroll to bottom to trigger lazy loading
async function autoScroll(page: Page): Promise<void> {
    await page.evaluate(async () => {
        await new Promise<void>((resolve) => {
            let totalHeight = 0;
            const distance = 300;
            const timer = setInterval(() => {
                window.scrollBy(0, distance);
                totalHeight += distance;
                if (totalHeight >= document.body.scrollHeight) {
                    clearInterval(timer);
                    resolve();
                }
            }, 100);
        });
    });
}
```

---

## Table Data Extraction

```typescript
interface TableRow {
    [key: string]: string;
}

async function scrapeTable(page: Page, tableSelector: string): Promise<TableRow[]> {
    return page.evaluate<TableRow[]>((selector) => {
        const table = document.querySelector(selector);
        if (!table) return [];

        const headers = Array.from(table.querySelectorAll('thead th')).map((th) => th.textContent?.trim() ?? '');

        return Array.from(table.querySelectorAll('tbody tr')).map((row) => {
            const cells = Array.from(row.querySelectorAll('td'));
            return Object.fromEntries(headers.map((header, i) => [header, cells[i]?.textContent?.trim() ?? '']));
        });
    }, tableSelector);
}
```

---

## Waiting Strategies Cheatsheet

```typescript
// Wait for element to appear
await page.waitForSelector('.spinner', { hidden: true }); // wait for spinner to disappear
await page.waitForSelector('.content', { visible: true }); // wait for content to appear

// Wait for navigation after action
await Promise.all([page.waitForNavigation({ waitUntil: 'domcontentloaded' }), page.click('.submit-btn')]);

// Wait for a custom condition
await page.waitForFunction(
    (minCount) => document.querySelectorAll('.item').length >= minCount,
    { timeout: 10_000 },
    5, // argument passed to the browser function
);

// Wait for URL change
await page.waitForFunction((expectedUrl) => window.location.href.includes(expectedUrl), {}, '/dashboard');

// Wait for network request/response
const [response] = await Promise.all([page.waitForResponse((res) => res.url().includes('/api/data') && res.status() === 200), page.click('.load-data-btn')]);
const data = await response.json();
```

---

## Extracting from iframes

```typescript
async function scrapeIframe(page: Page, iframeSelector: string): Promise<string> {
    const frameHandle = await page.waitForSelector(iframeSelector);
    const frame = await frameHandle?.contentFrame();

    if (!frame) throw new Error('Could not get iframe content frame');

    await frame.waitForSelector('.target-content');

    return frame.evaluate<string>(() => document.querySelector('.target-content')?.textContent?.trim() ?? '');
}
```
