# Especificação: Páginas

## Visão Geral

Estrutura de páginas do aplicativo usando Next.js App Router.

## Mapa de Rotas

| Rota | Página | Auth | Descrição |
|------|--------|------|-----------|
| `/` | Landing | Não | Homepage com proposta de valor |
| `/gallery` | Gallery | Não | Galeria pública de imagens |
| `/generate` | Generate | Sim | Interface de geração |
| `/library` | Library | Sim | Biblioteca pessoal |
| `/pricing` | Pricing | Não | Página de preços |
| `/account` | Account | Sim | Configurações da conta |
| `/sign-in` | Sign In | Não | Login (Clerk) |
| `/sign-up` | Sign Up | Não | Cadastro (Clerk) |

## Estrutura de Arquivos

```
app/
├── layout.tsx              # Root layout
├── page.tsx                # Landing (/)
├── gallery/
│   └── page.tsx            # Gallery
├── generate/
│   └── page.tsx            # Generate (protected)
├── library/
│   └── page.tsx            # Library (protected)
├── pricing/
│   └── page.tsx            # Pricing
├── account/
│   └── page.tsx            # Account (protected)
├── sign-in/
│   └── [[...sign-in]]/
│       └── page.tsx        # Clerk Sign In
├── sign-up/
│   └── [[...sign-up]]/
│       └── page.tsx        # Clerk Sign Up
└── api/
    ├── generate/
    │   └── route.ts
    ├── images/
    │   ├── route.ts
    │   └── [id]/
    │       └── route.ts
    ├── gallery/
    │   └── route.ts
    ├── checkout/
    │   └── route.ts
    ├── download/
    │   └── [id]/
    │       └── route.ts
    └── webhooks/
        ├── clerk/
        │   └── route.ts
        └── stripe/
            └── route.ts
```

---

## Página: Landing (`/`)

### Objetivo
Converter visitantes em usuários registrados.

### Seções

1. **Hero**
   - Headline: proposta de valor clara
   - Subheadline: benefício principal
   - CTA: "Create Your First Coloring Page"
   - Imagem/demo do produto

2. **How It Works**
   - 3 passos simples
   - Ícones ilustrativos

3. **Gallery Preview**
   - 6-8 imagens da galeria pública
   - Link para galeria completa

4. **Pricing Preview**
   - Preço único destacado
   - Benefícios inclusos
   - CTA para compra

5. **FAQ**
   - Perguntas frequentes
   - Accordion expandível

6. **Footer**
   - Links úteis
   - Copyright

### Metadata

```typescript
export const metadata: Metadata = {
  title: 'PaintGPT - AI Coloring Pages for Kids',
  description: 'Create unique coloring pages for children using AI. Simple, fun, and perfect for printing.',
  openGraph: {
    title: 'PaintGPT - AI Coloring Pages for Kids',
    description: 'Create unique coloring pages for children using AI.',
    images: ['/og-image.png'],
  },
};
```

---

## Página: Gallery (`/gallery`)

### Objetivo
Mostrar comunidade ativa e inspirar criações.

### Layout

```
┌─────────────────────────────────────┐
│ [Logo]              [Sign In] [CTA] │
├─────────────────────────────────────┤
│                                     │
│  Explore Coloring Pages             │
│  Discover creations from our        │
│  community                          │
│                                     │
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐          │
│  │   │ │   │ │   │ │   │          │
│  └───┘ └───┘ └───┘ └───┘          │
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐          │
│  │   │ │   │ │   │ │   │          │
│  └───┘ └───┘ └───┘ └───┘          │
│                                     │
│         [Load More]                 │
│                                     │
└─────────────────────────────────────┘
```

### Funcionalidades

- Grid infinito de imagens
- Click para expandir
- Download PDF (requer login)
- Filtros (Phase 2)

---

## Página: Generate (`/generate`)

### Objetivo
Interface principal de criação de imagens.

### Layout

