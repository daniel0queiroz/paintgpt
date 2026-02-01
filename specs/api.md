# Especificação: API Routes

## Visão Geral

API RESTful usando Next.js Route Handlers (App Router).

## Base URL

```
Development: http://localhost:3000/api
Production:  https://paintgpt.com/api
```

## Autenticação

Rotas protegidas usam Clerk `auth()` para validar sessão.

```typescript
import { auth } from '@clerk/nextjs/server';

export async function GET() {
  const { userId } = await auth();

  if (!userId) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // ...
}
```

---

## Endpoints

### POST /api/generate

Gera nova imagem de colorir.

| Campo | Valor |
|-------|-------|
| Auth | Requerido |
| Custo | 1 crédito |

**Request:**
```typescript
{
  prompt: string  // 3-500 caracteres
}
```

**Response 200:**
```typescript
{
  success: true,
  image: {
    id: string,
    prompt: string,
    imageUrl: string,
    createdAt: string
  },
  creditsRemaining: number
}
```

**Response 400:**
```typescript
{
  success: false,
  error: {
    code: 'INVALID_PROMPT',
    message: 'Prompt must be between 3 and 500 characters'
  }
}
```

**Response 402:**
```typescript
{
  success: false,
  error: {
    code: 'INSUFFICIENT_CREDITS',
    message: 'You need credits to generate images'
  }
}
```

---

### GET /api/images

Lista imagens do usuário autenticado.

| Campo | Valor |
|-------|-------|
| Auth | Requerido |

**Query Parameters:**
| Param | Tipo | Default | Descrição |
|-------|------|---------|-----------|
| visibility | `'all' \| 'public' \| 'private'` | `'all'` | Filtro de visibilidade |
| cursor | string | - | ID para paginação |
| limit | number | 20 | Máx 50 |

**Response 200:**
```typescript
{
  images: Array<{
    id: string,
    prompt: string,
    imageUrl: string,
    isPublic: boolean,
    createdAt: string
  }>,
  total: number,
  nextCursor: string | null
}
```

---

### PATCH /api/images/[id]

Atualiza visibilidade de uma imagem.

| Campo | Valor |
|-------|-------|
| Auth | Requerido |
| Ownership | Deve ser dono da imagem |

**Request:**
```typescript
{
  isPublic: boolean
}
```

**Response 200:**
```typescript
{
  success: true,
  image: {
    id: string,
    isPublic: boolean
  }
}
```

**Response 404:**
```typescript
{
  success: false,
  error: {
    code: 'NOT_FOUND',
    message: 'Image not found'
  }
}
```

---

### DELETE /api/images/[id]

Deleta uma imagem permanentemente.

| Campo | Valor |
|-------|-------|
| Auth | Requerido |
| Ownership | Deve ser dono da imagem |

**Response 200:**
```typescript
{
  success: true
}
```

**Response 404:**
```typescript
{
  success: false,
  error: {
    code: 'NOT_FOUND',
    message: 'Image not found'
  }
}
```

---

### GET /api/gallery

Lista imagens públicas da galeria.

| Campo | Valor |
|-------|-------|
| Auth | Não requerido |

**Query Parameters:**
| Param | Tipo | Default | Descrição |
|-------|------|---------|-----------|
| cursor | string | - | ID para paginação |
| limit | number | 20 | Máx 50 |

**Response 200:**
```typescript
{
  images: Array<{
    id: string,
    prompt: string,
    imageUrl: string,
    createdAt: string
  }>,
  nextCursor: string | null,
  hasMore: boolean
}
```

---

### GET /api/credits

Retorna saldo de créditos do usuário.

| Campo | Valor |
|-------|-------|
| Auth | Requerido |

**Response 200:**
```typescript
{
  credits: number,
  canGenerate: boolean
}
```

---

### POST /api/checkout

Cria sessão de checkout no Stripe.

| Campo | Valor |
|-------|-------|
| Auth | Requerido |

**Request:**
```typescript
{} // Sem body - pacote único no MVP
```

**Response 200:**
```typescript
{
  url: string  // Stripe Checkout URL
}
```

---

### GET /api/download/[id]

Gera e retorna PDF da imagem.

| Campo | Valor |
|-------|-------|
| Auth | Requerido para imagens privadas |
| Content-Type | application/pdf |

**Response 200:**
- Headers: `Content-Disposition: attachment; filename="coloring-page-{id}.pdf"`
- Body: PDF binary

