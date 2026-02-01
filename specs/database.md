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

## Testes TDD

### Schema Validation

| Test Case | Description | Expected Result | Status |
|-----------|-------------|-----------------|--------|
| Users table exists | Verify `users` table is created with all required columns | Table exists with columns: id, clerk_id, email, credits, created_at, updated_at | PENDING |
| Users primary key | Verify `id` is UUID primary key | UUID auto-generates via gen_random_uuid() | PENDING |
| Users clerk_id unique | Verify `clerk_id` has unique constraint | Inserting duplicate clerk_id raises constraint violation | PENDING |
| Users credits check | Verify `credits` has CHECK constraint >= 0 | Inserting negative credits raises constraint violation | PENDING |
| Images table exists | Verify `images` table is created with all required columns | Table exists with columns: id, user_id, prompt, image_url, is_public, created_at | PENDING |
| Images foreign key | Verify `user_id` references users(id) with ON DELETE CASCADE | Deleting user cascades to images | PENDING |
| Images is_public default | Verify `is_public` defaults to false | New images have is_public = false | PENDING |
| Purchases table exists | Verify `purchases` table is created with all required columns | Table exists with columns: id, user_id, stripe_session_id, stripe_payment_intent_id, amount_cents, credits, status, created_at, completed_at | PENDING |
| Purchases status check | Verify `status` has CHECK constraint with allowed values | Only 'pending', 'completed', 'failed', 'refunded' are accepted | PENDING |
| Purchases amount_cents check | Verify `amount_cents` has CHECK constraint > 0 | Inserting zero or negative amount raises constraint violation | PENDING |

### Indexes

| Test Case | Description | Expected Result | Status |
|-----------|-------------|-----------------|--------|
| idx_users_clerk_id | Verify index on users(clerk_id) exists | Query by clerk_id uses index | PENDING |
| idx_images_user_id | Verify index on images(user_id) exists | Query by user_id uses index | PENDING |
| idx_images_is_public | Verify partial index on images(is_public) WHERE is_public = true | Public gallery queries use index | PENDING |
| idx_images_created_at | Verify index on images(created_at DESC) exists | Sorting by created_at uses index | PENDING |
| idx_purchases_user_id | Verify index on purchases(user_id) exists | Query by user_id uses index | PENDING |
| idx_purchases_stripe_session | Verify index on purchases(stripe_session_id) exists | Lookup by stripe_session_id uses index | PENDING |
| idx_purchases_status | Verify index on purchases(status) exists | Filtering by status uses index | PENDING |

### CRUD Operations - Users

| Test Case | Description | Expected Result | Status |
|-----------|-------------|-----------------|--------|
| Create user | Insert new user with clerk_id, email | User created with id, credits=0, timestamps set | PENDING |
| Create user defaults | Insert user without specifying defaults | credits defaults to 0, created_at and updated_at auto-set | PENDING |
| Read user by id | Query user by id | Returns user object with all fields | PENDING |
| Read user by clerk_id | Query user by clerk_id | Returns correct user | PENDING |
| Update user credits | Increment credits by N | credits field updates, updated_at changes | PENDING |
| Update user email | Change email | email field updates, updated_at changes | PENDING |
| Delete user | Delete user by id | User and cascading images/purchases are deleted | PENDING |
| List all users | Query all users | Returns paginated or complete list | PENDING |

### CRUD Operations - Images

| Test Case | Description | Expected Result | Status |
|-----------|-------------|-----------------|--------|
| Create image | Insert new image with user_id, prompt, image_url | Image created with is_public=false, created_at set | PENDING |
| Create image required fields | Insert without prompt or image_url | Constraint violation (NOT NULL) | PENDING |
| Read image by id | Query image by id | Returns image object | PENDING |
| Read user images | Query images WHERE user_id = $1 | Returns all images for user, ordered by created_at DESC | PENDING |
| Read public images | Query images WHERE is_public = true | Returns only public images | PENDING |
| Update image is_public | Change is_public to true | Image marked public, no timestamp change | PENDING |
| Update image metadata | Update prompt or other fields | Fields update correctly | PENDING |
| Delete image | Delete by id | Image removed, user and purchases unaffected | PENDING |
| Cascade delete on user | Delete user | All user's images deleted | PENDING |

