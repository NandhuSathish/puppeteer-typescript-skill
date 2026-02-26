# Stealth & Anti-Bot Bypass Reference

## Detection Threat Model

Websites detect bots via:

1. **`navigator.webdriver` flag** — set to `true` in automation
2. **User-Agent** — contains "HeadlessChrome"
3. **Browser fingerprint** — missing plugins, WebGL anomalies, Canvas irregularities
4. **Behavioral signals** — robotic mouse movements, no scroll variance, instant clicks
5. **IP reputation** — datacenter IPs flagged by Cloudflare, DataDome, etc.
6. **TLS fingerprint** — Node.js http has a different TLS handshake than real Chrome

---

## Tier 1: Basic Headers (no extra dependencies)

Use when sites check User-Agent but don't use advanced fingerprinting:

```typescript
import puppeteer, { Page } from 'puppeteer';

const REALISTIC_UA = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36';

async function setupBasicStealth(page: Page): Promise<void> {
    // Override User-Agent
    await page.setUserAgent(REALISTIC_UA);

    // Set realistic extra headers
    await page.setExtraHTTPHeaders({
        'Accept-Language': 'en-US,en;q=0.9',
        Accept: 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8',
        'Accept-Encoding': 'gzip, deflate, br',
        'Upgrade-Insecure-Requests': '1',
        'Sec-Fetch-Dest': 'document',
        'Sec-Fetch-Mode': 'navigate',
        'Sec-Fetch-Site': 'none',
        'Sec-Fetch-User': '?1',
    });

    // Set a realistic viewport (don't use the Puppeteer default 800x600)
    await page.setViewport({ width: 1366, height: 768, deviceScaleFactor: 1 });
}
```

---

## Tier 2: Puppeteer Extra + Stealth Plugin

Use when sites use fingerprinting (Cloudflare JS challenge, DataDome, PerimeterX):

```typescript
import puppeteer from 'puppeteer-extra';
import StealthPlugin from 'puppeteer-extra-plugin-stealth';
import AnonUaPlugin from 'puppeteer-extra-plugin-anonymize-ua';
import type { Browser } from 'puppeteer';

// Apply plugins ONCE at module level, not inside functions
puppeteer.use(StealthPlugin());
puppeteer.use(AnonUaPlugin({ makeWindows: true })); // Makes UA look like Windows Chrome

export async function launchStealthBrowser(): Promise<Browser> {
    return puppeteer.launch({
        headless: true,
        args: [
            '--no-sandbox',
            '--disable-setuid-sandbox',
            '--disable-dev-shm-usage',
            '--disable-blink-features=AutomationControlled', // Hide automation flag
            '--disable-infobars',
            '--window-size=1366,768',
        ],
    });
}
```

The stealth plugin handles 17 evasion modules including:

