# Especificação: Integração Stripe

## Visão Geral

Sistema de pagamentos usando Stripe Checkout para compra de créditos.

## Modo de Integração

**Stripe Checkout** (hosted) - Página de pagamento hospedada pelo Stripe.

Vantagens:
- PCI compliance automático
- UI otimizada para conversão
- Suporte a múltiplos métodos de pagamento
- Menos código para manter

## Produto

| Campo | Valor |
|-------|-------|
| Nome | PaintGPT Credits |
| Descrição | 20 credits for AI coloring pages |
| Preço | $9.00 USD |
| Tipo | One-time payment |
| Créditos | 20 |

## Fluxo de Compra

```
[Pricing Page] → [Create Session] → [Stripe Checkout] → [Payment] → [Webhook] → [Credits Added]
```

### Detalhado:

1. Usuário clica "Buy Credits" em `/pricing`
2. Frontend chama `POST /api/checkout`
3. Backend cria Checkout Session no Stripe
4. Backend salva `purchases` com status `pending`
5. Frontend redireciona para Stripe Checkout URL
6. Usuário completa pagamento
7. Stripe envia webhook `checkout.session.completed`
8. Backend atualiza `purchases` para `completed`
9. Backend adiciona créditos ao usuário
10. Usuário redirecionado para `/generate?success=true`

## API Endpoints

### POST /api/checkout

Cria sessão de checkout.

**Request:**
```typescript
// Não precisa de body - pacote único no MVP
{}
```

**Response:**
```typescript
{
  url: string // Stripe Checkout URL
}
```

**Implementação:**
```typescript
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(request: Request) {
  const { userId } = auth();
  if (!userId) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Buscar usuário do banco
  const user = await db.query.users.findFirst({
    where: eq(users.clerkId, userId)
  });

  if (!user) {
    return Response.json({ error: 'User not found' }, { status: 404 });
  }

  // Criar sessão Stripe
  const session = await stripe.checkout.sessions.create({
    mode: 'payment',
    payment_method_types: ['card'],
    line_items: [
      {
        price_data: {
          currency: 'usd',
          product_data: {
            name: 'PaintGPT Credits',
            description: '20 credits for AI coloring pages',
          },
          unit_amount: 900, // $9.00 em centavos
        },
        quantity: 1,
      },
    ],
    success_url: `${process.env.NEXT_PUBLIC_APP_URL}/generate?success=true`,
    cancel_url: `${process.env.NEXT_PUBLIC_APP_URL}/pricing?canceled=true`,
    client_reference_id: user.id,
    customer_email: user.email,
    metadata: {
      userId: user.id,
      credits: '20',
    },
  });

  // Salvar purchase pendente
  await db.insert(purchases).values({
    userId: user.id,
    stripeSessionId: session.id,
    amountCents: 900,
    credits: 20,
    status: 'pending',
  });

  return Response.json({ url: session.url });
}
```

### POST /api/webhooks/stripe

Processa webhooks do Stripe.

**Implementação:**
```typescript
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(request: Request) {
  const body = await request.text();
  const signature = request.headers.get('stripe-signature')!;

  let event: Stripe.Event;

  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    console.error('Webhook signature verification failed');
    return Response.json({ error: 'Invalid signature' }, { status: 400 });
  }

  if (event.type === 'checkout.session.completed') {
    const session = event.data.object as Stripe.Checkout.Session;

    await handleCheckoutCompleted(session);
  }

  return Response.json({ received: true });
}

async function handleCheckoutCompleted(session: Stripe.Checkout.Session) {
  const { userId, credits } = session.metadata!;

  // Verificar idempotência
  const existingPurchase = await db.query.purchases.findFirst({
    where: and(
      eq(purchases.stripeSessionId, session.id),
      eq(purchases.status, 'completed')
    )
  });

  if (existingPurchase) {
    console.log('Purchase already processed:', session.id);
    return;
  }

  // Transação: atualizar purchase + adicionar créditos
  await db.transaction(async (tx) => {
    await tx
      .update(purchases)
      .set({
        status: 'completed',
        stripePaymentIntentId: session.payment_intent as string,
        completedAt: new Date()
      })
      .where(eq(purchases.stripeSessionId, session.id));

    await tx
      .update(users)
      .set({
        credits: sql`credits + ${parseInt(credits)}`,
        updatedAt: new Date()
      })
      .where(eq(users.id, userId));
  });

  console.log(`Added ${credits} credits to user ${userId}`);
}
```

