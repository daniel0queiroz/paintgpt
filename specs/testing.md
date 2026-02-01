# Especificação: Testes (TDD)

## Visão Geral

Estratégia de testes usando Test-Driven Development (TDD) com Vitest, Testing Library e Playwright.

## Filosofia TDD

**Nunca escreva código de produção sem um teste falhando primeiro.**

### Ciclo Red-Green-Refactor

1. **RED** - Escreva um teste que falha
2. **GREEN** - Escreva o código mínimo para passar
3. **REFACTOR** - Melhore sem quebrar testes

```typescript
// 1. RED - Escreva o teste primeiro
describe('calculateCredits', () => {
  it('should return 10 credits for starter pack', () => {
    expect(calculateCredits('starter')).toBe(10);
  });
});

// 2. GREEN - Implemente o mínimo
const calculateCredits = (pack: string) => {
  if (pack === 'starter') return 10;
  return 0;
};

// 3. REFACTOR - Melhore se necessário
const CREDIT_PACKS = { starter: 10, pro: 50, unlimited: 100 } as const;
const calculateCredits = (pack: keyof typeof CREDIT_PACKS) => CREDIT_PACKS[pack];
```

---

## Estrutura de Arquivos

```
paintgpt/
├── __tests__/
│   ├── api/
│   │   ├── generate.test.ts
│   │   ├── images.test.ts
│   │   ├── gallery.test.ts
│   │   ├── credits.test.ts
│   │   ├── checkout.test.ts
│   │   └── webhooks/
│   │       ├── clerk.test.ts
│   │       └── stripe.test.ts
│   ├── components/
│   │   ├── PromptInput.test.tsx
│   │   ├── ImageCard.test.tsx
│   │   ├── Gallery.test.tsx
│   │   ├── CreditDisplay.test.tsx
│   │   └── DownloadButton.test.tsx
│   ├── hooks/
│   │   ├── useCredits.test.ts
│   │   ├── useImages.test.ts
│   │   └── useGenerate.test.ts
│   └── lib/
│       ├── prompt-validation.test.ts
│       ├── credit-calculation.test.ts
│       └── pdf-generation.test.ts
├── e2e/
│   ├── auth.spec.ts
│   ├── generation.spec.ts
│   ├── gallery.spec.ts
│   └── purchase.spec.ts
├── vitest.config.ts
├── vitest.setup.ts
└── playwright.config.ts
```

---

## Ferramentas

### Vitest (Unitários/Integração)

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./vitest.setup.ts'],
    include: ['__tests__/**/*.test.{ts,tsx}'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: ['node_modules/', 'e2e/', '**/*.config.*'],
      thresholds: {
        statements: 80,
        branches: 80,
        functions: 80,
        lines: 80,
      },
    },
  },
  resolve: {
    alias: {
      '@': resolve(__dirname, './'),
    },
  },
});
```

### Setup File

```typescript
// vitest.setup.ts
import '@testing-library/jest-dom/vitest';
import { cleanup } from '@testing-library/react';
import { afterEach, vi } from 'vitest';
import { server } from './__tests__/mocks/server';

// MSW Server
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => {
  cleanup();
  server.resetHandlers();
});
afterAll(() => server.close());

// Mock Next.js router
vi.mock('next/navigation', () => ({
  useRouter: () => ({
    push: vi.fn(),
    replace: vi.fn(),
    back: vi.fn(),
  }),
  useSearchParams: () => new URLSearchParams(),
  usePathname: () => '/',
}));
```

### Testing Library

```typescript
// Exemplo de teste de componente
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { PromptInput } from '@/components/PromptInput';

describe('PromptInput', () => {
  it('should submit prompt when form is submitted', async () => {
    const onSubmit = vi.fn();
    render(<PromptInput onSubmit={onSubmit} />);

    const input = screen.getByPlaceholderText(/describe your coloring page/i);
    const button = screen.getByRole('button', { name: /generate/i });

    await userEvent.type(input, 'a cute dinosaur');
    await userEvent.click(button);

    expect(onSubmit).toHaveBeenCalledWith('a cute dinosaur');
  });

  it('should disable submit when prompt is too short', () => {
    render(<PromptInput onSubmit={vi.fn()} />);

    const input = screen.getByPlaceholderText(/describe your coloring page/i);
    const button = screen.getByRole('button', { name: /generate/i });

    fireEvent.change(input, { target: { value: 'ab' } });

    expect(button).toBeDisabled();
  });
});
```

### MSW (Mock Service Worker)

```typescript
// __tests__/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.post('/api/generate', async ({ request }) => {
    const { prompt } = await request.json();

    if (prompt.length < 3) {
      return HttpResponse.json({
        success: false,
        error: { code: 'INVALID_PROMPT', message: 'Prompt too short' }
      }, { status: 400 });
    }

    return HttpResponse.json({
      success: true,
      image: {
        id: 'test-id-123',
        url: 'https://example.com/test-image.png',
        prompt,
      },
      creditsRemaining: 9,
    });
  }),

  http.get('/api/credits', () => {
    return HttpResponse.json({
      credits: 10,
      canGenerate: true,
    });
  }),

  http.get('/api/gallery', () => {
    return HttpResponse.json({
      images: [
        { id: '1', prompt: 'dinosaur', imageUrl: 'https://example.com/1.png' },
        { id: '2', prompt: 'unicorn', imageUrl: 'https://example.com/2.png' },
      ],
      nextCursor: null,
      hasMore: false,
    });
  }),
];