**Response 403:**
```typescript
{
  success: false,
  error: {
    code: 'FORBIDDEN',
    message: 'You do not have access to this image'
  }
}
```

**Response 404:**
```typescript
{
  success: false,
  error: {
    code: 'NOT_FOUND',
    message: 'Image not found'
  }
}
```

---

### POST /api/webhooks/clerk

Webhook para eventos do Clerk.

| Campo | Valor |
|-------|-------|
| Auth | Verificação via Svix |

**Eventos Tratados:**
- `user.created` → Cria registro em `users`
- `user.deleted` → Remove dados do usuário
- `user.updated` → Atualiza email

---

### POST /api/webhooks/stripe

Webhook para eventos do Stripe.

| Campo | Valor |
|-------|-------|
| Auth | Verificação via assinatura Stripe |

**Eventos Tratados:**
- `checkout.session.completed` → Adiciona créditos
- `checkout.session.expired` → Marca purchase como failed
- `charge.refunded` → Remove créditos (se aplicável)

---

## Estrutura de Arquivos

```
app/api/
├── generate/
│   └── route.ts           # POST
├── images/
│   ├── route.ts           # GET
│   └── [id]/
│       └── route.ts       # PATCH, DELETE
├── gallery/
│   └── route.ts           # GET
├── credits/
│   └── route.ts           # GET
├── checkout/
│   └── route.ts           # POST
├── download/
│   └── [id]/
│       └── route.ts       # GET
└── webhooks/
    ├── clerk/
    │   └── route.ts       # POST
    └── stripe/
        └── route.ts       # POST
```

---

## Padrões de Resposta

### Sucesso
```typescript
{
  success: true,
  data?: any,
  // ou campos específicos no root
}
```

### Erro
```typescript
{
  success: false,
  error: {
    code: string,      // Código único do erro
    message: string    // Mensagem legível
  }
}
```

### Códigos de Erro

| Código | HTTP Status | Descrição |
|--------|-------------|-----------|
| `UNAUTHORIZED` | 401 | Não autenticado |
| `FORBIDDEN` | 403 | Sem permissão |
| `NOT_FOUND` | 404 | Recurso não encontrado |
| `INVALID_PROMPT` | 400 | Prompt inválido |
| `INSUFFICIENT_CREDITS` | 402 | Créditos insuficientes |
| `GENERATION_FAILED` | 502 | Erro na API de geração |
| `INTERNAL_ERROR` | 500 | Erro interno |

---

## Rate Limiting

| Endpoint | Limite |
|----------|--------|
| POST /api/generate | 10/min por usuário |
| GET /api/gallery | 60/min por IP |
| POST /api/checkout | 5/min por usuário |
| Webhooks | Sem limite |

### Implementação

```typescript
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '1 m'),
});

// No handler
const { success } = await ratelimit.limit(userId);
if (!success) {
  return Response.json(
    { error: { code: 'RATE_LIMITED', message: 'Too many requests' } },
    { status: 429 }
  );
}
```

**Nota MVP:** Rate limiting pode ser simplificado ou omitido inicialmente.

---

## Validação

Usar Zod para validação de input:

```typescript
import { z } from 'zod';

const generateSchema = z.object({
  prompt: z.string().min(3).max(500),
});

export async function POST(request: Request) {
  const body = await request.json();

  const result = generateSchema.safeParse(body);
  if (!result.success) {
    return Response.json({
      success: false,
      error: {
        code: 'INVALID_INPUT',
        message: result.error.issues[0].message
      }
    }, { status: 400 });
  }

  // ...
}
```

---

## CORS

Não necessário para MVP (same-origin). Se precisar:

```typescript
// middleware.ts ou em cada route
const headers = {
  'Access-Control-Allow-Origin': process.env.NEXT_PUBLIC_APP_URL,
  'Access-Control-Allow-Methods': 'GET, POST, PATCH, DELETE',
  'Access-Control-Allow-Headers': 'Content-Type, Authorization',
};
```

---

## Testes TDD

### POST /api/generate