## Webhook Events

| Evento | Ação |
|--------|------|
| `checkout.session.completed` | Creditar conta |
| `checkout.session.expired` | Marcar purchase como `failed` |
| `charge.refunded` | Debitar créditos, marcar como `refunded` |

## Configuração Stripe

### Dashboard
1. Criar conta Stripe
2. Ativar modo de produção quando pronto
3. Configurar webhook endpoint
4. Copiar chaves

### Webhook Endpoint
```
URL: https://paintgpt.com/api/webhooks/stripe
Events: checkout.session.completed, checkout.session.expired, charge.refunded
```

## Variáveis de Ambiente

```env
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...
```

## UI - Página de Pricing

```
┌─────────────────────────────────────┐
│                                     │
│         Get Started with            │
│           PaintGPT                  │
│                                     │
│  ┌─────────────────────────────┐   │
│  │                             │   │
│  │     20 Credits - $9         │   │
│  │                             │   │
│  │  ✓ Generate 20 unique       │   │
│  │    coloring pages           │   │
│  │  ✓ Download as PDF          │   │
│  │  ✓ Share to gallery         │   │
│  │  ✓ Credits never expire     │   │
│  │                             │   │
│  │     [Buy Credits →]         │   │
│  │                             │   │
│  └─────────────────────────────┘   │
│                                     │
│  Secure payment via Stripe          │
│  [Visa] [Mastercard] [Amex]         │
│                                     │
└─────────────────────────────────────┘
```

## Segurança

- Validar assinatura do webhook
- Não confiar em dados do frontend para preço/quantidade
- Usar `client_reference_id` para vincular ao usuário
- Verificar idempotência em webhooks
- Logs de todas as transações

## Testes

### Modo de Teste
- Usar chaves `sk_test_` e `pk_test_`
- Cartão de teste: `4242 4242 4242 4242`
- Testar webhook com Stripe CLI: `stripe listen --forward-to localhost:3000/api/webhooks/stripe`

## Métricas

- Taxa de conversão (visits → checkouts)
- Taxa de abandono (checkouts → completed)
- Receita total
- Ticket médio

## Testes TDD

### Unit Tests

#### Cálculo de Créditos por Pacote

| Teste ID | Descrição | Input | Expected Output | Critério de Sucesso |
|----------|-----------|-------|-----------------|---------------------|
| UT-001 | Calcular créditos para pacote básico | pacoteId: 'basic' | 20 créditos | Retorna 20 |
| UT-002 | Calcular créditos para pacote premium | pacoteId: 'premium' | 50 créditos | Retorna 50 |
| UT-003 | Calcular créditos para pacote enterprise | pacoteId: 'enterprise' | 200 créditos | Retorna 200 |
| UT-004 | Rejeitar pacote inválido | pacoteId: 'invalid' | Error | Lança exceção "Invalid package" |
| UT-005 | Calcular valor em centavos para pacote | pacoteId: 'basic' | 900 (centavos) | Retorna 900 |
| UT-006 | Converter créditos para valor USD corretamente | credits: 20 | $9.00 | Multiplicação correta (20 * 0.45) |

**Implementação de exemplo:**
```typescript
// utils/creditCalculator.ts
export function getCreditsForPackage(packageId: string): number {
  const packages: Record<string, number> = {
    'basic': 20,
    'premium': 50,
    'enterprise': 200,
  };

  if (!packages[packageId]) {
    throw new Error(`Invalid package: ${packageId}`);
  }

  return packages[packageId];
}

export function getAmountCentsForPackage(packageId: string): number {
  const amounts: Record<string, number> = {
    'basic': 900,    // $9.00
    'premium': 20.99, // $20.99
    'enterprise': 99.99, // $99.99
  };

  if (!amounts[packageId]) {
    throw new Error(`Invalid package: ${packageId}`);
  }

  return Math.round(amounts[packageId] * 100);
}
```

#### Verificação de Assinatura Webhook

