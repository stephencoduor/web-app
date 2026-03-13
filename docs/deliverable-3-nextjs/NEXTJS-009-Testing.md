# NEXTJS-009: Testing Strategy

## Status: Draft

## Last Updated: 2026-03-13

---

## 1. Test Stack

| Layer             | Tool                                      | Purpose                                              |
| ----------------- | ----------------------------------------- | ---------------------------------------------------- |
| Unit Tests        | **Vitest**                                | Fast, Vite-powered test runner (Jest-compatible API) |
| Component Tests   | **React Testing Library**                 | DOM-based component testing                          |
| Hook Tests        | **@testing-library/react** + `renderHook` | Testing React Query hooks                            |
| E2E Tests         | **Playwright**                            | Full browser automation tests                        |
| API Mocking       | **MSW (Mock Service Worker)**             | Intercepts HTTP requests at the network level        |
| Visual Regression | **Playwright screenshots**                | Optional: visual snapshot comparison                 |

### Why Vitest over Jest

- Native ESM support (no transform workarounds for Next.js)
- Built-in TypeScript support without `ts-jest`
- Compatible with Jest API (`describe`, `it`, `expect`, `vi.fn()`)
- Significantly faster execution due to Vite's transform pipeline
- Built-in coverage via `@vitest/coverage-v8`

---

## 2. Vitest Configuration

### 2.1 Installation

```bash
npm install -D vitest @vitejs/plugin-react @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom msw
```

### 2.2 Configuration File

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    include: ['src/**/*.{test,spec}.{ts,tsx}'],
    exclude: [
      'node_modules',
      '.next',
      'e2e'
    ],
    coverage: {
      provider: 'v8',
      reporter: [
        'text',
        'lcov',
        'html'
      ],
      include: ['src/**/*.{ts,tsx}'],
      exclude: [
        'src/**/*.d.ts',
        'src/**/*.test.{ts,tsx}',
        'src/**/*.spec.{ts,tsx}',
        'src/test/**',
        'src/lib/types/**'
      ],
      thresholds: {
        statements: 80,
        branches: 75,
        functions: 80,
        lines: 80
      }
    },
    css: {
      // Skip CSS module resolution in tests
      modules: { classNameStrategy: 'non-scoped' }
    }
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src')
    }
  }
});
```

### 2.3 Test Setup File

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom/vitest';
import { cleanup } from '@testing-library/react';
import { afterEach, vi } from 'vitest';

// Cleanup after each test
afterEach(() => {
  cleanup();
});

// Mock next/navigation
vi.mock('next/navigation', () => ({
  useRouter: () => ({
    push: vi.fn(),
    replace: vi.fn(),
    refresh: vi.fn(),
    back: vi.fn(),
    prefetch: vi.fn()
  }),
  usePathname: () => '/clients',
  useSearchParams: () => new URLSearchParams(),
  useParams: () => ({}),
  redirect: vi.fn(),
  notFound: vi.fn()
}));

// Mock next-intl
vi.mock('next-intl', () => ({
  useTranslations: () => (key: string) => key,
  useLocale: () => 'en-US',
  useFormatter: () => ({
    dateTime: (date: Date) => date.toISOString(),
    number: (num: number) => num.toString(),
    relativeTime: (date: Date) => 'just now'
  })
}));

// Mock next-intl/server
vi.mock('next-intl/server', () => ({
  getTranslations: async () => (key: string) => key,
  getFormatter: async () => ({
    dateTime: (date: Date) => date.toISOString(),
    number: (num: number) => num.toString()
  }),
  getLocale: async () => 'en-US'
}));
```

### 2.4 Package.json Scripts

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui"
  }
}
```

---

## 3. Component Test Patterns

### 3.1 Test Utilities and Wrappers

Create a custom render function that wraps components with all necessary providers:

```typescript
// src/test/test-utils.tsx
import { render, RenderOptions } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { NextIntlClientProvider } from 'next-intl';
import { ReactElement } from 'react';