| Endpoint | Teste | Tipo | Assertions |
|----------|-------|------|-----------|
| POST /api/generate | Gera imagem com prompt válido | Success | `status === 200`, `response.success === true`, `response.image.id` existe, `response.image.prompt === prompt`, `response.creditsRemaining >= 0` |
| POST /api/generate | Prompt muito curto (< 3 caracteres) | Validation | `status === 400`, `response.error.code === 'INVALID_PROMPT'`, `response.success === false` |
| POST /api/generate | Prompt muito longo (> 500 caracteres) | Validation | `status === 400`, `response.error.code === 'INVALID_PROMPT'`, `response.success === false` |
| POST /api/generate | Body vazio | Validation | `status === 400`, `response.error.code === 'INVALID_INPUT'`, `response.success === false` |
| POST /api/generate | Sem autenticação | Auth | `status === 401`, `response.error.code === 'UNAUTHORIZED'`, `response.success === false` |
| POST /api/generate | Sem créditos suficientes | Business Logic | `status === 402`, `response.error.code === 'INSUFFICIENT_CREDITS'`, `response.success === false` |
| POST /api/generate | Rate limit (> 10/min) | Rate Limiting | `status === 429`, `response.error.code === 'RATE_LIMITED'`, `response.success === false` |
| POST /api/generate | Falha na API de geração | Error Handling | `status === 502`, `response.error.code === 'GENERATION_FAILED'`, `response.success === false` |
| POST /api/generate | Deduz 1 crédito da conta | Business Logic | `response.creditsRemaining === creditsBefore - 1` |

### GET /api/images

| Endpoint | Teste | Tipo | Assertions |
|----------|-------|------|-----------|
| GET /api/images | Lista imagens sem filtro | Success | `status === 200`, `response.images` é array, `response.total >= 0`, `response.nextCursor` pode ser nulo |
| GET /api/images | Filtra imagens públicas | Success | `status === 200`, `response.images.every(img => img.isPublic === true)` |
| GET /api/images | Filtra imagens privadas | Success | `status === 200`, `response.images.every(img => img.isPublic === false)` |
| GET /api/images | Paginação com cursor válido | Success | `status === 200`, `response.images.length <= limit`, `response.nextCursor` existe para próxima página |
| GET /api/images | Limit maior que 50 | Validation | `status === 400`, `response.error.code === 'INVALID_INPUT'` ou `limit === 50` |
| GET /api/images | Limite 0 ou negativo | Validation | `status === 400`, `response.error.code === 'INVALID_INPUT'` |
| GET /api/images | Sem autenticação | Auth | `status === 401`, `response.error.code === 'UNAUTHORIZED'` |
| GET /api/images | Visibility inválido | Validation | `status === 400`, `response.error.code === 'INVALID_INPUT'` |
| GET /api/images | Rate limit (sem limite específico) | Success | `status === 200` (sem rate limit aplicado) |

### PATCH /api/images/[id]

| Endpoint | Teste | Tipo | Assertions |
|----------|-------|------|-----------|
| PATCH /api/images/[id] | Muda visibilidade para pública | Success | `status === 200`, `response.success === true`, `response.image.isPublic === true` |
| PATCH /api/images/[id] | Muda visibilidade para privada | Success | `status === 200`, `response.success === true`, `response.image.isPublic === false` |
| PATCH /api/images/[id] | Body vazio | Validation | `status === 400`, `response.error.code === 'INVALID_INPUT'` |
| PATCH /api/images/[id] | isPublic não é booleano | Validation | `status === 400`, `response.error.code === 'INVALID_INPUT'` |
| PATCH /api/images/[id] | Sem autenticação | Auth | `status === 401`, `response.error.code === 'UNAUTHORIZED'` |
| PATCH /api/images/[id] | ID não pertence ao usuário | Auth | `status === 403`, `response.error.code === 'FORBIDDEN'` |
| PATCH /api/images/[id] | ID não existe | Not Found | `status === 404`, `response.error.code === 'NOT_FOUND'` |
| PATCH /api/images/[id] | ID formato inválido | Validation | `status === 400`, `response.error.code === 'INVALID_INPUT'` |

### DELETE /api/images/[id]

| Endpoint | Teste | Tipo | Assertions |
|----------|-------|------|-----------|
| DELETE /api/images/[id] | Deleta imagem com sucesso | Success | `status === 200`, `response.success === true` |
| DELETE /api/images/[id] | Imagem removida da base | Business Logic | Query `SELECT * FROM images WHERE id = ?` retorna vazio |
| DELETE /api/images/[id] | Sem autenticação | Auth | `status === 401`, `response.error.code === 'UNAUTHORIZED'` |
| DELETE /api/images/[id] | ID não pertence ao usuário | Auth | `status === 403`, `response.error.code === 'FORBIDDEN'` |
| DELETE /api/images/[id] | ID não existe | Not Found | `status === 404`, `response.error.code === 'NOT_FOUND'` |
| DELETE /api/images/[id] | ID formato inválido | Validation | `status === 400`, `response.error.code === 'INVALID_INPUT'` |
| DELETE /api/images/[id] | Deleta duas vezes mesma imagem | Idempotency | Primeira: `status === 200`, Segunda: `status === 404` |