```
┌─────────────────────────────────────┐
│ [Logo]    🎨 12 credits   [Avatar]  │
├─────────────────────────────────────┤
│                                     │
│  Create a Coloring Page             │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ Describe what you want...   │   │
│  │                             │   │
│  └─────────────────────────────┘   │
│                                     │
│  [Generate] (1 credit)              │
│                                     │
│  ─────────────────────────────────  │
│                                     │
│  ┌─────────────────────────────┐   │
│  │                             │   │
│  │     [Generated Image]       │   │
│  │                             │   │
│  └─────────────────────────────┘   │
│                                     │
│  [Download PDF] [Share] [New]       │
│                                     │
└─────────────────────────────────────┘
```

### Estados

1. **Inicial:** Campo de prompt vazio
2. **Gerando:** Loading spinner, prompt disabled
3. **Resultado:** Imagem gerada com ações
4. **Erro:** Mensagem de erro, retry
5. **Sem créditos:** Prompt para comprar

### Sem Créditos

```
┌─────────────────────────────────┐
│                                 │
│  You're out of credits!         │
│                                 │
│  Get 20 credits for $9 to       │
│  continue creating amazing      │
│  coloring pages.                │
│                                 │
│  [Buy Credits →]                │
│                                 │
└─────────────────────────────────┘
```

---

## Página: Library (`/library`)

### Objetivo
Gerenciar imagens pessoais.

### Layout

Ver spec `library.md` para detalhes completos.

### Estados

1. **Com imagens:** Grid com ações
2. **Vazio:** Empty state com CTA
3. **Loading:** Skeleton cards

---

## Página: Pricing (`/pricing`)

### Objetivo
Converter visitantes em compradores.

### Layout

```
┌─────────────────────────────────────┐
│                                     │
│      Simple, Fair Pricing           │
│                                     │
│  ┌─────────────────────────────┐   │
│  │                             │   │
│  │        20 Credits           │   │
│  │           $9                │   │
│  │                             │   │
│  │  ✓ 20 unique coloring pages │   │
│  │  ✓ High-quality PDF export  │   │
│  │  ✓ Share to community       │   │
│  │  ✓ Credits never expire     │   │
│  │                             │   │
│  │      [Get Started →]        │   │
│  │                             │   │
│  └─────────────────────────────┘   │
│                                     │
│  🔒 Secure payment by Stripe       │
│                                     │
│  ────────────────────────────────  │
│                                     │
│  FAQ                                │
│  [accordion items]                  │
│                                     │
└─────────────────────────────────────┘
```

### FAQ Sugerido

- How many images can I generate?
- What payment methods do you accept?
- Do credits expire?
- Can I get a refund?
- How do I download my images?

---

## Página: Account (`/account`)

### Objetivo
Configurações e informações da conta.

### Layout

```
┌─────────────────────────────────────┐
│                                     │
│  Account Settings                   │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ Profile                     │   │
│  │ email@example.com           │   │
│  │ [Manage Account →]          │   │
│  └─────────────────────────────┘   │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ Credits                     │   │
│  │ 12 credits remaining        │   │
│  │ [Buy More Credits →]        │   │
│  └─────────────────────────────┘   │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ Purchase History            │   │
│  │ - Mar 15: 20 credits - $9   │   │
│  │ - Feb 02: 20 credits - $9   │   │
│  └─────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘
```

### Funcionalidades

- Ver/editar perfil (via Clerk)
- Ver saldo de créditos
- Histórico de compras
- Link para comprar mais

---

## Componentes Compartilhados

### Header

```typescript
interface HeaderProps {
  showCredits?: boolean;
}

// Variantes:
// - Público: Logo + Sign In + CTA
// - Logado: Logo + Credits + Avatar
```

### Footer

```typescript
// Links: About, Privacy, Terms, Contact
// Copyright
// Social links (opcional)
```

### Layout Wrapper

```typescript
// Providers: Clerk, Theme, Toast
// Analytics: PostHog
// Global styles
```

---

## Responsividade

| Breakpoint | Largura | Layout |
|------------|---------|--------|
| Mobile | < 640px | Stack vertical |
| Tablet | 640-1024px | 2-3 colunas |
| Desktop | > 1024px | Layout completo |

## Performance

- Server Components por padrão
- Client Components apenas quando necessário
- Image optimization via Next/Image
- Lazy loading de componentes pesados
