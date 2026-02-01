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

---

## Testes TDD

### Component Tests

#### Landing Page (`/`)

| Test Case | Description | Expected Result | Priority |
|-----------|-------------|-----------------|----------|
| Renders hero section | Hero section with headline and CTA displays | CTA button visible with "Create Your First Coloring Page" text | P0 |
| Hero CTA navigates | Click CTA button on hero | Redirects to `/sign-up` | P0 |
| Renders "How It Works" section | "How It Works" section with 3 steps displays | All 3 steps visible with icons and descriptions | P0 |
| Renders gallery preview | Gallery preview section with 6-8 images loads | Images load and display in grid | P1 |
| Gallery preview link works | Click "View Gallery" link | Navigates to `/gallery` | P1 |
| Renders pricing preview | Pricing section displays price and benefits | Price "$9", benefits list visible | P0 |
| Pricing CTA navigates | Click "Get Started" on pricing section | Redirects to `/sign-up` | P0 |
| Renders FAQ section | FAQ accordion renders | All questions visible, expandable | P1 |
| FAQ accordion functionality | Click on FAQ question | Answer expands/collapses | P1 |
| Footer renders | Footer with links and copyright displays | All links present (About, Privacy, Terms) | P2 |
| Metadata set correctly | Page metadata configured | title and description in head | P0 |

#### Gallery Page (`/gallery`)

| Test Case | Description | Expected Result | Priority |
|-----------|-------------|-----------------|----------|
| Loads and displays grid | Gallery grid with images loads | Grid of images visible | P0 |
| Infinite scroll functionality | Scroll to bottom of page | More images load automatically | P0 |
| Image click expand | Click on gallery image | Image expands/modal opens | P0 |
| Requires login for download | Click download on modal (not logged in) | Redirected to `/sign-in` | P0 |
| Download PDF (logged in) | Click download on modal (logged in) | PDF download initiates | P0 |
| Empty state handling | No images returned from API | Empty state message displays | P1 |
| Loading state | Gallery initially loading | Skeleton cards display | P1 |
| Image lazy loading | Images below fold | Images lazy load as scrolled into view | P1 |
| Responsive grid | View on different screen sizes | Grid adjusts columns (mobile: 1, tablet: 2-3, desktop: 4) | P1 |

#### Generate Page (`/generate`)

| Test Case | Description | Expected Result | Priority |
|-----------|-------------|-----------------|----------|
| Protected route | Not logged in, access `/generate` | Redirected to `/sign-in` | P0 |
| Renders prompt input | Page loads | Textarea with placeholder "Describe what you want..." displays | P0 |
| Shows credit balance | Page loads (logged in) | Credit count displays (e.g., "🎨 12 credits") | P0 |
| Generate button disabled on empty | Textarea empty | Generate button disabled/grayed out | P1 |
| Generate button enabled | Textarea has content | Generate button enabled | P1 |
| Generate submission | Click Generate with valid prompt | Loading state shows, request sent to API | P0 |
| Generates image | API returns image | Image displays below prompt | P0 |
| Credits deducted | Image generated successfully | Credit count decreases by 1 | P0 |
| Download PDF button | Generated image displays | Download PDF button clickable | P1 |
| Share button | Generated image displays | Share button shows share options/modal | P1 |
| New button clears state | Click New after generation | Form clears, image hidden | P1 |
| Error handling | API returns error | Error message displays with retry button | P0 |
| Out of credits state | User has 0 credits | "You're out of credits!" message displays with "Buy Credits" CTA | P0 |
| Buy credits navigates | Click "Buy Credits" button | Redirects to `/pricing` | P0 |
| Loading state UI | Generation in progress | Spinner displays, prompt textarea disabled | P1 |

#### Library Page (`/library`)

| Test Case | Description | Expected Result | Priority |
|-----------|-------------|-----------------|----------|
| Protected route | Not logged in, access `/library` | Redirected to `/sign-in` | P0 |
| Displays user images | User has generated images | Grid of user's images displays | P0 |
| Empty state | User has no images | Empty state message with CTA to generate | P1 |
| Image actions visible | Hover/click image | Download, delete, share actions display | P0 |
| Delete image | Click delete on image | Image removed from grid, confirmation shown | P1 |
| Download image | Click download | PDF download initiates | P1 |
| Share image | Click share | Share modal/options display | P1 |
| Loading state | Page initially loading | Skeleton cards display | P1 |
| Pagination/infinite scroll | Many images | Load more/pagination works | P1 |
| Responsive grid | Different screen sizes | Grid adjusts columns appropriately | P1 |