### GET /api/gallery

| Endpoint | Teste | Tipo | Assertions |
|----------|-------|------|-----------|
| GET /api/gallery | Lista imagens públicas | Success | `status === 200`, `response.images` é array, `response.hasMore` é booleano |
| GET /api/gallery | Todas as imagens são públicas | Success | `response.images.every(img => img.isPublic === true)` |
| GET /api/gallery | Paginação com cursor válido | Success | `status === 200`, `response.images.length <= limit`, `response.nextCursor` válido |
| GET /api/gallery | Limit maior que 50 | Validation | `status === 400` ou `limit === 50` |
| GET /api/gallery | Limit 0 ou negativo | Validation | `status === 400`, `response.error.code === 'INVALID_INPUT'` |
| GET /api/gallery | Sem autenticação | Success | `status === 200` (públicopermitido) |
| GET /api/gallery | Com token inválido | Success | `status === 200` (não afeta acesso público) |
| GET /api/gallery | Rate limit por IP (> 60/min) | Rate Limiting | `status === 429`, `response.error.code === 'RATE_LIMITED'` |
| GET /api/gallery | Cursor inválido | Validation | `status === 400`, `response.error.code === 'INVALID_INPUT'` |

### GET /api/credits

| Endpoint | Teste | Tipo | Assertions |
|----------|-------|------|-----------|
| GET /api/credits | Retorna créditos válidos | Success | `status === 200`, `response.credits >= 0`, `response.canGenerate` é booleano |
| GET /api/credits | canGenerate true se credits >= 1 | Business Logic | `credits >= 1 && canGenerate === true` |
| GET /api/credits | canGenerate false se credits < 1 | Business Logic | `credits === 0 && canGenerate === false` |
| GET /api/credits | Sem autenticação | Auth | `status === 401`, `response.error.code === 'UNAUTHORIZED'` |
| GET /api/credits | Token expirado | Auth | `status === 401`, `response.error.code === 'UNAUTHORIZED'` |
| GET /api/credits | Novo usuário tem 0 créditos | Business Logic | `response.credits === 0` |

### POST /api/checkout

| Endpoint | Teste | Tipo | Assertions |
|----------|-------|------|-----------|
| POST /api/checkout | Cria sessão Stripe com sucesso | Success | `status === 200`, `response.url` contém `checkout.stripe.com` |
| POST /api/checkout | URL retornada é válida | Success | `URL.parse(response.url)` não lança erro |
| POST /api/checkout | Sem autenticação | Auth | `status === 401`, `response.error.code === 'UNAUTHORIZED'` |
| POST /api/checkout | Body com campos extras | Validation | `status === 200` (ignorados) ou `status === 400` |
| POST /api/checkout | Rate limit (> 5/min) | Rate Limiting | `status === 429`, `response.error.code === 'RATE_LIMITED'` |
| POST /api/checkout | Falha na API do Stripe | Error Handling | `status === 502`, `response.error.code === 'STRIPE_ERROR'` |
| POST /api/checkout | Cria novo Stripe Customer se não existe | Business Logic | Stripe Customer criado na API |

### GET /api/download/[id]

| Endpoint | Teste | Tipo | Assertions |
|----------|-------|------|-----------|
| GET /api/download/[id] | Download PDF de imagem pública | Success | `status === 200`, `content-type === 'application/pdf'`, arquivo PDF válido |
| GET /api/download/[id] | Download PDF de imagem privada do dono | Success | `status === 200`, `content-type === 'application/pdf'`, arquivo PDF válido |
| GET /api/download/[id] | Header Content-Disposition correto | Success | `content-disposition` contém `attachment; filename="coloring-page-{id}.pdf"` |
| GET /api/download/[id] | Imagem privada sem autenticação | Auth | `status === 403`, `response.error.code === 'FORBIDDEN'` |
| GET /api/download/[id] | Imagem privada de outro usuário | Auth | `status === 403`, `response.error.code === 'FORBIDDEN'` |
| GET /api/download/[id] | ID não existe | Not Found | `status === 404`, `response.error.code === 'NOT_FOUND'` |
| GET /api/download/[id] | ID formato inválido | Validation | `status === 400`, `response.error.code === 'INVALID_INPUT'` |
| GET /api/download/[id] | Gera PDF corretamente | Business Logic | PDF contém imagem e informações corretas |