- Hiding `navigator.webdriver`
- Fixing `navigator.plugins` (real browsers have plugins, headless doesn't)
- Fixing WebGL vendor/renderer strings
- Fixing `navigator.languages`
- Patching `window.chrome` object (missing in headless)
- Canvas fingerprint normalization

---

## Tier 3: Human-Like Behavior Simulation

Use alongside stealth plugin for behavioral analysis systems:

```typescript
import { Page } from 'puppeteer';

// Random delay mimicking human reaction time
const humanDelay = (min = 500, max = 2000): Promise<void> => new Promise((r) => setTimeout(r, min + Math.random() * (max - min)));

// Type like a human (character by character with variance)
async function humanType(page: Page, selector: string, text: string): Promise<void> {
    await page.click(selector);
    await humanDelay(200, 500);

    for (const char of text) {
        await page.keyboard.type(char);
        // Random delay between keystrokes: 50-200ms
        await new Promise((r) => setTimeout(r, 50 + Math.random() * 150));

        // Occasionally pause longer (simulating thinking)
        if (Math.random() < 0.05) {
            await humanDelay(300, 800);
        }
    }
}

// Move mouse in a curved path (not straight line)
async function humanMouseMove(page: Page, targetX: number, targetY: number): Promise<void> {
    const currentPos = { x: 0, y: 0 }; // Approximate start
    const steps = 15 + Math.floor(Math.random() * 10);

    for (let i = 0; i <= steps; i++) {
        const progress = i / steps;
        // Add slight curve/wobble to the path
        const wobble = (Math.random() - 0.5) * 20 * Math.sin(progress * Math.PI);
        const x = currentPos.x + (targetX - currentPos.x) * progress + wobble;
        const y = currentPos.y + (targetY - currentPos.y) * progress + wobble * 0.5;
        await page.mouse.move(x, y);
        await new Promise((r) => setTimeout(r, 10 + Math.random() * 20));
    }
}

// Human-like click with mouse movement
async function humanClick(page: Page, selector: string): Promise<void> {
    const element = await page.waitForSelector(selector, { visible: true });
    if (!element) throw new Error(`Element not found: ${selector}`);

    const box = await element.boundingBox();
    if (!box) throw new Error(`Could not get bounding box for: ${selector}`);

    // Click slightly off-center (humans rarely click exactly the center)
    const x = box.x + box.width * (0.3 + Math.random() * 0.4);
    const y = box.y + box.height * (0.3 + Math.random() * 0.4);

    await humanMouseMove(page, x, y);
    await humanDelay(50, 200);
    await page.mouse.click(x, y);
}

// Human-like scroll
async function humanScroll(page: Page): Promise<void> {
    const scrollDistance = 200 + Math.random() * 400;
    await page.evaluate((distance) => window.scrollBy(0, distance), scrollDistance);
    await humanDelay(200, 600);
}
```

---

## Proxy Integration

```typescript
import puppeteer from 'puppeteer-extra';
import StealthPlugin from 'puppeteer-extra-plugin-stealth';

puppeteer.use(StealthPlugin());

interface ProxyConfig {
    host: string;
    port: number;
    username?: string;
    password?: string;
}

async function launchWithProxy(proxy: ProxyConfig) {
    const browser = await puppeteer.launch({
        headless: true,
        args: ['--no-sandbox', '--disable-setuid-sandbox', `--proxy-server=${proxy.host}:${proxy.port}`],
    });

    // Authenticate if credentials provided
    if (proxy.username && proxy.password) {
        const page = await browser.newPage();
        await page.authenticate({ username: proxy.username, password: proxy.password });
        await page.close();
    }

    return browser;
}

// Proxy rotation example
class ProxyRotator {
    private currentIndex = 0;

    constructor(private proxies: ProxyConfig[]) {}

    next(): ProxyConfig {
        const proxy = this.proxies[this.currentIndex];
        this.currentIndex = (this.currentIndex + 1) % this.proxies.length;
        return proxy;
    }
}
```

---

## Manual `navigator.webdriver` Patch (no extra deps)

When you can't use puppeteer-extra but need basic evasion:

```typescript
async function patchWebdriver(page: Page): Promise<void> {
    await page.evaluateOnNewDocument(() => {
        // Remove webdriver property
        Object.defineProperty(navigator, 'webdriver', {
            get: () => undefined,
        });

        // Add fake plugins
        Object.defineProperty(navigator, 'plugins', {
            get: () => [{ name: 'Chrome PDF Plugin' }, { name: 'Chrome PDF Viewer' }, { name: 'Native Client' }],
        });

        // Set realistic languages
        Object.defineProperty(navigator, 'languages', {
            get: () => ['en-US', 'en'],
        });

        // Add window.chrome
        (window as Window & { chrome?: object }).chrome = {
            runtime: {},
        };
    });
}
```

---

## Detection Testing

Before deploying, test your stealth setup:

```typescript
// Navigate to these sites and take screenshots to verify stealth
const testSites = [
    'https://bot.sannysoft.com', // Comprehensive bot detection test
    'https://arh.antoinevastel.com/bots/areyouheadless', // Headless detection
    'https://fingerprintjs.github.io/fingerprintjs/', // Fingerprint analysis
];
```