import enMessages from '@/lib/i18n/messages/en-US.json';

function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,         // No retries in tests
        gcTime: 0,            // No caching in tests
        staleTime: 0,
      },
      mutations: {
        retry: false,
      },
    },
  });
}

interface WrapperProps {
  children: React.ReactNode;
}

function AllProviders({ children }: WrapperProps) {
  const queryClient = createTestQueryClient();

  return (
    <QueryClientProvider client={queryClient}>
      <NextIntlClientProvider locale="en-US" messages={enMessages}>
        {children}
      </NextIntlClientProvider>
    </QueryClientProvider>
  );
}

function customRender(
  ui: ReactElement,
  options?: Omit<RenderOptions, 'wrapper'>
) {
  return render(ui, { wrapper: AllProviders, ...options });
}

// Re-export everything from testing-library
export * from '@testing-library/react';
export { customRender as render };
```

### 3.2 Testing React Query Hooks

```typescript
// src/lib/hooks/__tests__/clients.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useClients, useClient, useCreateClient } from '../clients';
import { server } from '@/test/msw/server';
import { http, HttpResponse } from 'msw';

const FINERACT_BASE = '/fineract-provider/api/v1';

function createWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return function Wrapper({ children }: { children: React.ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    );
  };
}

describe('useClients', () => {
  it('fetches client list successfully', async () => {
    server.use(
      http.get(`${FINERACT_BASE}/clients`, () => {
        return HttpResponse.json({
          totalFilteredRecords: 2,
          pageItems: [
            { id: 1, displayName: 'John Doe', accountNo: '000000001' },
            { id: 2, displayName: 'Jane Smith', accountNo: '000000002' },
          ],
        });
      })
    );

    const { result } = renderHook(() => useClients(), {
      wrapper: createWrapper(),
    });

    await waitFor(() => expect(result.current.isSuccess).toBe(true));

    expect(result.current.data?.pageItems).toHaveLength(2);
    expect(result.current.data?.pageItems[0].displayName).toBe('John Doe');
  });

  it('handles fetch error', async () => {
    server.use(
      http.get(`${FINERACT_BASE}/clients`, () => {
        return HttpResponse.json(
          { error: 'Unauthorized' },
          { status: 401 }
        );
      })
    );

    const { result } = renderHook(() => useClients(), {
      wrapper: createWrapper(),
    });

    await waitFor(() => expect(result.current.isError).toBe(true));
  });
});

describe('useCreateClient', () => {
  it('creates a client and invalidates list cache', async () => {
    server.use(
      http.post(`${FINERACT_BASE}/clients`, () => {
        return HttpResponse.json({ clientId: 42, resourceId: 42 });
      })
    );

    const { result } = renderHook(() => useCreateClient(), {
      wrapper: createWrapper(),
    });

    await result.current.mutateAsync({
      officeId: 1,
      firstname: 'Test',
      lastname: 'User',
      active: false,
      submittedOnDate: '15 March 2026',
      locale: 'en',
      dateFormat: 'dd MMMM yyyy',
    });

    expect(result.current.data?.clientId).toBe(42);
  });
});
```

### 3.3 Testing Forms with React Hook Form

```typescript
// src/components/clients/__tests__/create-client-wizard.test.tsx
import { render, screen, waitFor } from '@/test/test-utils';
import userEvent from '@testing-library/user-event';
import { CreateClientWizard } from '../create-client-wizard';
import { server } from '@/test/msw/server';
import { http, HttpResponse } from 'msw';

const mockTemplate = {
  officeOptions: [{ id: 1, name: 'Head Office' }],
  staffOptions: [{ id: 1, displayName: 'Admin' }],
  genderOptions: [{ id: 22, name: 'Male' }, { id: 24, name: 'Female' }],
  clientTypeOptions: [],
  clientClassificationOptions: [],
  clientLegalFormOptions: [{ id: 1, value: 'PERSON' }],
};

