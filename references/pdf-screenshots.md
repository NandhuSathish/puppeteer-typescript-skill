# PDF & Screenshot Generation Reference

## Screenshots

```typescript
import puppeteer, { Page, ScreenshotOptions } from 'puppeteer';

// Full page screenshot
async function screenshotFullPage(url: string, outputPath: string): Promise<void> {
    const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
    const page = await browser.newPage();

    try {
        await page.setViewport({ width: 1440, height: 900, deviceScaleFactor: 2 }); // Retina
        await page.goto(url, { waitUntil: 'networkidle0', timeout: 30_000 });

        // Wait for fonts and images to load
        await page.evaluate(() => document.fonts.ready);

        await page.screenshot({
            path: outputPath,
            fullPage: true,
            type: 'png',
        });
    } finally {
        await page.close();
        await browser.close();
    }
}

// Element screenshot (crop to specific element)
async function screenshotElement(url: string, selector: string, outputPath: string): Promise<void> {
    const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
    const page = await browser.newPage();

    try {
        await page.setViewport({ width: 1440, height: 900 });
        await page.goto(url, { waitUntil: 'networkidle0' });

        const element = await page.waitForSelector(selector, { visible: true, timeout: 10_000 });
        if (!element) throw new Error(`Element not found: ${selector}`);

        await element.screenshot({ path: outputPath, type: 'png' });
    } finally {
        await page.close();
        await browser.close();
    }
}

// Screenshot as buffer (no file write — return to API)
async function screenshotToBuffer(url: string): Promise<Buffer> {
    const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
    const page = await browser.newPage();

    try {
        await page.setViewport({ width: 1280, height: 800 });
        await page.goto(url, { waitUntil: 'networkidle2' });

        return (await page.screenshot({
            type: 'jpeg',
            quality: 90,
            fullPage: false, // viewport only
        })) as Buffer;
    } finally {
        await page.close();
        await browser.close();
    }
}

// Screenshot multiple viewports (responsive testing)
const VIEWPORTS = [
    { name: 'mobile', width: 375, height: 667 },
    { name: 'tablet', width: 768, height: 1024 },
    { name: 'desktop', width: 1440, height: 900 },
] as const;

async function screenshotAllViewports(url: string, outputDir: string): Promise<void> {
    const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });

    try {
        for (const viewport of VIEWPORTS) {
            const page = await browser.newPage();
            try {
                await page.setViewport({ width: viewport.width, height: viewport.height });
                await page.goto(url, { waitUntil: 'networkidle0' });
                await page.screenshot({
                    path: `${outputDir}/${viewport.name}.png`,
                    fullPage: true,
                });
            } finally {
                await page.close();
            }
        }
    } finally {
        await browser.close();
    }
}
```

---

## PDF Generation

```typescript
import puppeteer, { PDFOptions } from 'puppeteer';
import * as fs from 'fs/promises';

// Standard PDF from URL
async function generatePdfFromUrl(url: string, outputPath: string): Promise<void> {
    const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
    const page = await browser.newPage();

    try {
        await page.goto(url, { waitUntil: 'networkidle0', timeout: 30_000 });

        // Ensure fonts are loaded
        await page.evaluate(() => document.fonts.ready);

        await page.pdf({
            path: outputPath,
            format: 'A4',
            printBackground: true, // Include CSS backgrounds (CRITICAL — default is false)
            margin: { top: '1cm', bottom: '1cm', left: '1.5cm', right: '1.5cm' },
        });
    } finally {
        await page.close();
        await browser.close();
    }
}

// PDF from HTML string (no URL needed)
async function generatePdfFromHtml(html: string, outputPath: string): Promise<Buffer> {
    const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
    const page = await browser.newPage();

    try {
        await page.setContent(html, { waitUntil: 'networkidle0' });
        await page.evaluate(() => document.fonts.ready);

        const pdfBuffer = await page.pdf({
            format: 'A4',
            printBackground: true,
            margin: { top: '20mm', bottom: '20mm', left: '15mm', right: '15mm' },
        });

        if (outputPath) {
            await fs.writeFile(outputPath, pdfBuffer);
        }

        return Buffer.from(pdfBuffer);
    } finally {
        await page.close();
        await browser.close();
    }
}

// PDF with header/footer templates
async function generatePdfWithHeaderFooter(url: string, outputPath: string, title: string): Promise<void> {
    const browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox'] });
    const page = await browser.newPage();

    try {
        await page.goto(url, { waitUntil: 'networkidle0' });

        await page.pdf({
            path: outputPath,
            format: 'A4',
            printBackground: true,
            displayHeaderFooter: true,
            // NOTE: Header/footer templates use inline styles only (no external CSS)
            // Available classes: date, title, url, pageNumber, totalPages
            headerTemplate: `
        <div style="font-size: 9px; width: 100%; padding: 5px 20px; border-bottom: 1px solid #ccc; display: flex; justify-content: space-between;">
          <span>${title}</span>
          <span class="date"></span>
        </div>
      `,
            footerTemplate: `
        <div style="font-size: 9px; width: 100%; padding: 5px 20px; border-top: 1px solid #ccc; display: flex; justify-content: space-between;">
          <span>Confidential</span>
          <span>Page <span class="pageNumber"></span> of <span class="totalPages"></span></span>
        </div>
      `,
            margin: { top: '2cm', bottom: '2cm', left: '1.5cm', right: '1.5cm' },
        });
    } finally {
        await page.close();
        await browser.close();
    }
}
```

---

## CSS for Print (inject into pages before PDF)

```typescript
// Inject print-optimized CSS before generating PDF
async function injectPrintStyles(page: Page): Promise<void> {
    await page.addStyleTag({
        content: `
      @media print {
        /* Hide nav, ads, buttons */
        nav, .advertisement, .cookie-banner, button, .social-share { display: none !important; }

        /* Prevent page breaks inside important elements */
        .card, .product, article { page-break-inside: avoid; }

        /* Ensure links show URL */
        a[href]::after { content: ' (' attr(href) ')'; font-size: 0.8em; }

        /* Reset backgrounds if needed */
        body { background: white !important; color: black !important; }
      }
    `,
    });
}
```

---

## Common PDF Pitfalls

```typescript
// ❌ Wrong — printBackground defaults to false, CSS backgrounds won't appear
await page.pdf({ format: 'A4' });

// ✅ Correct
await page.pdf({ format: 'A4', printBackground: true });

// ❌ Wrong — setContent without waiting for network (external assets won't load)
await page.setContent(html);

// ✅ Correct — wait for resources to load
await page.setContent(html, { waitUntil: 'networkidle0' });

// ❌ Wrong — header/footer with external CSS (ignored in header/footer context)
headerTemplate: '<link rel="stylesheet" href="style.css"><div>...</div>';

// ✅ Correct — always use inline styles in header/footer templates
headerTemplate: '<div style="font-size: 10px; ...">...</div>';
```