#### Pricing Page (`/pricing`)

| Test Case | Description | Expected Result | Priority |
|-----------|-------------|-----------------|----------|
| Renders pricing card | Page loads | Pricing card displays: "20 Credits - $9" | P0 |
| Benefits list displays | Page loads | All benefits visible (PDF export, share, no expiry) | P0 |
| CTA button visible | Page loads | "Get Started" button displays | P0 |
| CTA navigates | Click "Get Started" (not logged in) | Redirected to `/sign-up` | P0 |
| CTA navigates (logged in) | Click "Get Started" (logged in) | Redirected to Stripe checkout | P0 |
| FAQ section | Page loads | FAQ section with questions visible | P1 |
| FAQ functionality | Click FAQ question | Answer expands/collapses | P1 |
| Security badge | Page loads | Stripe security badge displays | P1 |
| Metadata set | Page loads | title: "Pricing - PaintGPT" in head | P0 |

#### Account Page (`/account`)

| Test Case | Description | Expected Result | Priority |
|-----------|-------------|-----------------|----------|
| Protected route | Not logged in, access `/account` | Redirected to `/sign-in` | P0 |
| Displays user profile | Page loads (logged in) | User email and profile info displays | P0 |
| Manage account link | Page loads | "Manage Account" button visible and clickable | P1 |
| Displays credit balance | Page loads | Credit count displays (e.g., "12 credits remaining") | P0 |
| Buy more button | Page loads | "Buy More Credits" button visible | P1 |
| Buy credits navigates | Click "Buy More Credits" | Redirects to `/pricing` | P1 |
| Purchase history displays | User has purchases | Purchase history table/list displays | P0 |
| Purchase history empty | No purchases | "No purchases yet" message displays | P1 |
| All sections render | Page loads | Profile, Credits, and Purchase History sections visible | P0 |

### E2E Tests

#### Navigation Flows

| Test Case | Description | Steps | Expected Result | Priority |
|-----------|-------------|-------|-----------------|----------|
| Landing to Gallery | User explores public content | 1. Load `/` 2. Click "Explore Gallery" or gallery preview | Navigate to `/gallery`, images load | P0 |
| Landing to Sign Up | User creates account | 1. Load `/` 2. Click CTA "Create Your First..." | Redirect to `/sign-up`, Clerk form displays | P0 |
| Sign Up to Generate | New user creates image | 1. Complete sign-up 2. Redirect to `/generate` 3. Enter prompt 4. Click Generate | Image generates, displays, credit deducted | P0 |
| Generate to Library | User views personal library | 1. Generate image 2. Navigate to `/library` | Generated image appears in library | P0 |
| Library to Download | User downloads image | 1. Navigate to `/library` 2. Click download on image | PDF downloads successfully | P1 |
| Generate to Pricing | User runs out of credits | 1. Generate until out of credits 2. Click "Buy Credits" | Redirect to `/pricing`, pricing displays | P0 |
| Pricing to Checkout | User purchases credits | 1. Navigate to `/pricing` 2. Click "Get Started" 3. Fill Stripe form | Stripe checkout modal opens, transaction completes | P0 |
| Account management | User manages account | 1. Navigate to `/account` 2. Click "Manage Account" | Clerk account management modal opens | P1 |
| Gallery to Share | User shares image | 1. Navigate to `/gallery` 2. Click image 3. Click share | Share modal/options display, URL copyable | P1 |

#### Authentication Flows

| Test Case | Description | Expected Result | Priority |
|-----------|-------------|-----------------|----------|
| Sign Up flow complete | User creates account with email/password | Account created, redirected to `/generate` | P0 |
| Sign In flow complete | Registered user logs in | Session created, redirected to `/generate` | P0 |
| Logout flow | Logged in user clicks logout | Session destroyed, redirected to `/`, private routes blocked | P0 |
| Protected route access (not logged in) | Try to access `/generate`, `/library`, `/account` | Redirected to `/sign-in` | P0 |
| Protected route access (logged in) | Access `/generate`, `/library`, `/account` | Pages load and display user content | P0 |
| Session persistence | User logs in, refreshes page | Session persists, user remains logged in | P1 |