beforeEach(() => {
  server.use(
    http.get('/fineract-provider/api/v1/clients/template', () => {
      return HttpResponse.json(mockTemplate);
    })
  );
});

describe('CreateClientWizard', () => {
  it('renders the first step with office selection', async () => {
    render(<CreateClientWizard />);

    await waitFor(() => {
      expect(screen.getByText('General')).toBeInTheDocument();
    });

    expect(screen.getByLabelText(/office/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/first name/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/last name/i)).toBeInTheDocument();
  });

  it('validates required fields before advancing', async () => {
    const user = userEvent.setup();
    render(<CreateClientWizard />);

    await waitFor(() => {
      expect(screen.getByText('labels.buttons.next')).toBeInTheDocument();
    });

    await user.click(screen.getByText('labels.buttons.next'));

    // Should show validation errors
    await waitFor(() => {
      expect(screen.getByText(/required/i)).toBeInTheDocument();
    });
  });

  it('navigates through steps', async () => {
    const user = userEvent.setup();
    render(<CreateClientWizard />);

    await waitFor(() => {
      expect(screen.getByLabelText(/first name/i)).toBeInTheDocument();
    });

    // Fill required fields
    await user.type(screen.getByLabelText(/first name/i), 'John');
    await user.type(screen.getByLabelText(/last name/i), 'Doe');

    // Advance to step 2
    await user.click(screen.getByText('labels.buttons.next'));

    await waitFor(() => {
      expect(screen.getByText(/family members/i)).toBeInTheDocument();
    });
  });
});
```

### 3.4 Testing Dialog Interactions

```typescript
// src/components/clients/__tests__/client-action-dialog.test.tsx
import { render, screen, waitFor } from '@/test/test-utils';
import userEvent from '@testing-library/user-event';
import { ClientActionDialog } from '../client-action-dialog';

describe('ClientActionDialog', () => {
  it('opens and submits the activate action', async () => {
    const user = userEvent.setup();
    const onConfirm = vi.fn();

    render(
      <ClientActionDialog
        action="activate"
        clientId={42}
        open={true}
        onOpenChange={() => {}}
        onConfirm={onConfirm}
      />
    );

    expect(screen.getByText(/activate client/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/activation date/i)).toBeInTheDocument();

    await user.click(screen.getByRole('button', { name: /confirm/i }));

    await waitFor(() => {
      expect(onConfirm).toHaveBeenCalled();
    });
  });

  it('closes on cancel', async () => {
    const user = userEvent.setup();
    const onOpenChange = vi.fn();

    render(
      <ClientActionDialog
        action="activate"
        clientId={42}
        open={true}
        onOpenChange={onOpenChange}
        onConfirm={() => {}}
      />
    );

    await user.click(screen.getByRole('button', { name: /cancel/i }));

    expect(onOpenChange).toHaveBeenCalledWith(false);
  });
});
```

### 3.5 Testing Server Components

Server Components cannot be rendered in jsdom. Test their data-fetching logic separately:

```typescript
// src/app/(dashboard)/clients/[clientId]/general/__tests__/page.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { clientsApi } from '@/lib/api/fineract/clients';

// Mock the API client
vi.mock('@/lib/api/fineract/clients');

