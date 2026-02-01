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
