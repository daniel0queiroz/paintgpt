# Especificação: Autenticação

## Visão Geral

Sistema de autenticação usando Clerk para gerenciar usuários, login e sessões.

## Provedor

**Clerk** - Autenticação gerenciada com suporte a múltiplos providers.

## Métodos de Login

| Método | Prioridade |
|--------|------------|
| Email/Password | MVP |
| Google OAuth | MVP |
| Apple OAuth | MVP |

## Integração com Database

### Sincronização de Usuários

Quando um usuário se registra no Clerk, um webhook deve criar o registro correspondente na tabela `users`:

```typescript
// Webhook: user.created
{
  clerk_id: string,
  email: string,
  credits: 0 // Novo usuário começa com 0 créditos
}
```

### Campos Sincronizados

| Clerk | Database |
|-------|----------|
| `user.id` | `clerk_id` |
| `user.emailAddresses[0]` | `email` |

## Proteção de Rotas

### Rotas Públicas
- `/` - Landing page
- `/gallery` - Galeria pública
- `/pricing` - Página de preços
- `/sign-in` - Login
- `/sign-up` - Cadastro

### Rotas Protegidas (Auth Required)
- `/generate` - Geração de imagens
- `/library` - Biblioteca pessoal
- `/account` - Configurações da conta
- `/api/generate` - API de geração
- `/api/images` - API de imagens do usuário
- `/api/checkout` - API de checkout

## Componentes Clerk

```typescript
// Providers necessários
<ClerkProvider>
  <SignInButton />
  <SignUpButton />
  <UserButton />
  <SignedIn />
  <SignedOut />
</ClerkProvider>
```

## Middleware

```typescript
// middleware.ts
import { clerkMiddleware } from '@clerk/nextjs/server'

export default clerkMiddleware()

export const config = {
  matcher: ['/((?!.*\\..*|_next).*)', '/', '/(api|trpc)(.*)'],
}
```

## Webhooks Clerk

| Evento | Ação |
|--------|------|
| `user.created` | Criar registro em `users` |
| `user.deleted` | Soft delete ou anonimizar dados |
| `user.updated` | Atualizar email se mudou |

## Variáveis de Ambiente

```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
CLERK_WEBHOOK_SECRET=
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/generate
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/pricing
```

## Fluxo de Novo Usuário

1. Usuário clica em "Create Your Own"
2. Redirecionado para `/sign-up`
3. Completa cadastro via Clerk
4. Webhook `user.created` dispara
5. Registro criado em `users` com 0 créditos
6. Redirecionado para `/pricing`
7. Após compra, pode gerar imagens

## Considerações de Segurança

- Validar `clerk_id` em todas as requisições autenticadas
- Nunca expor `clerk_id` no frontend
- Usar `auth()` do Clerk para validação server-side
- Rate limiting por usuário em rotas sensíveis