describe('Client General Page data fetching', () => {
  beforeEach(() => {
    vi.resetAllMocks();
  });

  it('fetches accounts, charges, and collateral in parallel', async () => {
    const mockAccounts = { loanAccounts: [], savingsAccounts: [] };
    const mockCharges = [{ id: 1, name: 'Fee' }];
    const mockCollateral = [];

    vi.mocked(clientsApi.getAccounts).mockResolvedValue(mockAccounts);
    vi.mocked(clientsApi.getCharges).mockResolvedValue(mockCharges);
    vi.mocked(clientsApi.getCollateral).mockResolvedValue(mockCollateral);

    // Simulate the data fetching logic from the Server Component
    const [
      accounts,
      charges,
      collateral
    ] = await Promise.all([
      clientsApi.getAccounts('42'),
      clientsApi.getCharges('42'),
      clientsApi.getCollateral('42')
    ]);

    expect(accounts).toEqual(mockAccounts);
    expect(charges).toEqual(mockCharges);
    expect(collateral).toEqual(mockCollateral);

    expect(clientsApi.getAccounts).toHaveBeenCalledWith('42');
    expect(clientsApi.getCharges).toHaveBeenCalledWith('42');
    expect(clientsApi.getCollateral).toHaveBeenCalledWith('42');
  });
});
```

---

## 4. MSW (Mock Service Worker) Setup

### 4.1 Handler Definitions

Organize MSW handlers by Fineract API domain:

```typescript
// src/test/msw/handlers/clients.ts
import { http, HttpResponse } from 'msw';

const BASE = '/fineract-provider/api/v1';

export const clientHandlers = [
  // List clients
  http.get(`${BASE}/clients`, ({ request }) => {
    const url = new URL(request.url);
    const offset = parseInt(url.searchParams.get('offset') ?? '0');
    const limit = parseInt(url.searchParams.get('limit') ?? '20');

    return HttpResponse.json({
      totalFilteredRecords: 100,
      pageItems: Array.from({ length: limit }, (_, i) => ({
        id: offset + i + 1,
        accountNo: `00000000${offset + i + 1}`,
        displayName: `Client ${offset + i + 1}`,
        officeName: 'Head Office',
        status: { id: 300, code: 'clientStatusType.active', value: 'Active' }
      }))
    });
  }),

  // Get single client
  http.get(`${BASE}/clients/:clientId`, ({ params }) => {
    return HttpResponse.json({
      id: parseInt(params.clientId as string),
      accountNo: `00000000${params.clientId}`,
      displayName: `Test Client ${params.clientId}`,
      firstname: 'Test',
      lastname: `Client ${params.clientId}`,
      officeName: 'Head Office',
      officeId: 1,
      status: { id: 300, code: 'clientStatusType.active', value: 'Active' },
      active: true,
      activationDate: [
        2024,
        1,
        15
      ],
      imagePresent: false,
      timeline: {
        submittedOnDate: [
          2024,
          1,
          1
        ],
        activatedOnDate: [
          2024,
          1,
          15
        ]
      }
    });
  }),

  // Client template
  http.get(`${BASE}/clients/template`, () => {
    return HttpResponse.json({
      officeOptions: [{ id: 1, name: 'Head Office' }],
      staffOptions: [],
      genderOptions: [
        { id: 22, name: 'Male', isActive: true },
        { id: 24, name: 'Female', isActive: true }
      ],
      clientTypeOptions: [],
      clientClassificationOptions: [],
      clientLegalFormOptions: [{ id: 1, value: 'PERSON' }]
    });
  }),

  // Create client
  http.post(`${BASE}/clients`, async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ clientId: 999, resourceId: 999 });
  }),

  // Client accounts
  http.get(`${BASE}/clients/:clientId/accounts`, () => {
    return HttpResponse.json({
      loanAccounts: [],
      savingsAccounts: [],
      shareAccounts: []
    });
  }),

  // Client charges
  http.get(`${BASE}/clients/:clientId/charges`, () => {
    return HttpResponse.json([]);
  }),

  // Client notes
  http.get(`${BASE}/clients/:clientId/notes`, () => {
    return HttpResponse.json([]);
  })
];
```

```typescript
// src/test/msw/handlers/loans.ts
import { http, HttpResponse } from 'msw';

const BASE = '/fineract-provider/api/v1';