### POST /api/webhooks/clerk

| Endpoint | Teste | Tipo | Assertions |
|----------|-------|------|-----------|
| POST /api/webhooks/clerk | Evento user.created | Success | `status === 200`, registro criado em `users` |
| POST /api/webhooks/clerk | Evento user.deleted | Success | `status === 200`, dados do usuário removidos |
| POST /api/webhooks/clerk | Evento user.updated | Success | `status === 200`, email atualizado |
| POST /api/webhooks/clerk | Signature inválida | Auth | `status === 401`, webhook rejeitado |
| POST /api/webhooks/clerk | Body vazio | Validation | `status === 400`, webhook rejeitado |
| POST /api/webhooks/clerk | Tipo de evento desconhecido | Success | `status === 200` (ignorado) |
| POST /api/webhooks/clerk | Sem header Svix-Signature | Auth | `status === 401`, webhook rejeitado |
| POST /api/webhooks/clerk | Usuário criado com créditos iniciais | Business Logic | `credits === 0` (ou valor padrão) |

### POST /api/webhooks/stripe

| Endpoint | Teste | Tipo | Assertions |
|----------|-------|------|-----------|
| POST /api/webhooks/stripe | Evento checkout.session.completed | Success | `status === 200`, créditos adicionados à conta |
| POST /api/webhooks/stripe | Evento checkout.session.expired | Success | `status === 200`, purchase marcado como failed |
| POST /api/webhooks/stripe | Evento charge.refunded | Success | `status === 200`, créditos removidos se aplicável |
| POST /api/webhooks/stripe | Signature inválida | Auth | `status === 401`, webhook rejeitado |
| POST /api/webhooks/stripe | Body vazio | Validation | `status === 400`, webhook rejeitado |
| POST /api/webhooks/stripe | Tipo de evento desconhecido | Success | `status === 200` (ignorado) |
| POST /api/webhooks/stripe | Sem header Stripe-Signature | Auth | `status === 401`, webhook rejeitado |
| POST /api/webhooks/stripe | Processamento idempotente | Idempotency | Mesmo webhook processado 2x = resultado idêntico |

### Testes de Integração

| Caso | Teste | Tipo | Assertions |
|------|-------|------|-----------|
| Fluxo Completo | Usuário cria conta → Gera imagem → Compra créditos | Integration | `status === 200` em cada etapa, créditos atualizados corretamente |
| Fluxo Completo | Gera imagem → Torna pública → Aparece na galeria | Integration | Imagem visível em `GET /api/gallery` |
| Fluxo Completo | Compra créditos via webhook → Créditos refletidos | Integration | `GET /api/credits` retorna novo saldo |
| Fluxo Completo | Usuário deletado → Imagens removidas | Integration | `DELETE /api/images` return `404` para imagens do usuário |

### Testes de Performance

| Caso | Teste | Tipo | Assertions |
|------|-------|------|-----------|
| Performance | GET /api/images com 1000+ imagens | Performance | `response time < 500ms`, pagination funciona |
| Performance | GET /api/gallery com 10000+ imagens públicas | Performance | `response time < 1s`, pagination eficiente |
| Performance | POST /api/generate com muitos usuários | Load Testing | `response time < 2s` mesmo sob carga |

### Configuração de Testes

Use as seguintes bibliotecas para implementar os testes:

```typescript
// Tech stack para testes
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { http, HttpResponse, server } from 'msw';

// Exemplo de teste
describe('POST /api/generate', () => {
  it('should generate image with valid prompt', async () => {
    const response = await fetch('/api/generate', {
      method: 'POST',
      body: JSON.stringify({ prompt: 'A beautiful cat' }),
    });

    expect(response.status).toBe(200);
    const data = await response.json();
    expect(data.success).toBe(true);
    expect(data.image).toBeDefined();
    expect(data.image.id).toBeDefined();
  });

  it('should reject prompt < 3 characters', async () => {
    const response = await fetch('/api/generate', {
      method: 'POST',
      body: JSON.stringify({ prompt: 'ab' }),
    });

    expect(response.status).toBe(400);
    const data = await response.json();
    expect(data.error.code).toBe('INVALID_PROMPT');
  });
});
```
