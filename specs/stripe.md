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