export const loanHandlers = [
  http.get(`${BASE}/loans/:loanId`, ({ params }) => {
    return HttpResponse.json({
      id: parseInt(params.loanId as string),
      accountNo: `L${params.loanId}`,
      status: { id: 300, code: 'loanStatusType.active', value: 'Active' },
      clientId: 1,
      clientName: 'Test Client',
      loanProductId: 1,
      loanProductName: 'Standard Loan',
      principal: 10000,
      currency: { code: 'USD', name: 'US Dollar', decimalPlaces: 2 }
      // ... more fields
    });
  }),

  http.get(`${BASE}/loans/template`, () => {
    return HttpResponse.json({
      productOptions: [{ id: 1, name: 'Standard Loan' }]
      // ... template data
    });
  }),

  http.post(`${BASE}/loans`, async () => {
    return HttpResponse.json({ loanId: 100, resourceId: 100 });
  })
];
```

```typescript
// src/test/msw/handlers/index.ts
import { clientHandlers } from './clients';
import { loanHandlers } from './loans';
// Import more handlers as modules are migrated

export const handlers = [
  ...clientHandlers,
  ...loanHandlers
  // ...accountingHandlers,
  // ...productsHandlers,
  // ...organizationHandlers,
  // ...systemHandlers,
];
```

### 4.2 Server Setup (Vitest)

```typescript
// src/test/msw/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);

// In setup.ts, add:
// beforeAll(() => server.listen({ onUnhandledRequest: 'warn' }));
// afterEach(() => server.resetHandlers());
// afterAll(() => server.close());
```

Update `src/test/setup.ts`:

```typescript
// Add to existing setup.ts
import { server } from './msw/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'warn' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### 4.3 Browser Setup (Development / Storybook)

```typescript
// src/test/msw/browser.ts
import { setupWorker } from 'msw/browser';
import { handlers } from './handlers';

export const worker = setupWorker(...handlers);
```

Initialize in development when Fineract backend is unavailable:

```typescript
// src/lib/msw-init.ts
export async function initMSW() {
  if (process.env.NEXT_PUBLIC_ENABLE_MSW === 'true') {
    const { worker } = await import('@/test/msw/browser');
    await worker.start({
      onUnhandledRequest: 'bypass',
      serviceWorker: { url: '/mockServiceWorker.js' }
    });
  }
}
```

---

## 5. E2E Test Patterns with Playwright

### 5.1 Playwright Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: process.env.CI ? [
        ['html'],
        ['github']
      ] : [[
          'html',
          { open: 'on-failure' }]],
  use: {
    baseURL: process.env.PLAYWRIGHT_BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure'
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    // Mobile viewport
    { name: 'mobile-chrome', use: { ...devices['Pixel 5'] } }
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000
  }
});
```

### 5.2 Page Object Model

```typescript
// e2e/pages/login.page.ts
import { Page, Locator, expect } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly usernameInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.usernameInput = page.getByLabel('Username');
    this.passwordInput = page.getByLabel('Password');
    this.loginButton = page.getByRole('button', { name: /sign in|log in/i });
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(username: string, password: string) {
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
    await this.page.waitForURL('/home');
  }
}
```

```typescript
// e2e/pages/clients.page.ts
import { Page, Locator, expect } from '@playwright/test';

export class ClientsPage {
  readonly page: Page;
  readonly heading: Locator;
  readonly searchInput: Locator;
  readonly createButton: Locator;
  readonly table: Locator;

  constructor(page: Page) {
    this.page = page;
    this.heading = page.getByRole('heading', { name: /clients/i });
    this.searchInput = page.getByPlaceholder(/search/i);
    this.createButton = page.getByRole('link', { name: /create/i });
    this.table = page.getByRole('table');
  }

  async goto() {
    await this.page.goto('/clients');
    await expect(this.heading).toBeVisible();
  }

  async search(query: string) {
    await this.searchInput.fill(query);
    await this.searchInput.press('Enter');
    await this.page.waitForLoadState('networkidle');
  }

  async clickClient(name: string) {
    await this.page.getByRole('link', { name }).click();
  }

