# Especificação: Database

## Visão Geral

PostgreSQL hospedado no Neon para persistência de dados.

## Provedor

**Neon** - PostgreSQL serverless com branching e auto-scaling.

## Schema

### Tabela: users

Armazena usuários sincronizados do Clerk.

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clerk_id TEXT UNIQUE NOT NULL,
  email TEXT NOT NULL,
  credits INTEGER DEFAULT 0 CHECK (credits >= 0),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_clerk_id ON users(clerk_id);
```

### Tabela: images

Armazena imagens geradas pelos usuários.

```sql
CREATE TABLE images (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  prompt TEXT NOT NULL,
  image_url TEXT NOT NULL,
  is_public BOOLEAN DEFAULT false,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_images_user_id ON images(user_id);
CREATE INDEX idx_images_is_public ON images(is_public) WHERE is_public = true;
CREATE INDEX idx_images_created_at ON images(created_at DESC);
```

### Tabela: purchases

Armazena histórico de compras de créditos.

```sql
CREATE TABLE purchases (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  stripe_session_id TEXT UNIQUE,
  stripe_payment_intent_id TEXT,
  amount_cents INTEGER NOT NULL CHECK (amount_cents > 0),
  credits INTEGER NOT NULL CHECK (credits > 0),
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'completed', 'failed', 'refunded')),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  completed_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_purchases_user_id ON purchases(user_id);
CREATE INDEX idx_purchases_stripe_session ON purchases(stripe_session_id);
CREATE INDEX idx_purchases_status ON purchases(status);
```

## Relacionamentos

```
users (1) ──── (N) images
users (1) ──── (N) purchases
```

## Queries Principais

### Buscar usuário por clerk_id
```sql
SELECT * FROM users WHERE clerk_id = $1;
```

### Listar imagens do usuário
```sql
SELECT * FROM images
WHERE user_id = $1
ORDER BY created_at DESC;
```

### Listar galeria pública
```sql
SELECT i.*, u.email
FROM images i
JOIN users u ON i.user_id = u.id
WHERE i.is_public = true
ORDER BY i.created_at DESC
LIMIT $1 OFFSET $2;
```

### Debitar crédito (transação)
```sql
UPDATE users
SET credits = credits - 1, updated_at = NOW()
WHERE id = $1 AND credits > 0
RETURNING credits;
```

### Creditar após compra (transação)
```sql
BEGIN;
UPDATE purchases SET status = 'completed', completed_at = NOW() WHERE stripe_session_id = $1;
UPDATE users SET credits = credits + $2, updated_at = NOW() WHERE id = $3;
COMMIT;
```

## Migrations

Usar Drizzle ORM para migrations:

```
paintgpt/
├── drizzle/
│   ├── migrations/
│   │   └── 0000_initial.sql
│   └── schema.ts
└── drizzle.config.ts
```

## ORM

**Drizzle ORM** - Type-safe, lightweight, SQL-like syntax.

```typescript
// drizzle/schema.ts
import { pgTable, uuid, text, integer, boolean, timestamp } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  clerkId: text('clerk_id').unique().notNull(),
  email: text('email').notNull(),
  credits: integer('credits').default(0).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
});

export const images = pgTable('images', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  prompt: text('prompt').notNull(),
  imageUrl: text('image_url').notNull(),
  isPublic: boolean('is_public').default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
});

export const purchases = pgTable('purchases', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  stripeSessionId: text('stripe_session_id').unique(),
  stripePaymentIntentId: text('stripe_payment_intent_id'),
  amountCents: integer('amount_cents').notNull(),
  credits: integer('credits').notNull(),
  status: text('status').default('pending'),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
  completedAt: timestamp('completed_at', { withTimezone: true }),
});
```

## Variáveis de Ambiente

```env
DATABASE_URL=postgresql://user:pass@host/db?sslmode=require
```

## Considerações

- Usar connection pooling do Neon
- Índices em campos frequentemente filtrados
- Soft delete não necessário para MVP (CASCADE suficiente)
- Backup automático via Neon
