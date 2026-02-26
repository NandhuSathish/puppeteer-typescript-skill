# Forms & Interactions Reference

## Input Types & How to Fill Them

```typescript
import { Page, ElementHandle } from 'puppeteer';

// TEXT INPUT — clear first, then type
async function fillText(page: Page, selector: string, value: string): Promise<void> {
    await page.waitForSelector(selector, { visible: true });
    await page.click(selector, { clickCount: 3 }); // Select all existing text
    await page.keyboard.press('Backspace');
    await page.type(selector, value, { delay: 30 }); // delay mimics human typing
}

// Alternative: faster fill using evaluate (no typing delay)
async function fillTextFast(page: Page, selector: string, value: string): Promise<void> {
    await page.waitForSelector(selector);
    await page.$eval(
        selector,
        (el, val) => {
            (el as HTMLInputElement).value = val;
            // Trigger React/Vue/Angular change detection
            el.dispatchEvent(new Event('input', { bubbles: true }));
            el.dispatchEvent(new Event('change', { bubbles: true }));
        },
        value,
    );
}

// SELECT DROPDOWN
async function selectOption(page: Page, selector: string, value: string): Promise<void> {
    await page.waitForSelector(selector);
    await page.select(selector, value); // value = the <option value="..."> attribute
}

// Get all options from a select
async function getSelectOptions(page: Page, selector: string): Promise<{ value: string; text: string }[]> {
    return page.$$eval(`${selector} option`, (options) =>
        options.map((o) => ({
            value: (o as HTMLOptionElement).value,
            text: (o as HTMLOptionElement).text.trim(),
        })),
    );
}

// CHECKBOX & RADIO
async function setCheckbox(page: Page, selector: string, checked: boolean): Promise<void> {
    const currentState = await page.$eval(selector, (el) => (el as HTMLInputElement).checked);
    if (currentState !== checked) {
        await page.click(selector);
    }
}

// DATE INPUT
async function fillDate(page: Page, selector: string, dateString: string): Promise<void> {
    // dateString format: 'YYYY-MM-DD'
    await page.$eval(
        selector,
        (el, date) => {
            (el as HTMLInputElement).value = date;
            el.dispatchEvent(new Event('change', { bubbles: true }));
        },
        dateString,
    );
}

// FILE UPLOAD
async function uploadFile(page: Page, selector: string, filePath: string): Promise<void> {
    const fileInput = (await page.waitForSelector(selector)) as ElementHandle<HTMLInputElement>;
    if (!fileInput) throw new Error(`File input not found: ${selector}`);
    await fileInput.uploadFile(filePath);
}

// CONTENT EDITABLE (rich text editors)
async function fillContentEditable(page: Page, selector: string, text: string): Promise<void> {
    await page.click(selector);
    await page.keyboard.down('Control');
    await page.keyboard.press('KeyA');
    await page.keyboard.up('Control');
    await page.keyboard.type(text);
}
```

---

## Form Submission Patterns

```typescript
// Pattern 1: Submit button click + wait for navigation
async function submitAndNavigate(page: Page, buttonSelector: string): Promise<void> {
    await Promise.all([page.waitForNavigation({ waitUntil: 'domcontentloaded', timeout: 30_000 }), page.click(buttonSelector)]);
}

// Pattern 2: Submit + wait for API response (SPA forms)
async function submitAndWaitForApi(page: Page, buttonSelector: string, apiEndpoint: string): Promise<unknown> {
    const [response] = await Promise.all([
        page.waitForResponse((res) => res.url().includes(apiEndpoint) && res.request().method() === 'POST', { timeout: 15_000 }),
        page.click(buttonSelector),
    ]);
    return response.json();
}

// Pattern 3: Submit form via keyboard (Enter key)
async function submitWithEnter(page: Page, inputSelector: string): Promise<void> {
    await Promise.all([page.waitForNavigation({ waitUntil: 'domcontentloaded' }), page.press(inputSelector, 'Enter')]);
}

// Pattern 4: Submit via JS (bypass disabled button)
async function forceSubmit(page: Page, formSelector: string): Promise<void> {
    await page.$eval(formSelector, (form) => (form as HTMLFormElement).submit());
}
```