#### Payment Flows

| Test Case | Description | Expected Result | Priority |
|-----------|-------------|-----------------|----------|
| Checkout redirect | User on `/pricing` clicks "Get Started" | Stripe checkout session created, user redirected to Stripe | P0 |
| Payment success | User completes Stripe payment | Credits added to account, redirect to `/generate`, credits updated | P0 |
| Payment cancellation | User closes Stripe checkout | Redirected back to `/pricing` | P1 |
| Stripe webhook | Payment confirmed via webhook | Database updated with transaction record | P0 |

### Accessibility Tests

#### WCAG 2.1 AA Compliance

| Test Case | Component | Success Criteria | Expected Result | Priority |
|-----------|-----------|-----------------|-----------------|----------|
| Color contrast | All text | Minimum 4.5:1 ratio (normal text), 3:1 (large text) | All text passes contrast checker | P0 |
| Keyboard navigation | All pages | All interactive elements accessible via Tab key | Can tab through all buttons, links, inputs | P0 |
| Focus indicators | All interactive elements | Visible focus outline | Focus indicator visible on all focusable elements | P0 |
| ARIA labels | Form inputs | Proper labels or aria-label | All inputs have associated labels | P0 |
| Semantic HTML | All pages | Proper heading hierarchy, semantic elements | H1 per page, proper nesting H1 → H6 | P0 |
| Image alt text | All images | Descriptive alt text | All images have meaningful alt text | P0 |
| Button semantics | All buttons | Proper button elements or role=button | All clickable elements are proper buttons/links | P0 |
| Form validation | Forms (Sign up, etc.) | Error messages associated with inputs | aria-invalid, error messages linked to inputs | P1 |
| Skip links | All pages | Skip to main content link | Skip link visible and functional | P1 |
| Modal accessibility | Modals (image expand, share, etc.) | Proper focus management and ARIA | Focus trapped in modal, close accessible | P1 |
| Responsive text sizing | All pages | Text resizable to 200% | Page readable at 200% zoom without horizontal scroll | P1 |
| Animation preferences | Animated elements | Respects prefers-reduced-motion | Animations disabled if user prefers reduced motion | P1 |
| Link text context | All links | Descriptive link text or aria-label | Links not just "click here", context provided | P1 |
| Error prevention | Form submissions | Clear error messages | Form errors prevent submission, message explains issue | P1 |

#### Screen Reader Testing

| Test Case | Page | Tool | Expected Behavior | Priority |
|-----------|------|------|-------------------|----------|
| Page structure | All pages | NVDA/JAWS | H1 announces page title, landmarks announced | P0 |
| Navigation menu | Header | NVDA/JAWS | All nav items announced with context | P1 |
| Image descriptions | Gallery, Generate results | NVDA/JAWS | Image alt text announces purpose | P0 |
| Form labels | All forms | NVDA/JAWS | Input labels announced before input | P0 |
| Button purposes | All pages | NVDA/JAWS | Button text/labels clearly announce purpose | P0 |
| Error messages | Forms | NVDA/JAWS | Error messages announced when form invalid | P1 |
| Status updates | Generate page | NVDA/JAWS | Loading status announced, completion announced | P1 |
| Modal announcements | Modals | NVDA/JAWS | Modal title announced when opens | P1 |

#### Mobile/Touch Accessibility

| Test Case | Page | Expected Result | Priority |
|-----------|------|-----------------|----------|
| Touch targets 48px | All interactive elements | All buttons, links, inputs minimum 48x48px | P0 |
| No hover-only content | All pages | Important info not hidden until hover | P0 |
| Zoom functional | All pages | Content remains usable at 200% zoom | P1 |
| Orientation support | All pages | Portrait and landscape work equally well | P1 |
| Form input zoom | Mobile forms | Input fields don't zoom when focused (font-size ≥ 16px) | P1 |
