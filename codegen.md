# 🎭 Playwright Codegen — Complete Tutorial (TypeScript)

> **Goal:** Master `playwright codegen` to record interactions, handle authentication, and generate production-ready E2E tests.

---

## Table of Contents

1. [What is Codegen?](#1-what-is-codegen)
2. [Setup](#2-setup)
3. [Basic Usage](#3-basic-usage)
4. [Codegen Flags Cheatsheet](#4-codegen-flags-cheatsheet)
5. [Recording AFTER Login (Auth-Aware Codegen)](#5-recording-after-login-auth-aware-codegen)
   - 5a. Strategy 1 — `--save-storage` / `--load-storage`
   - 5b. Strategy 2 — `storageState` in `playwright.config.ts`
   - 5c. Strategy 3 — `global-setup.ts` + `storageState`
6. [Codegen with Custom Browser State](#6-codegen-with-custom-browser-state)
7. [Refining Generated Code](#7-refining-generated-code)
8. [Common Pitfalls & Fixes](#8-common-pitfalls--fixes)
9. [Full Real-World Example](#9-full-real-world-example)

---

## 1. What is Codegen?

`playwright codegen` opens a **browser + inspector side panel**. Every click, type, and navigation you perform is instantly translated into TypeScript test code you can copy or save.

```
┌──────────────────────────┬────────────────────────────────────┐
│   Your browser (live)    │   Generated TypeScript (live)      │
│                          │                                    │
│  [click "Login"]  ──────►│  await page.click('text=Login')    │
│  [type "admin"]   ──────►│  await page.fill('#user', 'admin') │
└──────────────────────────┴────────────────────────────────────┘
```

---

## 2. Setup

```bash
# Install Playwright with TypeScript
npm init playwright@latest

# Or add to an existing project
npm install -D @playwright/test
npx playwright install
```

Confirm your `tsconfig.json` is present (Playwright init generates one automatically).

---

## 3. Basic Usage

```bash
# Open codegen on a URL
npx playwright codegen https://example.com

# Save the generated code directly to a file
npx playwright codegen https://example.com --output tests/my-test.spec.ts

# Specify a browser
npx playwright codegen --browser firefox https://example.com

# Set viewport size
npx playwright codegen --viewport-size="1280,720" https://example.com
```

After recording, the generated file looks like this:

```ts
// tests/my-test.spec.ts  (auto-generated)
import { test, expect } from '@playwright/test';

test('test', async ({ page }) => {
  await page.goto('https://example.com');
  await page.getByRole('link', { name: 'Sign in' }).click();
  await page.getByLabel('Email').fill('user@example.com');
  await page.getByLabel('Password').fill('secret');
  await page.getByRole('button', { name: 'Login' }).click();
  await expect(page).toHaveURL('https://example.com/dashboard');
});
```

---

## 4. Codegen Flags Cheatsheet

| Flag | Description | Example |
|------|-------------|---------|
| `--output <file>` | Save generated code to file | `--output tests/flow.spec.ts` |
| `--browser <name>` | chromium / firefox / webkit | `--browser webkit` |
| `--device <name>` | Emulate a device | `--device "iPhone 13"` |
| `--viewport-size` | Custom width,height | `--viewport-size="1440,900"` |
| `--lang <locale>` | Browser language | `--lang fr-FR` |
| `--timezone <tz>` | Timezone | `--timezone "Europe/Paris"` |
| `--geolocation` | Fake GPS coords | `--geolocation "48.85,2.35"` |
| `--color-scheme` | light / dark | `--color-scheme dark` |
| `--save-storage <file>` | Persist cookies/localStorage after session | `--save-storage auth.json` |
| `--load-storage <file>` | Load a saved session before recording | `--load-storage auth.json` |
| `--ignore-https-errors` | Skip SSL errors | |
| `--proxy-server <url>` | Route through proxy | `--proxy-server http://proxy:8080` |

---

## 5. Recording AFTER Login (Auth-Aware Codegen)

This is the **most important scenario** for real applications. You don't want to re-record the login steps in every test — you want to start recording from an already-authenticated state.

---

### Strategy 5a — `--save-storage` / `--load-storage` (Quickest)

**Step 1 — Record the login and save session:**

```bash
npx playwright codegen \
  --save-storage=auth.json \
  https://your-app.com/login
```

1. The browser opens on the login page.
2. Manually log in (or let codegen record it — it doesn't matter).
3. **Close the browser window** when you are on the authenticated page.
4. `auth.json` is written — it contains cookies + localStorage snapshot.

```json
// auth.json (example contents — do not commit to git!)
{
  "cookies": [
    {
      "name": "session_id",
      "value": "abc123xyz",
      "domain": "your-app.com",
      "path": "/",
      "expires": 1740000000,
      "httpOnly": true,
      "secure": true
    }
  ],
  "origins": [
    {
      "origin": "https://your-app.com",
      "localStorage": [
        { "name": "token", "value": "eyJhbGci..." }
      ]
    }
  ]
}
```

> ⚠️ Add `auth.json` to `.gitignore` — it contains real session credentials!

```bash
echo "auth.json" >> .gitignore
```

**Step 2 — Start a NEW codegen session already authenticated:**

```bash
npx playwright codegen \
  --load-storage=auth.json \
  --output=tests/dashboard.spec.ts \
  https://your-app.com/dashboard
```

The browser opens directly on `/dashboard` **already logged in**. Everything you record from here skips authentication entirely.

---

### Strategy 5b — `storageState` in `playwright.config.ts` (Team-Friendly)

If your whole team runs tests against a shared test account, store the auth state once and reference it in config.

**Step 1 — Create a dedicated auth script:**

```ts
// scripts/save-auth.ts
import { chromium } from '@playwright/test';

(async () => {
  const browser = await chromium.launch({ headless: false });
  const context = await browser.newContext();
  const page = await context.newPage();

  await page.goto('https://your-app.com/login');
  await page.fill('#email', process.env.TEST_EMAIL!);
  await page.fill('#password', process.env.TEST_PASSWORD!);
  await page.click('button[type=submit]');
  await page.waitForURL('**/dashboard');

  // Save session to disk
  await context.storageState({ path: 'playwright/.auth/user.json' });
  await browser.close();

  console.log('✅ Auth state saved to playwright/.auth/user.json');
})();
```

```bash
mkdir -p playwright/.auth
echo "playwright/.auth" >> .gitignore

# Run the script
TEST_EMAIL=test@example.com TEST_PASSWORD=secret npx ts-node scripts/save-auth.ts
```

**Step 2 — Reference the auth state in config:**

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  use: {
    baseURL: 'https://your-app.com',
    // All tests start already authenticated
    storageState: 'playwright/.auth/user.json',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],
});
```

**Step 3 — Codegen with that state:**

```bash
npx playwright codegen \
  --load-storage=playwright/.auth/user.json \
  --output=tests/settings.spec.ts \
  https://your-app.com/settings
```

---

### Strategy 5c — `global-setup.ts` + `storageState` (CI/Production)

This is the **recommended approach for CI pipelines**. Auth is performed once before all tests.

```ts
// global-setup.ts
import { chromium, FullConfig } from '@playwright/test';

async function globalSetup(config: FullConfig) {
  const { baseURL } = config.projects[0].use;
  const browser = await chromium.launch();
  const page = await browser.newPage();

  await page.goto(`${baseURL}/login`);
  await page.fill('[name=email]', process.env.TEST_EMAIL!);
  await page.fill('[name=password]', process.env.TEST_PASSWORD!);
  await page.click('[type=submit]');
  await page.waitForURL('**/dashboard');

  await page.context().storageState({ path: 'playwright/.auth/user.json' });
  await browser.close();
}

export default globalSetup;
```

```ts
// playwright.config.ts
export default defineConfig({
  globalSetup: require.resolve('./global-setup'),
  use: {
    storageState: 'playwright/.auth/user.json',
  },
});
```

Now every test (and every codegen session with `--load-storage`) starts authenticated. Login is done **once** for the whole suite.

---

## 6. Codegen with Custom Browser State

Beyond auth, you can pre-seed the browser with any state before recording.

### Pre-set localStorage / cookies programmatically:

```ts
// scripts/codegen-with-state.ts
import { chromium } from '@playwright/test';

(async () => {
  const browser = await chromium.launch({ headless: false });
  const context = await browser.newContext();

  // Inject a cookie before navigating
  await context.addCookies([
    {
      name: 'feature_flag',
      value: 'dark_mode_enabled',
      domain: 'your-app.com',
      path: '/',
    },
  ]);

  const page = await context.newPage();

  // Inject localStorage
  await page.goto('https://your-app.com');
  await page.evaluate(() => {
    localStorage.setItem('onboarding_complete', 'true');
    localStorage.setItem('user_role', 'admin');
  });

  // Now navigate to the page you want to record
  await page.goto('https://your-app.com/admin-panel');

  // 👇 Open the Playwright Inspector to start recording
  await page.pause(); // This opens the inspector UI
})();
```

```bash
npx ts-node scripts/codegen-with-state.ts
```

> `page.pause()` opens the **Playwright Inspector** mid-script — you can then hit **Record** to start codegen from that exact state.

---

### Emulate a mobile device + geolocation:

```bash
npx playwright codegen \
  --device="Pixel 7" \
  --geolocation="48.8566,2.3522" \
  --lang="fr-FR" \
  --load-storage=auth.json \
  https://your-app.com
```

---

## 7. Refining Generated Code

Codegen produces functional but verbose code. Here is how to clean it up.

### Before (raw codegen output):

```ts
test('test', async ({ page }) => {
  await page.goto('https://your-app.com/dashboard');
  await page.locator('div').filter({ hasText: /^Settings$/ }).nth(2).click();
  await page.locator('#email-input').click();
  await page.locator('#email-input').fill('new@example.com');
  await page.locator('button:nth-child(3)').click();
});
```

### After (cleaned up):

```ts
test('should update user email in settings', async ({ page }) => {
  await page.goto('/dashboard');  // baseURL handles the domain

  // Use semantic locators
  await page.getByRole('link', { name: 'Settings' }).click();
  await page.getByLabel('Email address').fill('new@example.com');
  await page.getByRole('button', { name: 'Save changes' }).click();

  // Add assertions (codegen never adds these — you must!)
  await expect(page.getByText('Saved successfully')).toBeVisible();
  await expect(page.getByLabel('Email address')).toHaveValue('new@example.com');
});
```

### Locator quality hierarchy (best → worst):

```
getByRole()          ← Best: semantic, accessible, resilient
getByLabel()         ← Great for form fields
getByPlaceholder()   ← Good fallback for inputs
getByText()          ← OK for static text
getByTestId()        ← Good if data-testid exists
locator('#id')       ← Fragile: avoid if possible
locator('nth-child') ← Very fragile: always replace
```

---

## 8. Common Pitfalls & Fixes

### ❌ Codegen records the login every time

**Fix:** Use `--load-storage` with a saved auth state (see Strategy 5a/5b/5c).

---

### ❌ Session expires between codegen and test run

**Fix:** Add a `beforeEach` that checks auth and refreshes the token if needed:

```ts
test.beforeEach(async ({ page }) => {
  await page.goto('/dashboard');
  // If redirected to login, the storageState is stale — regenerate it
  if (page.url().includes('/login')) {
    throw new Error('Session expired. Re-run: npx ts-node scripts/save-auth.ts');
  }
});
```

---

### ❌ Generated locators break after UI changes

**Fix:** Replace fragile locators with `data-testid` attributes:

```html
<!-- In your app's HTML -->
<button data-testid="submit-order">Place Order</button>
```

```ts
// In your test (stable across UI refactors)
await page.getByTestId('submit-order').click();
```

---

### ❌ Codegen misses dynamically loaded content

**Fix:** After recording, add explicit waits:

```ts
// Instead of immediately clicking
await page.getByRole('button', { name: 'Load more' }).click();

// Wait for the network call to finish
await page.waitForResponse(resp => resp.url().includes('/api/items'));
// Or wait for an element to appear
await page.waitForSelector('[data-testid="items-loaded"]');
```

---

### ❌ Multi-tab flows are not recorded

**Fix:** Use `context.waitForEvent('page')` to capture new tabs:

```ts
const [newPage] = await Promise.all([
  page.context().waitForEvent('page'),
  page.getByRole('link', { name: 'Open in new tab' }).click(),
]);
await newPage.waitForLoadState();
await expect(newPage).toHaveURL(/\/report/);
```

---

## 9. Full Real-World Example

This combines everything: global setup, auth state, and a clean post-auth recording.

### Project structure:

```
my-app/
├── playwright.config.ts
├── global-setup.ts
├── playwright/
│   └── .auth/
│       └── user.json        ← gitignored
├── tests/
│   ├── auth.setup.ts        ← generates user.json in CI
│   ├── dashboard.spec.ts    ← recorded after login
│   └── checkout.spec.ts     ← recorded after login
└── .env.test                ← TEST_EMAIL / TEST_PASSWORD
```

### `playwright.config.ts`:

```ts
import { defineConfig, devices } from '@playwright/test';
import dotenv from 'dotenv';

dotenv.config({ path: '.env.test' });

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,

  use: {
    baseURL: process.env.BASE_URL ?? 'https://staging.your-app.com',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },

  projects: [
    // Setup project — runs first, saves auth state
    {
      name: 'setup',
      testMatch: /auth\.setup\.ts/,
    },
    // All other tests depend on the setup
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },
  ],
});
```

### `tests/auth.setup.ts`:

```ts
import { test as setup, expect } from '@playwright/test';
import path from 'path';

const authFile = path.join(__dirname, '../playwright/.auth/user.json');

setup('authenticate', async ({ page }) => {
  await page.goto('/login');

  await page.getByLabel('Email').fill(process.env.TEST_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();

  // Wait until auth completes
  await expect(page).toHaveURL(/\/dashboard/);
  await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();

  // Persist session for all subsequent tests + codegen
  await page.context().storageState({ path: authFile });
});
```

### Start a post-auth codegen session:

```bash
# Auth state must be generated first
npx playwright test --project=setup

# Then record from an authenticated session
npx playwright codegen \
  --load-storage=playwright/.auth/user.json \
  --output=tests/checkout.spec.ts \
  https://staging.your-app.com/shop
```

### `tests/checkout.spec.ts` (after cleanup):

```ts
import { test, expect } from '@playwright/test';

test.describe('Checkout flow', () => {
  test('should complete purchase successfully', async ({ page }) => {
    await page.goto('/shop');

    await page.getByRole('button', { name: 'Add to cart', exact: false }).first().click();
    await page.getByRole('link', { name: 'Cart (1)' }).click();
    await page.getByRole('button', { name: 'Proceed to checkout' }).click();

    await page.getByLabel('Card number').fill('4242 4242 4242 4242');
    await page.getByLabel('Expiry').fill('12/28');
    await page.getByLabel('CVV').fill('123');

    await page.getByRole('button', { name: 'Pay now' }).click();

    await expect(page.getByRole('heading', { name: 'Order confirmed' })).toBeVisible();
    await expect(page.getByTestId('order-id')).toBeVisible();
  });
});
```

### Run the full suite:

```bash
# Run all tests (setup runs first automatically)
npx playwright test

# Run a specific file
npx playwright test tests/checkout.spec.ts

# Open the HTML report
npx playwright show-report
```

---

## Quick Reference Card

```bash
# ── Basic codegen ──────────────────────────────────────────────
npx playwright codegen https://app.com

# ── Save output to file ────────────────────────────────────────
npx playwright codegen https://app.com --output tests/flow.spec.ts

# ── Save auth state ────────────────────────────────────────────
npx playwright codegen --save-storage=auth.json https://app.com/login

# ── Record AFTER login ─────────────────────────────────────────
npx playwright codegen --load-storage=auth.json https://app.com/dashboard

# ── Mobile + language ──────────────────────────────────────────
npx playwright codegen --device="iPhone 14" --lang="fr-FR" https://app.com

# ── Dark mode ──────────────────────────────────────────────────
npx playwright codegen --color-scheme=dark https://app.com

# ── Pause mid-script and record ────────────────────────────────
await page.pause();  // Add to any .ts script, run with ts-node
```

---

*Generated for Playwright v1.44+ with TypeScript. All strategies are compatible with Playwright Test runner and CI environments.*
