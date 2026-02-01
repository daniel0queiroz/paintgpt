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

## Testes TDD

### Unit Tests

#### Validação de Sessão Clerk

| Teste | Arquivo | Descrição |
|-------|---------|-----------|
| Sessão ativa retorna userId | `__tests__/lib/clerk-validation.test.ts` | Validar que `auth()` retorna userId correto |
| Sessão inativa retorna null | `__tests__/lib/clerk-validation.test.ts` | Validar que `auth()` retorna null quando não autenticado |
| Clerk ID é extraído corretamente | `__tests__/lib/clerk-validation.test.ts` | Validar formato e estrutura do clerk_id |
| Email é validado | `__tests__/lib/clerk-validation.test.ts` | Validar que email é extraído corretamente |

#### Middleware de Proteção

| Teste | Arquivo | Descrição |
|-------|---------|-----------|
| Rotas públicas não bloqueadas | `__tests__/middleware.test.ts` | Verificar acesso a `/`, `/gallery`, `/pricing`, `/sign-in`, `/sign-up` |
| Rotas protegidas redirecionam | `__tests__/middleware.test.ts` | Verificar redirect para `/sign-in` quando acessa `/generate`, `/library`, `/account` |
| Usuário autenticado acessa rotas protegidas | `__tests__/middleware.test.ts` | Verificar que usuário logado consegue acessar rotas protegidas |
| Bearer token é validado em API | `__tests__/middleware.test.ts` | Verificar validação de token em rotas `/api/*` |

### Integration Tests

#### Webhook user.created

| Teste | Arquivo | Descrição |
|-------|---------|-----------|
| Webhook dispara e cria registro | `__tests__/api/webhooks/clerk.test.ts` | Validar que usuário é inserido em `users` quando webhook é recebido |
| Novo usuário tem 0 créditos | `__tests__/api/webhooks/clerk.test.ts` | Validar que `credits` inicia em 0 |
| Clerk_id e email são sincronizados | `__tests__/api/webhooks/clerk.test.ts` | Validar que `clerk_id` e `email` são armazenados corretamente |
| Webhook duplicado é idempotente | `__tests__/api/webhooks/clerk.test.ts` | Validar que webhook pode ser reenviado sem duplicar registros |
| Webhook com dados inválidos é rejeitado | `__tests__/api/webhooks/clerk.test.ts` | Validar que webhook sem email é rejeitado com status 400 |

#### Webhook user.deleted

| Teste | Arquivo | Descrição |
|-------|---------|-----------|
| Webhook dispara soft delete | `__tests__/api/webhooks/clerk.test.ts` | Validar que usuário é marcado como deletado |
| Imagens do usuário não aparecem mais | `__tests__/api/webhooks/clerk.test.ts` | Validar que `gallery` não retorna imagens de usuário deletado |
| Dados pessoais são anonimizados | `__tests__/api/webhooks/clerk.test.ts` | Validar que email é removido ou mascarado |
| Créditos restantes são zerados | `__tests__/api/webhooks/clerk.test.ts` | Validar que `credits` vai para 0 |

#### Sincronização de Email

| Teste | Arquivo | Descrição |
|-------|---------|-----------|
| Email é atualizado quando mudado | `__tests__/api/webhooks/clerk.test.ts` | Validar que webhook `user.updated` sincroniza novo email |
| Email antigo não é mais válido | `__tests__/api/webhooks/clerk.test.ts` | Validar que email antigo não pode ser usado para login |
| Email duplicado é rejeitado | `__tests__/api/webhooks/clerk.test.ts` | Validar constraint único de email |

### E2E Tests

#### Login com Email/Senha

| Teste | Arquivo | Descrição |
|-------|---------|-----------|
| Acesso a `/sign-in` funciona | `e2e/auth.spec.ts` | Validar que página de login carrega |
| Login com email válido funciona | `e2e/auth.spec.ts` | Validar fluxo completo de login com email/senha |
| Login com email inválido falha | `e2e/auth.spec.ts` | Validar mensagem de erro para email/senha incorretos |
| Session cookie é criado | `e2e/auth.spec.ts` | Validar que cookie de sessão é armazenado |
| Usuário pode fazer logout | `e2e/auth.spec.ts` | Validar fluxo de logout |

#### Login com Google OAuth

| Teste | Arquivo | Descrição |
|-------|---------|-----------|
| Botão Google OAuth visível | `e2e/auth.spec.ts` | Validar que botão "Sign in with Google" existe |
| Fluxo OAuth redireciona | `e2e/auth.spec.ts` | Validar que clique redireciona para consentimento Google |
| Novo usuário Google é criado | `e2e/auth.spec.ts` | Validar que novo usuário via Google é inserido no DB |
| Usuário Google existente pode fazer login | `e2e/auth.spec.ts` | Validar que usuário retornando via Google consegue fazer login |

#### Redirect de Rotas Protegidas

| Teste | Arquivo | Descrição |
|-------|---------|-----------|
| Acesso a `/generate` sem auth redireciona | `e2e/auth.spec.ts` | Validar redirect para `/sign-in` |
| Acesso a `/library` sem auth redireciona | `e2e/auth.spec.ts` | Validar redirect para `/sign-in` |
| Acesso a `/account` sem auth redireciona | `e2e/auth.spec.ts` | Validar redirect para `/sign-in` |
| Após login, redireciona para `/generate` | `e2e/auth.spec.ts` | Validar NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL |
| Após signup, redireciona para `/pricing` | `e2e/auth.spec.ts` | Validar NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL |

#### Logout

| Teste | Arquivo | Descrição |
|-------|---------|-----------|
| Botão logout visível quando logado | `e2e/auth.spec.ts` | Validar que UserButton aparece |
| Clique em logout funciona | `e2e/auth.spec.ts` | Validar que logout remove sessão |
| Após logout, acesso protegido redireciona | `e2e/auth.spec.ts` | Validar que rotas protegidas redirecionam novamente |
| Session cookie é removido | `e2e/auth.spec.ts` | Validar que cookie de sessão é deletado |