  async getRowCount(): Promise<number> {
    return this.table.getByRole('row').count() - 1; // minus header
  }
}
```

```typescript
// e2e/pages/client-create.page.ts
import { Page, Locator, expect } from '@playwright/test';

export class ClientCreatePage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  async goto() {
    await this.page.goto('/clients/create');
  }

  async fillGeneralStep(data: { office: string; firstName: string; lastName: string }) {
    await this.page.getByLabel(/office/i).selectOption({ label: data.office });
    await this.page.getByLabel(/first name/i).fill(data.firstName);
    await this.page.getByLabel(/last name/i).fill(data.lastName);
  }

  async nextStep() {
    await this.page.getByRole('button', { name: /next/i }).click();
  }

  async submit() {
    await this.page.getByRole('button', { name: /submit/i }).click();
  }

  async expectSuccess() {
    await expect(this.page.getByText(/created successfully/i)).toBeVisible();
  }
}
```

### 5.3 Authentication Flow Test

```typescript
// e2e/tests/auth.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/login.page';

test.describe('Authentication', () => {
  test('should redirect to login when not authenticated', async ({ page }) => {
    await page.goto('/clients');
    await expect(page).toHaveURL(/\/login/);
  });

  test('should login successfully with valid credentials', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login(process.env.TEST_USERNAME || 'mifos', process.env.TEST_PASSWORD || 'password');
    await expect(page).toHaveURL('/home');
  });

  test('should show error with invalid credentials', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('invalid', 'invalid');
    await expect(page.getByText(/incorrect/i)).toBeVisible();
  });
});
```

### 5.4 Client Creation Wizard E2E

```typescript
// e2e/tests/clients/create-client.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../../pages/login.page';
import { ClientCreatePage } from '../../pages/client-create.page';

test.describe('Create Client', () => {
  test.beforeEach(async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('mifos', 'password');
  });

  test('should create a new client through the wizard', async ({ page }) => {
    const createPage = new ClientCreatePage(page);
    await createPage.goto();

    // Step 1: General
    await createPage.fillGeneralStep({
      office: 'Head Office',
      firstName: 'E2E',
      lastName: `Test ${Date.now()}`
    });
    await createPage.nextStep();

    // Step 2: Family Members (skip)
    await createPage.nextStep();

    // Step 3: Address (skip)
    await createPage.nextStep();

    // Step 4: Data Tables (skip)
    await createPage.nextStep();

    // Step 5: Preview and submit
    await createPage.submit();

    // Verify redirect to client detail page
    await expect(page).toHaveURL(/\/clients\/\d+/);
    await expect(page.getByText('E2E')).toBeVisible();
  });
});
```

### 5.5 Loan Application and Approval E2E

```typescript
// e2e/tests/loans/loan-lifecycle.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../../pages/login.page';