| Teste ID | Descrição | Input | Expected Output | Critério de Sucesso |
|----------|-----------|-------|-----------------|---------------------|
| UT-007 | Validar assinatura webhook válida | signature válida | true | Retorna true |
| UT-008 | Rejeitar assinatura inválida | signature inválida | false | Lança erro "Invalid signature" |
| UT-009 | Rejeitar webhook sem signature | signature: undefined | Error | Lança exceção 400 |
| UT-010 | Validar timestamp do webhook (< 5 min) | timestamp válido | true | Retorna true |
| UT-011 | Rejeitar webhook expirado (> 5 min) | timestamp antigo | Error | Lança exceção "Webhook expired" |
| UT-012 | Construir event a partir de body e signature | body + signature válidos | Stripe.Event | Retorna evento correto |

**Implementação de exemplo:**
```typescript
// utils/webhookValidator.ts
import Stripe from 'stripe';

export function validateWebhookSignature(
  body: string,
  signature: string,
  secret: string
): Stripe.Event {
  if (!signature) {
    throw new Error('Missing stripe-signature header');
  }

  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

  try {
    const event = stripe.webhooks.constructEvent(body, signature, secret);
    return event;
  } catch (error) {
    throw new Error('Invalid webhook signature');
  }
}

export function isWebhookExpired(event: Stripe.Event, maxAgeSeconds = 300): boolean {
  const now = Math.floor(Date.now() / 1000);
  return now - event.created > maxAgeSeconds;
}
```

---

### Integration Tests

#### Criar Checkout Session

| Teste ID | Descrição | Setup | Action | Expected Result | Validações |
|----------|-----------|-------|--------|-----------------|------------|
| IT-001 | Criar checkout session com usuário autenticado | User autenticado, DB pronto | POST /api/checkout | { url: string } | URL inicia com https://checkout.stripe.com |
| IT-002 | Criar checkout session sem autenticação | Sem auth | POST /api/checkout | 401 Unauthorized | Response.status = 401 |
| IT-003 | Usuário não existe no BD | Auth OK, usuário não em BD | POST /api/checkout | 404 Not Found | Response.status = 404 |
| IT-004 | Salvar purchase pendente no BD | User OK, checkout criado | POST /api/checkout | Purchase com status='pending' | DB registra com stripeSessionId |
| IT-005 | Incluir metadata correta no Stripe | User OK | POST /api/checkout | Metadata contains userId + credits | metadata.userId = user.id, metadata.credits = '20' |
| IT-006 | Incluir email do usuário na sessão | User com email | POST /api/checkout | customer_email preenchido | session.customer_email = user.email |
| IT-007 | Incluir success_url correto | App URL configurada | POST /api/checkout | success_url aponta para /generate | Contains 'generate?success=true' |
| IT-008 | Incluir cancel_url correto | App URL configurada | POST /api/checkout | cancel_url aponta para /pricing | Contains 'pricing?canceled=true' |

**Implementação de exemplo:**
```typescript
// __tests__/integration/api/checkout.test.ts
import { POST } from '@/app/api/checkout/route';
import { db } from '@/db';
import { purchases } from '@/db/schema';

describe('POST /api/checkout', () => {
  it('should create checkout session for authenticated user', async () => {
    const mockAuth = { userId: 'user_123' };

    const response = await POST(new Request('http://localhost:3000/api/checkout'));
    const data = await response.json();

    expect(response.status).toBe(200);
    expect(data.url).toMatch(/^https:\/\/checkout\.stripe\.com/);
  });

  it('should save purchase with pending status', async () => {
    // Setup and execute...

    const purchase = await db.query.purchases.findFirst({
      where: eq(purchases.stripeSessionId, sessionId)
    });

    expect(purchase?.status).toBe('pending');
    expect(purchase?.credits).toBe(20);
  });

  it('should return 401 if not authenticated', async () => {
    const response = await POST(new Request('http://localhost:3000/api/checkout'));
    expect(response.status).toBe(401);
  });
});
```

#### Webhook: checkout.session.completed

