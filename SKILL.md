---
name: puppeteer-typescript
description: >
    Ultimate Puppeteer + TypeScript skill covering all aspects of browser automation: web scraping & data extraction,
    browser automation & testing, PDF/screenshot generation, and form filling & interactions.
    Includes TypeScript setup, anti-bot bypass techniques (stealth, fingerprinting, human-like behavior),
    robust error handling with retries, performance optimization (browser pooling, request interception,
    resource blocking), and production-ready patterns. Use this skill any time the user mentions Puppeteer,
    browser automation, headless Chrome, web scraping, screenshot generation, PDF rendering, form automation,
    crawling, or bot detection bypass — even if they don't explicitly mention TypeScript. Also trigger for
    tasks involving page interaction scripts, scraping pipelines, test automation with real browsers,
    or anything that would benefit from a controlled Chromium instance.
---

# Ultimate Puppeteer + TypeScript Skill

This skill makes Claude an expert Puppeteer + TypeScript developer who writes production-ready,
type-safe browser automation code. Always default to TypeScript with strict mode, proper cleanup,
and thoughtful error handling.

## Quick Reference — When to read which reference file

| Topic                                              | File                            |
| -------------------------------------------------- | ------------------------------- |
| Project setup, tsconfig, browser launch config     | `references/setup.md`           |
| Web scraping patterns, waitFor, data extraction    | `references/scraping.md`        |
| Anti-bot bypass, stealth, human-like behavior      | `references/stealth.md`         |
| Error handling, retries, timeouts                  | `references/error-handling.md`  |
| Performance, browser pooling, request interception | `references/performance.md`     |
| PDF generation, screenshots                        | `references/pdf-screenshots.md` |
| Form filling, interactions, file uploads           | `references/forms.md`           |
| Testing patterns with Puppeteer                    | `references/testing.md`         |

**Always read the relevant reference file(s) before writing code.** Multiple files may be needed for complex tasks.

---

## Core Principles

### TypeScript-First Rules

- **Puppeteer ships its own types** — never install `@types/puppeteer` (it's a stub and outdated)
- Use `import type { Browser, Page, ElementHandle }` for type-only imports
- Always type `page.evaluate()` return generics: `page.evaluate<string[]>(() => [...])`
- Use `ElementHandle<HTMLInputElement>` not plain `ElementHandle` for form inputs
- Return structured data types from evaluate, never raw DOM objects (they don't serialize)

### Universal Browser Setup Pattern

```typescript
import puppeteer, { Browser, Page } from 'puppeteer';

// ALWAYS use try/finally for guaranteed cleanup
async function withBrowser<T>(fn: (browser: Browser) => Promise<T>): Promise<T> {
    const browser = await puppeteer.launch(launchConfig);
    try {
        return await fn(browser);
    } finally {
        await browser.close();
    }
}

async function withPage<T>(browser: Browser, fn: (page: Page) => Promise<T>): Promise<T> {
    const page = await browser.newPage();
    try {
        return await fn(page);
    } finally {
        await page.close();
    }
}
```

### The Launch Config Baseline

```typescript
// Use this as the starting point for ALL launch configs
const launchConfig = {
    headless: true, // Use true (not 'new') for modern Puppeteer v21+
    args: [
        '--no-sandbox',
        '--disable-setuid-sandbox',
        '--disable-dev-shm-usage', // Critical for Docker/CI
        '--disable-gpu',
        '--disable-background-timer-throttling',
        '--disable-backgrounding-occluded-windows',
        '--disable-renderer-backgrounding',
    ],
};
```

### waitFor Strategy (avoid hard sleeps)

```typescript
// ❌ Bad — arbitrary wait, fragile
await page.waitForTimeout(3000);

// ✅ Good — wait for what you actually need
await page.waitForSelector('.results-list', { visible: true, timeout: 10_000 });
await page.waitForFunction(() => document.querySelectorAll('.item').length > 0);
await page.waitForNetworkIdle({ idleTime: 500, timeout: 15_000 });
await page.waitForNavigation({ waitUntil: 'domcontentloaded' });
```

---

## Key Decision Points

### Which waitUntil to use?

| Scenario                     | waitUntil                            |
| ---------------------------- | ------------------------------------ |
| Static HTML page             | `'domcontentloaded'` (fastest)       |
| SPA with API calls           | `'networkidle0'` or `'networkidle2'` |
| PDF/screenshot generation    | `'load'`                             |
| Just navigation confirmation | `'commit'`                           |

### Stealth: Do you need it?

- **No bot detection** → vanilla Puppeteer, no extra setup
- **Basic detection (User-Agent checks)** → custom headers + user agent spoofing
- **Moderate protection (Cloudflare, DataDome)** → `puppeteer-extra` + stealth plugin
- **Advanced protection** → stealth + residential proxies + human-like behavior delays

### Concurrency strategy?

- **1-10 pages** → single browser, multiple pages
- **10-100 tasks** → `puppeteer-cluster` with `CONCURRENCY_PAGE`
- **100+ tasks** → `puppeteer-cluster` with `CONCURRENCY_BROWSER` or cloud browsers

---

## Anti-Patterns to Always Avoid

```typescript
// ❌ Launching browser inside a loop — kills performance and memory
for (const url of urls) {
    const browser = await puppeteer.launch(); // Don't do this!
    // ...
}

// ❌ No cleanup on error
const browser = await puppeteer.launch();
const page = await browser.newPage();
await page.goto(url); // If this throws, browser leaks!

// ❌ Returning DOM elements from evaluate (they don't serialize)
const element = await page.evaluate(() => document.querySelector('.item')); // Returns null/{}

// ❌ Using waitForTimeout for dynamic content
await page.waitForTimeout(5000); // Fragile and slow

// ❌ Not handling navigation promises together
await page.click('.submit'); // Navigation starts
await page.waitForNavigation(); // Race condition! Already may have navigated
// ✅ Do this instead:
await Promise.all([page.waitForNavigation(), page.click('.submit')]);
```

---

## Reference Files

Read the relevant reference file(s) for detailed patterns, full code examples, and edge cases:

- **Setup & Config** → `references/setup.md`
- **Scraping Patterns** → `references/scraping.md`
- **Stealth & Anti-bot** → `references/stealth.md`
- **Error Handling & Retries** → `references/error-handling.md`
- **Performance & Pooling** → `references/performance.md`
- **PDF & Screenshots** → `references/pdf-screenshots.md`
- **Forms & Interactions** → `references/forms.md`
- **Testing Patterns** → `references/testing.md`