---

## Login Flow Pattern

```typescript
interface Credentials {
    username: string;
    password: string;
}

async function login(page: Page, loginUrl: string, selectors: { username: string; password: string; submit: string }, credentials: Credentials): Promise<boolean> {
    await page.goto(loginUrl, { waitUntil: 'domcontentloaded' });

    await fillText(page, selectors.username, credentials.username);
    await fillText(page, selectors.password, credentials.password);

    await Promise.all([page.waitForNavigation({ waitUntil: 'networkidle0', timeout: 30_000 }), page.click(selectors.submit)]);

    // Verify login success (adjust selector to match the target site)
    const loggedIn = await page.$('.user-dashboard, .account-menu, [data-testid="user-avatar"]');
    return loggedIn !== null;
}

// Reuse session to avoid re-login
async function loginAndSaveSession(credentials: Credentials, sessionFile: string): Promise<void> {
    const browser = await puppeteer.launch({ args: ['--no-sandbox'] });
    const page = await browser.newPage();

    await login(
        page,
        'https://example.com/login',
        {
            username: '#email',
            password: '#password',
            submit: '[type="submit"]',
        },
        credentials,
    );

    // Save cookies to file
    const cookies = await page.cookies();
    await fs.writeFile(sessionFile, JSON.stringify(cookies, null, 2));

    await browser.close();
}

async function loadSession(page: Page, sessionFile: string): Promise<void> {
    const cookiesJson = await fs.readFile(sessionFile, 'utf-8');
    const cookies = JSON.parse(cookiesJson);
    await page.setCookie(...cookies);
}
```

---

## Handling Modals & Dialogs

```typescript
// Browser alert/confirm/prompt dialogs
page.on('dialog', async (dialog) => {
    console.log(`Dialog type: ${dialog.type()}, message: ${dialog.message()}`);

    if (dialog.type() === 'confirm') {
        await dialog.accept(); // or dialog.dismiss()
    } else if (dialog.type() === 'prompt') {
        await dialog.accept('my answer');
    } else {
        await dialog.dismiss();
    }
});

// Wait for a modal to appear and interact with it
async function handleModal(page: Page, modalSelector: string): Promise<void> {
    await page.waitForSelector(modalSelector, { visible: true, timeout: 10_000 });

    // Interact with modal content
    await page.click(`${modalSelector} .modal-close`);

    // Wait for modal to disappear
    await page.waitForSelector(modalSelector, { hidden: true, timeout: 5_000 });
}
```

---

## Drag & Drop

```typescript
async function dragAndDrop(page: Page, sourceSelector: string, targetSelector: string): Promise<void> {
    const source = await page.waitForSelector(sourceSelector);
    const target = await page.waitForSelector(targetSelector);

    if (!source || !target) throw new Error('Source or target not found');

    const sourceBox = await source.boundingBox();
    const targetBox = await target.boundingBox();

    if (!sourceBox || !targetBox) throw new Error('Could not get bounding boxes');

    // Simulate drag using mouse events
    await page.mouse.move(sourceBox.x + sourceBox.width / 2, sourceBox.y + sourceBox.height / 2);
    await page.mouse.down();

    // Move incrementally to trigger drag events
    const steps = 10;
    for (let i = 1; i <= steps; i++) {
        const x = sourceBox.x + (targetBox.x - sourceBox.x) * (i / steps) + sourceBox.width / 2;
        const y = sourceBox.y + (targetBox.y - sourceBox.y) * (i / steps) + sourceBox.height / 2;
        await page.mouse.move(x, y);
    }

    await page.mouse.up();
}
```

---

## Keyboard Shortcuts

```typescript
// Common keyboard operations
await page.keyboard.press('Tab');
await page.keyboard.press('Escape');
await page.keyboard.press('Enter');

// Key combinations
await page.keyboard.down('Control');
await page.keyboard.press('KeyA'); // Ctrl+A (Select All)
await page.keyboard.up('Control');

await page.keyboard.down('Control');
await page.keyboard.press('KeyC'); // Ctrl+C (Copy)
await page.keyboard.up('Control');
```