| Teste ID | Descrição | Setup | Webhook Event | Expected Result | Validações |
|----------|-----------|-------|---------------|-----------------|------------|
| IT-009 | Processar webhook checkout completo | Purchase pendente no BD | checkout.session.completed | Purchase status = 'completed' | DB atualizado |
| IT-010 | Adicionar créditos ao usuário | User com credits=0, Purchase com credits=20 | checkout.session.completed | user.credits = 20 | Query user mostra credits aumentado |
| IT-011 | Idempotência: mesmo webhook 2x | Purchase já completed | checkout.session.completed (retry) | Sem duplicação de créditos | user.credits = 20 (não 40) |
| IT-012 | Usar transação atômica | Purchase + User updates | checkout.session.completed | Ambos atualizados ou nenhum | Sem estado inconsistente |
| IT-013 | Registrar stripePaymentIntentId | Session com payment_intent | checkout.session.completed | purchase.stripePaymentIntentId preenchido | Armazenar para futura auditoria |
| IT-014 | Registrar completedAt timestamp | Session recebida | checkout.session.completed | purchase.completedAt = now | Data/hora registrada |
| IT-015 | Extrair metadata corretamente | session.metadata = {userId, credits} | checkout.session.completed | Usar valores corretos | userId + credits do evento |
| IT-016 | Log de sucesso | Webhook processado | checkout.session.completed | Console.log com detalhes | Registra userId + credits adicionados |

**Implementação de exemplo:**
```typescript
// __tests__/integration/webhooks/stripe.test.ts
import { handleCheckoutCompleted } from '@/app/api/webhooks/stripe/handler';
import { db } from '@/db';
import { purchases, users } from '@/db/schema';

describe('handleCheckoutCompleted', () => {
  it('should update purchase status to completed', async () => {
    const session = mockStripeSession({ id: 'cs_123' });

    await handleCheckoutCompleted(session);

    const purchase = await db.query.purchases.findFirst({
      where: eq(purchases.stripeSessionId, 'cs_123')
    });
    expect(purchase?.status).toBe('completed');
  });

  it('should add credits to user', async () => {
    const userId = 'user_123';
    const initialCredits = 0;

    await db.update(users).set({ credits: initialCredits }).where(eq(users.id, userId));

    const session = mockStripeSession({
      id: 'cs_123',
      metadata: { userId, credits: '20' }
    });

    await handleCheckoutCompleted(session);

    const user = await db.query.users.findFirst({ where: eq(users.id, userId) });
    expect(user?.credits).toBe(20);
  });

  it('should be idempotent on duplicate webhooks', async () => {
    const session = mockStripeSession({ id: 'cs_123', metadata: { userId: 'user_123', credits: '20' } });

    await handleCheckoutCompleted(session);
    await handleCheckoutCompleted(session);

    const user = await db.query.users.findFirst({ where: eq(users.id, 'user_123') });
    expect(user?.credits).toBe(20);
  });

  it('should use atomic transaction', async () => {
    // Simular falha durante transaction
    // Verificar que nenhum update parcial ocorreu
  });
});
```

#### Webhook: charge.refunded

| Teste ID | Descrição | Setup | Webhook Event | Expected Result | Validações |
|----------|-----------|-------|---------------|-----------------|------------|
| IT-017 | Processar reembolso de cobrança | Purchase completed, créditos adicionados | charge.refunded | Purchase status = 'refunded' | DB atualizado |
| IT-018 | Debitar créditos do usuário | user.credits = 20, Purchase com credits=20 | charge.refunded | user.credits = 0 | Créditos removidos |
| IT-019 | Usar sessionId para encontrar purchase | charge event com session_id | charge.refunded | Encontra purchase correto | WHERE stripeSessionId = session_id |
| IT-020 | Registrar refund timestamp | Reembolso processado | charge.refunded | purchase.refundedAt = now | Data/hora registrada |
| IT-021 | Log de reembolso | Webhook processado | charge.refunded | Console.log com detalhes | Registra userId + créditos removidos |
| IT-022 | Idempotência em reembolso | Mesmo webhook 2x | charge.refunded | Sem duplicação de débito | user.credits correto |

