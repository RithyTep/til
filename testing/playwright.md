# Playwright - End-to-End Testing

<div align="center">

![Playwright](https://img.shields.io/badge/Playwright-2EAD33?style=for-the-badge&logo=playwright&logoColor=white)
![Testing](https://img.shields.io/badge/Testing-E2E-blue?style=for-the-badge)
![Cross-Browser](https://img.shields.io/badge/Cross-Browser-Chrome_Firefox_Safari-orange?style=for-the-badge)

*Fast, reliable end-to-end testing for modern web apps across all browsers.*

![Playwright](https://upload.wikimedia.org/wikipedia/commons/7/75/Playwright_Logo.svg)

</div>

## Why Playwright?

```
Cypress                        Playwright
──────────────────────────     ──────────────────────────
Chrome-focused                 All browsers native
Single tab only                Multi-tab, multi-window
JavaScript only                Multiple languages
Limited mobile                 Full mobile emulation
Slower execution               Parallel by default
```

## Installation

```bash
npm init playwright@latest

# Or add to existing project
npm install -D @playwright/test
npx playwright install
```

## Basic Test

```typescript
// tests/example.spec.ts
import { test, expect } from '@playwright/test'

test('homepage has title', async ({ page }) => {
  await page.goto('https://playwright.dev/')

  // Expect title
  await expect(page).toHaveTitle(/Playwright/)

  // Click link
  await page.getByRole('link', { name: 'Get started' }).click()

  // Expect URL
  await expect(page).toHaveURL(/.*intro/)
})
```

## Locators

```typescript
test('locator examples', async ({ page }) => {
  // By role (recommended)
  await page.getByRole('button', { name: 'Submit' }).click()
  await page.getByRole('heading', { name: 'Welcome' })
  await page.getByRole('link', { name: /learn more/i })

  // By text
  await page.getByText('Hello World')
  await page.getByText(/hello/i) // Regex

  // By label (for form inputs)
  await page.getByLabel('Email').fill('test@example.com')
  await page.getByLabel('Password').fill('secret')

  // By placeholder
  await page.getByPlaceholder('Search...').fill('query')

  // By test ID (data-testid)
  await page.getByTestId('submit-button').click()

  // CSS selector (when needed)
  await page.locator('.my-class').click()
  await page.locator('#my-id').click()

  // Chaining
  await page
    .getByRole('listitem')
    .filter({ hasText: 'Product A' })
    .getByRole('button', { name: 'Buy' })
    .click()
})
```

## Assertions

```typescript
test('assertions', async ({ page }) => {
  // Page assertions
  await expect(page).toHaveTitle('My App')
  await expect(page).toHaveURL('/dashboard')

  // Element assertions
  const button = page.getByRole('button', { name: 'Submit' })
  await expect(button).toBeVisible()
  await expect(button).toBeEnabled()
  await expect(button).toHaveText('Submit')
  await expect(button).toHaveAttribute('type', 'submit')
  await expect(button).toHaveClass(/primary/)

  // Input assertions
  const input = page.getByLabel('Email')
  await expect(input).toHaveValue('test@example.com')
  await expect(input).toBeFocused()

  // Count
  const items = page.getByRole('listitem')
  await expect(items).toHaveCount(5)

  // Not assertions
  await expect(button).not.toBeDisabled()
})
```

## Form Interactions

```typescript
test('form submission', async ({ page }) => {
  await page.goto('/contact')

  // Fill form
  await page.getByLabel('Name').fill('John Doe')
  await page.getByLabel('Email').fill('john@example.com')
  await page.getByLabel('Message').fill('Hello!')

  // Select dropdown
  await page.getByLabel('Country').selectOption('US')

  // Checkbox
  await page.getByLabel('Subscribe').check()

  // Radio
  await page.getByLabel('Priority').getByText('High').click()

  // File upload
  await page.getByLabel('Upload').setInputFiles('file.pdf')

  // Submit
  await page.getByRole('button', { name: 'Submit' }).click()

  // Assert success
  await expect(page.getByText('Thank you!')).toBeVisible()
})
```

## Page Object Model

```typescript
// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test'

export class LoginPage {
  readonly page: Page
  readonly emailInput: Locator
  readonly passwordInput: Locator
  readonly submitButton: Locator

  constructor(page: Page) {
    this.page = page
    this.emailInput = page.getByLabel('Email')
    this.passwordInput = page.getByLabel('Password')
    this.submitButton = page.getByRole('button', { name: 'Sign in' })
  }

  async goto() {
    await this.page.goto('/login')
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email)
    await this.passwordInput.fill(password)
    await this.submitButton.click()
  }
}

// tests/login.spec.ts
import { test, expect } from '@playwright/test'
import { LoginPage } from '../pages/LoginPage'

test('successful login', async ({ page }) => {
  const loginPage = new LoginPage(page)

  await loginPage.goto()
  await loginPage.login('user@example.com', 'password')

  await expect(page).toHaveURL('/dashboard')
})
```

## API Testing

```typescript
import { test, expect } from '@playwright/test'

test('API requests', async ({ request }) => {
  // GET request
  const response = await request.get('/api/users')
  expect(response.ok()).toBeTruthy()

  const users = await response.json()
  expect(users).toHaveLength(10)

  // POST request
  const newUser = await request.post('/api/users', {
    data: {
      name: 'John',
      email: 'john@example.com',
    },
  })
  expect(newUser.status()).toBe(201)

  // With authentication
  const authResponse = await request.get('/api/profile', {
    headers: {
      Authorization: `Bearer ${token}`,
    },
  })
})
```

## Fixtures

```typescript
// fixtures.ts
import { test as base } from '@playwright/test'

interface User {
  email: string
  password: string
}

export const test = base.extend<{
  authenticatedPage: Page
  testUser: User
}>({
  testUser: async ({}, use) => {
    // Create test user
    const user = await createTestUser()
    await use(user)
    // Cleanup
    await deleteTestUser(user.id)
  },

  authenticatedPage: async ({ page, testUser }, use) => {
    // Login before test
    await page.goto('/login')
    await page.getByLabel('Email').fill(testUser.email)
    await page.getByLabel('Password').fill(testUser.password)
    await page.getByRole('button', { name: 'Sign in' }).click()
    await page.waitForURL('/dashboard')

    await use(page)
  },
})

// tests/dashboard.spec.ts
import { test } from '../fixtures'

test('authenticated user sees dashboard', async ({ authenticatedPage }) => {
  await expect(authenticatedPage.getByText('Welcome')).toBeVisible()
})
```

## Visual Regression

```typescript
test('visual comparison', async ({ page }) => {
  await page.goto('/homepage')

  // Full page screenshot comparison
  await expect(page).toHaveScreenshot('homepage.png')

  // Element screenshot
  const header = page.getByRole('banner')
  await expect(header).toHaveScreenshot('header.png')

  // With options
  await expect(page).toHaveScreenshot('page.png', {
    maxDiffPixels: 100,
    threshold: 0.2,
  })
})
```

## Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',

  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
  ],

  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
})
```

## CLI Commands

```bash
# Run all tests
npx playwright test

# Run specific test file
npx playwright test tests/login.spec.ts

# Run in headed mode (see browser)
npx playwright test --headed

# Run specific browser
npx playwright test --project=chromium

# Debug mode
npx playwright test --debug

# UI mode (interactive)
npx playwright test --ui

# Show report
npx playwright show-report

# Generate tests (codegen)
npx playwright codegen localhost:3000
```

---

*Learned: December 20, 2025*
*Tags: Playwright, Testing, E2E, Cross-Browser, Automation*
