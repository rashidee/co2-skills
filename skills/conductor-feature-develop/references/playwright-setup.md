# Playwright E2E Test Setup Reference

## Project Initialization

```bash
mkdir -p <source-code-path>/e2e
cd <source-code-path>/e2e
npm init -y
npm install -D @playwright/test
npx playwright install chromium
```

## playwright.config.ts

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: false,        // Sequential — module order matters
  forbidOnly: true,
  retries: 1,
  workers: 1,                  // Single worker — tests share state
  reporter: [['html'], ['list']],
  use: {
    baseURL: 'http://localhost:8080',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { browserName: 'chromium' },
    },
  ],
});
```

## Helper: Login Utility

```typescript
// e2e/helpers/auth.ts
import { Page } from '@playwright/test';

export async function login(page: Page, username: string, password: string) {
  await page.goto('/');
  // Keycloak login page
  await page.fill('#username', username);
  await page.fill('#password', password);
  await page.click('#kc-login');
  await page.waitForURL('**/dashboard**');
}
```

## Helper: Data Seeding via CLI

```typescript
// e2e/helpers/seed.ts
import { execSync } from 'child_process';

export function seedMongo(script: string) {
  execSync(`mongosh "mongodb://localhost:27017/urp-hub-middleware" --eval '${script}'`, {
    timeout: 30000,
  });
}

export function publishRabbitMQ(exchange: string, routingKey: string, payload: object) {
  const json = JSON.stringify(payload).replace(/'/g, "'\\''");
  execSync(
    `rabbitmqadmin --host=localhost --port=15672 --username=guest --password=guest ` +
    `publish exchange=${exchange} routing_key=${routingKey} payload='${json}'`,
    { timeout: 30000 }
  );
}

export function keycloakAdmin(command: string) {
  execSync(`hub_single_sign_on/kcadm.bat ${command}`, { timeout: 30000 });
}
```

## Test File Pattern

```typescript
// e2e/tests/<module-slug>.spec.ts
import { test, expect } from '@playwright/test';
import { login } from '../helpers/auth';
import { seedMongo } from '../helpers/seed';

test.describe('<Module Name>', () => {

  test.beforeAll(async () => {
    // Run seeding script from TEST_SPEC.md Section 4
    seedMongo(`
      db.<collection>.insertMany([...]);
    `);
  });

  // NO afterAll cleanup — test data persists for downstream modules

  test('<SCENARIO-ID>: <description>', async ({ page }) => {
    await login(page, 'hub_admin_test', 'Test@1234');
    // Steps from TEST_SPEC.md scenarios
  });
});
```

## Helper: Visual Comparison Utilities

```typescript
// e2e/helpers/visual.ts
import { Page, expect } from '@playwright/test';
import { execSync } from 'child_process';
import path from 'path';

const MOCKUP_BASE_URL = 'http://localhost:3000';
const VISUAL_BASELINES_DIR = path.resolve(__dirname, '..', 'visual-baselines');

/**
 * Start the mockup server in the background.
 * Call this in beforeAll of visual tests if baselines need capturing.
 */
export function startMockupServer(mockupDir: string): void {
  // Start in background — the server stays alive for the test suite
  execSync(`cd "${mockupDir}" && npm start &`, { timeout: 10000 });
  // Give server time to start
  execSync('sleep 2');
}

/**
 * Capture a baseline screenshot from the HTML mockup server.
 * The mockup must be running on localhost:3000.
 */
export async function captureMockupBaseline(
  page: Page,
  mockupRoute: string,
  module: string,
  screenName: string
): Promise<void> {
  await page.goto(`${MOCKUP_BASE_URL}${mockupRoute}`);
  await page.waitForLoadState('networkidle');

  const screenshotPath = path.join(VISUAL_BASELINES_DIR, module, `${screenName}.png`);
  await page.screenshot({ path: screenshotPath, fullPage: true });
}

/**
 * Compare the Spring application screen against a mockup baseline.
 * Uses Playwright's toHaveScreenshot with relaxed thresholds to allow
 * content differences while catching aesthetic deviations (colors,
 * alignment, padding, margins, layout structure).
 */
export async function compareWithMockup(
  page: Page,
  appRoute: string,
  snapshotName: string
): Promise<void> {
  await page.goto(appRoute);
  await page.waitForLoadState('networkidle');

  await expect(page).toHaveScreenshot(snapshotName, {
    maxDiffPixelRatio: 0.15,   // Up to 15% pixel difference (content varies)
    threshold: 0.3,            // Per-pixel color threshold (0=exact, 1=any)
    animations: 'disabled',
    fullPage: true,
  });
}
```

## Visual Test File Pattern

```typescript
// e2e/tests/<module-slug>.visual.spec.ts
import { test, expect } from '@playwright/test';
import { loginAs } from '../helpers/auth';

test.describe('<Module Name> - Visual Consistency', () => {

  test('<screen-name> layout matches mockup', async ({ page }) => {
    await loginAs(page, '<role>');
    await page.goto('<app-route>');
    await page.waitForLoadState('networkidle');

    // Compares against snapshot — on first run, generates baseline
    // Threshold allows content differences but catches styling issues:
    // - Colors (backgrounds, borders, text, buttons)
    // - Spacing (padding, margins, gaps)
    // - Layout (grid/flex arrangement, proportions)
    // - Typography (font sizes, weights — relative, not exact)
    await expect(page).toHaveScreenshot('<module>-<screen>.png', {
      maxDiffPixelRatio: 0.15,
      threshold: 0.3,
      animations: 'disabled',
    });
  });
});
```

## Running Tests

```bash
# Run specific module functional tests
cd <source-code-path>/e2e && npx playwright test tests/<module-slug>.spec.ts

# Run specific module visual tests
cd <source-code-path>/e2e && npx playwright test tests/<module-slug>.visual.spec.ts

# Run all tests in order
cd <source-code-path>/e2e && npx playwright test

# Update visual snapshots (after reviewing and approving changes)
cd <source-code-path>/e2e && npx playwright test tests/<module-slug>.visual.spec.ts --update-snapshots

# Run with headed browser (for debugging)
cd <source-code-path>/e2e && npx playwright test tests/<module-slug>.spec.ts --headed

# View HTML report
cd <source-code-path>/e2e && npx playwright show-report
```