// __tests__/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

### Playwright (E2E)

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    { name: 'Mobile Chrome', use: { ...devices['Pixel 5'] } },
    { name: 'Mobile Safari', use: { ...devices['iPhone 12'] } },
  ],
  webServer: {
    command: 'bun run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

---

## Padrões de Teste

### API Routes

```typescript
// __tests__/api/generate.test.ts
import { POST } from '@/app/api/generate/route';
import { NextRequest } from 'next/server';

// Mock dependencies
vi.mock('@clerk/nextjs/server', () => ({
  auth: vi.fn(() => ({ userId: 'user_123' })),
}));

vi.mock('@/lib/db', () => ({
  db: {
    select: vi.fn(),
    insert: vi.fn(),
    update: vi.fn(),
  },
}));

describe('POST /api/generate', () => {
  it('should return 401 when not authenticated', async () => {
    vi.mocked(auth).mockReturnValueOnce({ userId: null });

    const request = new NextRequest('http://localhost/api/generate', {
      method: 'POST',
      body: JSON.stringify({ prompt: 'a cat' }),
    });

    const response = await POST(request);
    const data = await response.json();

    expect(response.status).toBe(401);
    expect(data.error.code).toBe('UNAUTHORIZED');
  });

  it('should return 402 when user has no credits', async () => {
    vi.mocked(db.select).mockResolvedValueOnce([{ credits: 0 }]);

    const request = new NextRequest('http://localhost/api/generate', {
      method: 'POST',
      body: JSON.stringify({ prompt: 'a cat' }),
    });

    const response = await POST(request);
    const data = await response.json();

    expect(response.status).toBe(402);
    expect(data.error.code).toBe('INSUFFICIENT_CREDITS');
  });

  it('should generate image and debit credit on success', async () => {
    vi.mocked(db.select).mockResolvedValueOnce([{ credits: 5 }]);

    const request = new NextRequest('http://localhost/api/generate', {
      method: 'POST',
      body: JSON.stringify({ prompt: 'a cute cat playing' }),
    });

    const response = await POST(request);
    const data = await response.json();

    expect(response.status).toBe(200);
    expect(data.success).toBe(true);
    expect(data.image).toBeDefined();
    expect(data.creditsRemaining).toBe(4);
  });
});
```

### Componentes React

```typescript
// __tests__/components/CreditDisplay.test.tsx
import { render, screen } from '@testing-library/react';
import { CreditDisplay } from '@/components/CreditDisplay';

describe('CreditDisplay', () => {
  it('should show credit count', () => {
    render(<CreditDisplay credits={10} />);
    expect(screen.getByText('10 credits')).toBeInTheDocument();
  });

  it('should show warning when credits are low', () => {
    render(<CreditDisplay credits={1} />);
    expect(screen.getByText('1 credit')).toBeInTheDocument();
    expect(screen.getByRole('alert')).toHaveTextContent(/running low/i);
  });

  it('should show buy button when no credits', () => {
    render(<CreditDisplay credits={0} />);
    expect(screen.getByRole('button', { name: /buy credits/i })).toBeInTheDocument();
  });
});
```

### Hooks

```typescript
// __tests__/hooks/useCredits.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { useCredits } from '@/hooks/useCredits';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const wrapper = ({ children }) => (
  <QueryClientProvider client={new QueryClient()}>
    {children}
  </QueryClientProvider>
);

describe('useCredits', () => {
  it('should fetch and return credits', async () => {
    const { result } = renderHook(() => useCredits(), { wrapper });

    await waitFor(() => {
      expect(result.current.isLoading).toBe(false);
    });

    expect(result.current.credits).toBe(10);
    expect(result.current.canGenerate).toBe(true);
  });

  it('should refetch credits after mutation', async () => {
    const { result } = renderHook(() => useCredits(), { wrapper });

    await waitFor(() => expect(result.current.isLoading).toBe(false));

    await result.current.refetch();

    expect(result.current.credits).toBeDefined();
  });
});
```

### Utilitários/Lib

```typescript
// __tests__/lib/prompt-validation.test.ts
import { validatePrompt, sanitizePrompt, isPromptSafe } from '@/lib/prompt-validation';

describe('validatePrompt', () => {
  it('should reject prompts shorter than 3 characters', () => {
    expect(validatePrompt('ab')).toEqual({
      valid: false,
      error: 'Prompt must be at least 3 characters',
    });
  });

  it('should reject prompts longer than 500 characters', () => {
    const longPrompt = 'a'.repeat(501);
    expect(validatePrompt(longPrompt)).toEqual({
      valid: false,
      error: 'Prompt must be at most 500 characters',
    });
  });

  it('should accept valid prompts', () => {
    expect(validatePrompt('a cute dinosaur')).toEqual({ valid: true });
  });
});

describe('sanitizePrompt', () => {
  it('should trim whitespace', () => {
    expect(sanitizePrompt('  hello world  ')).toBe('hello world');
  });

  it('should remove special characters', () => {
    expect(sanitizePrompt('hello<script>world')).toBe('helloworld');
  });
});

describe('isPromptSafe', () => {
  it('should reject inappropriate content', () => {
    expect(isPromptSafe('violent content here')).toBe(false);
  });

  it('should accept safe content', () => {
    expect(isPromptSafe('a happy puppy')).toBe(true);
  });
});
```

---

## Testes E2E

```typescript
// e2e/generation.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Image Generation', () => {
  test.beforeEach(async ({ page }) => {
    // Setup auth state
    await page.goto('/');
  });

  test('should generate image from prompt', async ({ page }) => {
    await page.goto('/generate');

    await page.fill('[data-testid="prompt-input"]', 'a friendly dragon');
    await page.click('[data-testid="generate-button"]');

    // Wait for generation
    await expect(page.locator('[data-testid="generated-image"]')).toBeVisible({
      timeout: 60000,
    });

    // Verify credits decreased
    await expect(page.locator('[data-testid="credit-count"]')).toHaveText('9');
  });

  test('should show error when no credits', async ({ page }) => {
    // Set up user with 0 credits
    await page.goto('/generate');

    await page.fill('[data-testid="prompt-input"]', 'a cat');
    await page.click('[data-testid="generate-button"]');

    await expect(page.locator('[data-testid="error-message"]')).toContainText(
      /need credits/i
    );
    await expect(page.locator('[data-testid="buy-credits-link"]')).toBeVisible();
  });
});

// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test('should redirect to sign-in when accessing protected route', async ({ page }) => {
    await page.goto('/generate');
    await expect(page).toHaveURL(/sign-in/);
  });

  test('should show user menu when signed in', async ({ page }) => {
    // Authenticate
    await page.goto('/');
    await expect(page.locator('[data-testid="user-button"]')).toBeVisible();
  });
});
```

---

## Cobertura de Código

### Requisitos Mínimos

| Métrica | Mínimo |
|---------|--------|
| Statements | 80% |
| Branches | 80% |
| Functions | 80% |
| Lines | 80% |

### Exceções

Arquivos que podem ser excluídos da cobertura:
- Arquivos de configuração (`*.config.*`)
- Arquivos de tipos (`*.d.ts`)
- Mocks (`__tests__/mocks/*`)
- Migrações de banco

---

## Testes por Feature

### Autenticação

| Teste | Tipo | Arquivo |
|-------|------|---------|
| Login com email/senha | E2E | `e2e/auth.spec.ts` |
| Login com Google | E2E | `e2e/auth.spec.ts` |
| Webhook user.created | Integração | `__tests__/api/webhooks/clerk.test.ts` |
| Proteção de rotas | E2E | `e2e/auth.spec.ts` |

### Geração de Imagens

| Teste | Tipo | Arquivo |
|-------|------|---------|
| Validação de prompt | Unit | `__tests__/lib/prompt-validation.test.ts` |
| Débito de crédito | Integração | `__tests__/api/generate.test.ts` |
| Upload para R2 | Integração | `__tests__/lib/storage.test.ts` |
| Refund em falha | Integração | `__tests__/api/generate.test.ts` |
| Fluxo completo | E2E | `e2e/generation.spec.ts` |

### Galeria

| Teste | Tipo | Arquivo |
|-------|------|---------|
| Listagem pública | Integração | `__tests__/api/gallery.test.ts` |
| Paginação | Integração | `__tests__/api/gallery.test.ts` |
| Toggle visibilidade | Integração | `__tests__/api/images.test.ts` |
| Componente ImageCard | Unit | `__tests__/components/ImageCard.test.tsx` |

### Pagamentos

| Teste | Tipo | Arquivo |
|-------|------|---------|
| Criar checkout session | Integração | `__tests__/api/checkout.test.ts` |
| Webhook checkout.completed | Integração | `__tests__/api/webhooks/stripe.test.ts` |
| Adicionar créditos | Integração | `__tests__/api/webhooks/stripe.test.ts` |
| Fluxo de compra | E2E | `e2e/purchase.spec.ts` |

---

## Scripts package.json

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:all": "vitest run && playwright test"
  }
}
```

---

## CI/CD

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bun run test:coverage
      - uses: codecov/codecov-action@v4

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install
      - run: bunx playwright install --with-deps
      - run: bun run test:e2e
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
```

---

## Checklist de Implementação

Antes de mergear qualquer PR:

- [ ] Todos os testes existentes passam
- [ ] Novos testes foram adicionados para novas features
- [ ] Cobertura de código >= 80%
- [ ] Testes E2E passam
- [ ] Nenhum teste está pulado (`skip`, `only`)
- [ ] Testes são independentes (não dependem de ordem)