**Implementação de exemplo:**
```typescript
// __tests__/integration/webhooks/refund.test.ts
import { handleChargeRefunded } from '@/app/api/webhooks/stripe/handler';
import { db } from '@/db';
import { purchases, users } from '@/db/schema';

describe('handleChargeRefunded', () => {
  it('should mark purchase as refunded', async () => {
    const event = mockStripeEvent({
      type: 'charge.refunded',
      data: { object: mockChargeRefunded({ id: 'ch_123' }) }
    });

    await handleChargeRefunded(event.data.object);

    const purchase = await db.query.purchases.findFirst({
      where: eq(purchases.stripePaymentIntentId, 'ch_123')
    });
    expect(purchase?.status).toBe('refunded');
  });

  it('should debit credits from user', async () => {
    const userId = 'user_123';

    await db.update(users).set({ credits: 20 }).where(eq(users.id, userId));

    const charge = mockChargeRefunded({
      id: 'ch_123',
      metadata: { userId }
    });

    await handleChargeRefunded(charge);

    const user = await db.query.users.findFirst({ where: eq(users.id, userId) });
    expect(user?.credits).toBe(0);
  });
});
```

#### Adicionar Créditos ao Usuário

| Teste ID | Descrição | Setup | Action | Expected Result | Validações |
|----------|-----------|-------|--------|-----------------|------------|
| IT-023 | Adicionar créditos com valor > 0 | user.credits = 0 | addCredits(userId, 20) | user.credits = 20 | DB reflete mudança |
| IT-024 | Adicionar múltiplos lotes | user.credits = 0 | addCredits(userId, 20) x 3 | user.credits = 60 | Soma corretamente |
| IT-025 | Não permitir créditos negativos | user.credits = 10 | addCredits(userId, -20) | Error | Lança exceção "Invalid amount" |
| IT-026 | Atualizar timestamp updatedAt | user.updatedAt = old | addCredits(userId, 20) | user.updatedAt = now | Reflete data/hora atual |
| IT-027 | Registrar histórico (auditoria) | Nenhum histórico | addCredits(userId, 20) | Entrada em creditHistory | Log da transação |

---

### E2E Tests

#### Fluxo Completo de Compra (Stripe Test Mode)

| Teste ID | Descrição | Passos | Expected Flow | Validações |
|----------|-----------|--------|----------------|------------|
| E2E-001 | Compra completa de créditos (navegador) | 1. Acesso /pricing<br>2. Click "Buy Credits"<br>3. Preencher stripe form (4242 4242...)<br>4. Submit<br>5. Redirecionado para /generate?success=true | Fluxo completo sem erros | 1. Página pricing carrega<br>2. Modal/checkout abre<br>3. Stripe Checkout renderiza<br>4. Success page mostra "Credits Added"<br>5. User credits aumentou no DB |
| E2E-002 | Cancelar checkout | 1. Acesso /pricing<br>2. Click "Buy Credits"<br>3. Click "Cancel" no Stripe<br>4. Retorna para /pricing?canceled=true | Voltar para pricing sem compra | 1. Redirect correto<br>2. Mensagem "Cancelled" exibida<br>3. Purchase status permanece 'pending'<br>4. Créditos não adicionados |
| E2E-003 | Cartão rejeitado | 1. Acesso /pricing<br>2. Click "Buy Credits"<br>3. Preencher cartão inválido (4000 0000...)<br>4. Submit | Erro exibido no Stripe Checkout | 1. Stripe mostra mensagem de erro<br>2. Não redireciona<br>3. Purchase não criado/atualizado |
| E2E-004 | Usar créditos adquiridos | 1. Compra bem-sucedida (+20 credits)<br>2. Acesso /generate<br>3. Gerar página de colorir<br>4. Verificar créditos após uso | Geração funciona + créditos debitados | 1. user.credits = 20 após compra<br>2. Formulário /generate aceita submit<br>3. Imagem gerada<br>4. user.credits = 19 após geração |
| E2E-005 | Webhook com delay (simular retry) | 1. Compra iniciada<br>2. Webhook não processa (simular)<br>3. Aguardar 30s<br>4. Webhook processado (Stripe retry) | Créditos adicionados mesmo com delay | 1. Purchase status = 'pending' inicialmente<br>2. Webhook eventualmente processa<br>3. Purchase status = 'completed'<br>4. Créditos no user |
| E2E-006 | Testar Stripe test mode | Usar chaves sk_test_ e pk_test_ | Operações funcionam sem cobrar de verdade | 1. ENV vars com chaves test<br>2. Checkout session criada (test)<br>3. Nenhuma transação real |
| E2E-007 | Testar Stripe CLI webhook forwarding | `stripe listen --forward-to localhost:3000/api/webhooks/stripe` | Webhooks em desenvolvimento processados | 1. Stripe CLI conectado<br>2. Eventos encaminhados<br>3. Handler processa corretamente<br>4. DB atualizado localmente |
| E2E-008 | Multiple purchases by same user | 1. Compra 1 (20 credits)<br>2. Compra 2 (20 credits)<br>3. Compra 3 (20 credits) | Acumular créditos sem conflito | 1. Cada purchase cria nova sessão<br>2. user.credits = 60<br>3. Nenhuma duplicação de créditos |
| E2E-009 | Reembolso via Stripe Dashboard | 1. Compra bem-sucedida<br>2. Admin faz refund no Stripe Dashboard<br>3. Webhook charge.refunded chega<br>4. Verificar créditos do usuário | Créditos removidos após refund | 1. Initial: user.credits = 20<br>2. Após refund: user.credits = 0<br>3. Purchase.status = 'refunded'<br>4. Auditoria registrada |
| E2E-010 | Security: SQL injection em userId | POST /api/checkout<br>userId = "'; DROP TABLE users; --" | Request rejeitado ou sanitizado | 1. Parametrized queries usadas<br>2. Nenhuma SQL injection possível<br>3. Erro apropriado retornado |