test.describe('Loan Lifecycle', () => {
  let clientId: string;

  test.beforeAll(async ({ browser }) => {
    // Create a test client first
    const page = await browser.newPage();
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('mifos', 'password');

    // Navigate to existing test client or create one
    await page.goto('/clients');
    // ... setup
    await page.close();
  });

  test('should apply for a loan', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('mifos', 'password');

    await page.goto(`/clients/${clientId}/loans-accounts/create`);

    // Fill loan details
    await page.getByLabel(/product/i).selectOption({ index: 1 });
    await page.waitForLoadState('networkidle'); // Wait for template to load

    await page.getByLabel(/principal/i).fill('10000');
    await page.getByLabel(/loan term/i).fill('12');

    // Navigate through wizard steps
    await page.getByRole('button', { name: /next/i }).click();
    // ... fill remaining steps

    await page.getByRole('button', { name: /submit/i }).click();

    await expect(page).toHaveURL(/\/loans-accounts\/\d+/);
  });

  test('should approve a loan', async ({ page }) => {
    // ... navigate to loan actions/approve
    // ... fill approval form
    // ... verify status change
  });

  test('should disburse a loan', async ({ page }) => {
    // ... navigate to loan actions/disburse
    // ... fill disbursement form
    // ... verify status change
  });
});
```

---

## 6. Test Coverage Goals

### 6.1 Coverage Targets

| Category                  | Target        | Rationale                  |
| ------------------------- | ------------- | -------------------------- |
| Unit tests (hooks, utils) | 80% lines     | Core business logic        |
| Component tests           | 70% lines     | Interactive UI behavior    |
| API client functions      | 90% lines     | Critical integration layer |
| E2E critical paths        | 100% coverage | Revenue-critical workflows |

### 6.2 Critical Paths Requiring 100% E2E Coverage

1. **Authentication:** Login, logout, session expiry, token refresh
2. **Client creation:** Full wizard flow
3. **Loan application:** Create, approve, disburse, repayment
4. **Savings:** Create account, deposit, withdrawal
5. **Journal entry:** Create, search, view
6. **User management:** Create user, assign roles
7. **Product creation:** Loan product full wizard
8. **Report generation:** Run a parameterized report

### 6.3 Coverage Exclusions

- Generated type files (`*.d.ts`)
- Test files themselves
- Configuration files
- Static asset imports
- Layout files with minimal logic

---

## 7. CI Integration -- GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run test:coverage
      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/

  build:
    runs-on: ubuntu-latest
    needs: [lint, unit-tests]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - name: Upload build
        uses: actions/upload-artifact@v4
        with:
          name: nextjs-build
          path: .next/

  e2e-tests:
    runs-on: ubuntu-latest
    needs: [build]
    strategy:
      matrix:
        project: [chromium, firefox]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npx playwright install --with-deps ${{ matrix.project }}
      - name: Download build
        uses: actions/download-artifact@v4
        with:
          name: nextjs-build
          path: .next/
      - name: Run E2E tests
        run: npx playwright test --project=${{ matrix.project }}
        env:
          PLAYWRIGHT_BASE_URL: http://localhost:3000
          FINERACT_API_URL: ${{ secrets.E2E_FINERACT_URL }}
          TEST_USERNAME: ${{ secrets.E2E_USERNAME }}
          TEST_PASSWORD: ${{ secrets.E2E_PASSWORD }}
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report-${{ matrix.project }}
          path: playwright-report/
```

### 7.1 Pre-commit Hooks

```json
// package.json (lint-staged config)
{
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "vitest related --run"
    ]
  }
}
```

---

## 8. Test File Naming and Organization

```
src/
├── lib/
│   ├── hooks/
│   │   ├── clients.ts
│   │   └── __tests__/
│   │       └── clients.test.ts
│   ├── api/
│   │   └── fineract/
│   │       ├── clients.ts
│   │       └── __tests__/
│   │           └── clients.test.ts
│   └── utils/
│       ├── date.ts
│       └── __tests__/
│           └── date.test.ts
├── components/
│   └── clients/
│       ├── client-list-table.tsx
│       └── __tests__/
│           └── client-list-table.test.tsx
├── test/
│   ├── setup.ts
│   ├── test-utils.tsx
│   └── msw/
│       ├── server.ts
│       ├── browser.ts
│       └── handlers/
│           ├── index.ts
│           ├── clients.ts
│           ├── loans.ts
│           ├── accounting.ts
│           └── ...
e2e/
├── pages/                       # Page object models
│   ├── login.page.ts
│   ├── clients.page.ts
│   └── client-create.page.ts
├── tests/
│   ├── auth.spec.ts
│   ├── clients/
│   │   ├── list.spec.ts
│   │   └── create-client.spec.ts
│   ├── loans/
│   │   └── loan-lifecycle.spec.ts
│   └── accounting/
│       └── journal-entries.spec.ts
└── fixtures/                    # Test data files
    ├── clients.json
    └── loans.json
```