### CRUD Operations - Purchases

| Test Case | Description | Expected Result | Status |
|-----------|-------------|-----------------|--------|
| Create purchase | Insert new purchase with user_id, amount_cents, credits | Purchase created with status='pending', created_at set, completed_at=null | PENDING |
| Create purchase stripe_session_id | Insert with stripe_session_id | Unique constraint enforced | PENDING |
| Create purchase invalid amount | Insert with amount_cents <= 0 | Constraint violation (CHECK amount_cents > 0) | PENDING |
| Create purchase invalid credits | Insert with credits <= 0 | Constraint violation (CHECK credits > 0) | PENDING |
| Read purchase by id | Query purchase by id | Returns purchase object | PENDING |
| Read purchases by user | Query purchases WHERE user_id = $1 | Returns all purchases for user | PENDING |
| Read purchase by stripe_session | Query by stripe_session_id | Returns correct purchase | PENDING |
| Update purchase status | Change status to 'completed' | status updates, completed_at sets to NOW() | PENDING |
| Update purchase invalid status | Try to set status to invalid value | Constraint violation (CHECK status IN (...)) | PENDING |
| Delete purchase | Delete by id | Purchase removed, user credits unaffected | PENDING |
| Cascade delete on user | Delete user | All user's purchases deleted | PENDING |

### Transactions

| Test Case | Description | Expected Result | Status |
|-----------|-------------|-----------------|--------|
| Debit credit | UPDATE users SET credits = credits - 1 WHERE id = $1 AND credits > 0 | Credits decremented, updated_at changes, transaction atomic | PENDING |
| Debit insufficient credits | Try to debit when credits = 0 | Update returns 0 rows, credits unchanged | PENDING |
| Credit after purchase | BEGIN; UPDATE purchase status='completed'; UPDATE user credits += N; COMMIT; | Both updates succeed atomically or both rollback | PENDING |
| Concurrent debit | Two simultaneous debit operations on same user | Only one succeeds, credits deducted once | PENDING |
| Purchase refund | Update purchase status='refunded' and remove credits from user | Credits decremented, status changes atomically | PENDING |

### Data Integrity

| Test Case | Description | Expected Result | Status |
|-----------|-------------|-----------------|--------|
| Referential integrity users | Insert image with non-existent user_id | Foreign key constraint violation | PENDING |
| Referential integrity purchases | Insert purchase with non-existent user_id | Foreign key constraint violation | PENDING |
| Cascade delete images | Delete user, check images table | All user's images deleted | PENDING |
| Cascade delete purchases | Delete user, check purchases table | All user's purchases deleted | PENDING |
| Timestamp accuracy | Insert record, check created_at | created_at is within 1 second of NOW() | PENDING |
| Timestamp update | Update record, check updated_at | updated_at changes to current time | PENDING |
| UUID uniqueness | Insert 100 users | All IDs are unique | PENDING |

### Query Performance

| Test Case | Description | Expected Result | Status |
|-----------|-------------|-----------------|--------|
| Public gallery pagination | Query top 20 public images ordered by created_at DESC LIMIT 20 OFFSET 0 | Query returns in < 100ms, uses index | PENDING |
| User images sorted | Query user's images ordered by created_at DESC | Query returns in < 50ms, uses index | PENDING |
| Credits lookup | Query user by clerk_id for credit check | Query returns in < 20ms, uses index | PENDING |
| Purchase lookup by stripe | Query purchase by stripe_session_id | Query returns in < 20ms, uses index | PENDING |
| Pending purchases | Query all purchases WHERE status = 'pending' | Query returns in < 100ms, uses index | PENDING |

### Migration Testing

| Test Case | Description | Expected Result | Status |
|-----------|-------------|-----------------|--------|
| Initial migration | Run 0000_initial.sql | All 3 tables created with all indexes | PENDING |
| Migration idempotency | Run migration twice | Second run fails gracefully (IF NOT EXISTS) or succeeds without error | PENDING |
| Rollback migration | Rollback to pre-migration state | All tables and indexes removed | PENDING |
| Migration on fresh DB | Apply migration to empty database | Clean schema created with no errors | PENDING |