**Implementação de exemplo (Playwright):**
```typescript
// __tests__/e2e/stripe-purchase.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Stripe Purchase Flow', () => {
  test('should complete purchase successfully', async ({ page, context }) => {
    // 1. Navegar para pricing
    await page.goto('http://localhost:3000/pricing');
    expect(page.url()).toContain('/pricing');

    // 2. Click "Buy Credits"
    await page.click('button:has-text("Buy Credits")');

    // 3. Stripe Checkout abre
    const stripeFrame = page.frames().find(f => f.url().includes('stripe.com'));
    expect(stripeFrame).toBeTruthy();

    // 4. Preencher formulário (testando apenas, não real)
    // Note: Stripe Checkout é iframe - usar Stripe's test helpers

    // 5. Verificar redirecionamento
    // await page.waitForNavigation();
    // expect(page.url()).toContain('/generate?success=true');

    // 6. Verificar DB
    const response = await context.request.get('http://localhost:3000/api/user/credits');
    const data = await response.json();
    expect(data.credits).toBe(20);
  });

  test('should show error on invalid card', async ({ page }) => {
    await page.goto('http://localhost:3000/pricing');
    await page.click('button:has-text("Buy Credits")');

    const stripeFrame = page.frames().find(f => f.url().includes('stripe.com'));
    // Preencher com cartão rejeitado
    // Verificar mensagem de erro

    expect(page.url()).not.toContain('success=true');
  });

  test('should cancel purchase', async ({ page }) => {
    await page.goto('http://localhost:3000/pricing');
    await page.click('button:has-text("Buy Credits")');

    // Clickar "Cancel" no Stripe Checkout
    await page.click('button:has-text("Cancel")');

    expect(page.url()).toContain('/pricing?canceled=true');
  });
});
```

---

### Execução dos Testes

**Unit Tests:**
```bash
bun test:unit -- utils/creditCalculator.test.ts
bun test:unit -- utils/webhookValidator.test.ts
```

**Integration Tests:**
```bash
bun test:integration -- __tests__/integration/api/checkout.test.ts
bun test:integration -- __tests__/integration/webhooks/stripe.test.ts
bun test:integration -- __tests__/integration/webhooks/refund.test.ts
```

**E2E Tests:**
```bash
bun test:e2e -- __tests__/e2e/stripe-purchase.spec.ts
```

**Todos os testes:**
```bash
bun test
```

### Coverage Mínimo Esperado

- **Unit Tests:** 95%+ de cobertura
- **Integration Tests:** 90%+ de cobertura
- **E2E Tests:** Todos os fluxos críticos cobertos
- **Overall:** 85%+ de cobertura de código
