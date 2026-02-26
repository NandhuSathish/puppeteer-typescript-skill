# Setup & Configuration Reference

## Installation

```bash
# Core (types included in puppeteer itself — do NOT install @types/puppeteer)
npm install puppeteer
npm install --save-dev typescript ts-node @types/node

# For stealth/anti-bot
npm install puppeteer-extra puppeteer-extra-plugin-stealth puppeteer-extra-plugin-anonymize-ua

# For concurrency
npm install puppeteer-cluster
```

## Recommended tsconfig.json

```json
{
    "compilerOptions": {
        "target": "ES2022",
        "module": "commonjs",
        "lib": ["ES2022"],
        "outDir": "./dist",
        "rootDir": "./src",
        "strict": true,
        "esModuleInterop": true,
        "skipLibCheck": true,
        "resolveJsonModule": true,
        "declaration": true,
        "sourceMap": true,
        "noUnusedLocals": true,
        "noImplicitReturns": true
    },
    "include": ["src/**/*"],
    "exclude": ["node_modules", "dist"]
}
```

## package.json scripts

```json
{
    "scripts": {
        "dev": "ts-node src/index.ts",
        "build": "tsc",
        "start": "node dist/index.js"
    }
}
```

## Launch Configuration Reference

```typescript
import { LaunchOptions } from 'puppeteer';

// Base config — always start here
export const baseLaunchConfig: LaunchOptions = {
    headless: true,
    args: [
        '--no-sandbox',
        '--disable-setuid-sandbox',
        '--disable-dev-shm-usage',
        '--disable-gpu',
        '--disable-background-timer-throttling',
        '--disable-backgrounding-occluded-windows',
        '--disable-renderer-backgrounding',
    ],
};

// Docker/CI-optimized config
export const dockerLaunchConfig: LaunchOptions = {
    ...baseLaunchConfig,
    args: [
        ...baseLaunchConfig.args!,
        '--disable-software-rasterizer',
        '--disable-features=TranslateUI',
        '--disable-ipc-flooding-protection',
        '--single-process', // Only in Docker where process isolation isn't needed
        '--no-zygote',
    ],
};

// Development config (visible browser for debugging)
export const devLaunchConfig: LaunchOptions = {
    headless: false,
    slowMo: 50, // Slows down operations so you can see what happens
    devtools: true, // Opens DevTools automatically
    args: ['--no-sandbox'],
};

// High-performance config for scraping (no GPU, no images by default)
export const scrapingLaunchConfig: LaunchOptions = {
    ...baseLaunchConfig,
    args: [...baseLaunchConfig.args!, '--disable-gpu', '--disable-accelerated-2d-canvas', '--disable-canvas-aa', '--disable-2d-canvas-clip-aa', '--disable-gl-drawing-for-tests'],
};
```

## Environment-based configuration

```typescript
// config/puppeteer.ts
import { LaunchOptions } from 'puppeteer';

const isDocker = Boolean(process.env.PUPPETEER_DOCKER);
const isDebug = process.env.DEBUG === 'puppeteer';

export function getLaunchConfig(): LaunchOptions {
    if (isDebug) {
        return { headless: false, slowMo: 100, devtools: true };
    }
    if (isDocker) {
        return {
            headless: true,
            executablePath: process.env.PUPPETEER_EXECUTABLE_PATH || '/usr/bin/chromium-browser',
            args: ['--no-sandbox', '--disable-setuid-sandbox', '--disable-dev-shm-usage', '--disable-gpu', '--single-process'],
        };
    }
    return {
        headless: true,
        args: ['--no-sandbox', '--disable-setuid-sandbox', '--disable-dev-shm-usage'],
    };
}
```

## .puppeteerrc.cjs (customize download cache location)

```javascript
// .puppeteerrc.cjs — place in project root
const { join } = require('path');

module.exports = {
    cacheDirectory: join(__dirname, '.cache', 'puppeteer'),
};
```

## Docker setup

```dockerfile
FROM node:20-slim

# Install Chromium dependencies
RUN apt-get update && apt-get install -y \
    chromium \
    fonts-liberation \
    libasound2 \
    libatk-bridge2.0-0 \
    libatk1.0-0 \
    libcups2 \
    libdbus-1-3 \
    libdrm2 \
    libgbm1 \
    libgtk-3-0 \
    libnspr4 \
    libnss3 \
    libxcomposite1 \
    libxdamage1 \
    libxfixes3 \
    libxrandr2 \
    xdg-utils \
    --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true
ENV PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium
ENV PUPPETEER_DOCKER=true

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
CMD ["node", "dist/index.js"]
```
