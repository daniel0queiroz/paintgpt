# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

All commands run from `paintgpt/` directory using `bun`:

```bash
bun run dev      # Start dev server (localhost:3000)
bun run build    # Build for production
bun run start    # Run production server
bun run lint     # Run ESLint
bun run test     # Run tests (Vitest)
bun run test:watch  # Run tests in watch mode
bun run test:coverage  # Run tests with coverage
```

## Tech Stack

- **Framework:** Next.js 16 (App Router)
- **React:** 19
- **Styling:** Tailwind CSS 4
- **Language:** TypeScript 5
- **Package Manager:** bun (not npm)

## Architecture

```
paintgpt/
├── app/              # Next.js App Router
│   ├── layout.tsx    # Root layout (fonts, metadata)
│   ├── page.tsx      # Home page
│   └── globals.css   # Tailwind + CSS variables
└── public/           # Static assets
```

**Path Alias:** `@/*` maps to project root

## Skills

| Skill           | Command            | Description                              |
| --------------- | ------------------ | ---------------------------------------- |
| PostHog         | `/posthog`         | Analytics, feature flags, session replay |
| SEO Technical   | `/seo-technical`   | Technical SEO for Next.js                |
| Marketing Copy  | `/marketing-copy`  | Direct Response copywriting              |
| UX Design       | `/ux-design`       | UX Design (Apple principles)             |
| Stripe          | `/stripe`          | International payments                   |
| AbacatePay      | `/abacatepay`      | PIX payments (Brazil)                    |
| Cloudflare      | `/cloudflare`      | DNS, domains, email routing, R2 storage  |
| Favicon         | `/favicon`         | Favicon and app icon generation          |
| Frontend Design | `/frontend-design` | Production-grade frontend interfaces     |

## Commands

| Command   | Description                                         |
| --------- | --------------------------------------------------- |
| `/commit` | Stage all changes and create commit with AI message |
| `/push`   | Push current branch to remote                       |
| `/pr`     | Create Pull Request on GitHub                       |
| `/ship`   | Commit + Push + PR in one command                   |

## Test-Driven Development (TDD)

Este projeto usa TDD. **Sempre escreva testes ANTES de implementar código.**

### Workflow TDD

1. **RED** - Escreva um teste que falha para a funcionalidade desejada
2. **GREEN** - Escreva o código mínimo para o teste passar
3. **REFACTOR** - Melhore o código mantendo os testes passando

### Estrutura de Testes

```
paintgpt/
├── __tests__/            # Testes unitários e integração
│   ├── api/              # Testes de API routes
│   ├── components/       # Testes de componentes
│   ├── hooks/            # Testes de hooks
│   └── lib/              # Testes de utilitários
├── e2e/                  # Testes end-to-end (Playwright)
└── vitest.config.ts      # Configuração Vitest
```

### Comandos de Teste

```bash
bun run test              # Rodar todos os testes
bun run test:watch        # Watch mode durante desenvolvimento
bun run test:coverage     # Relatório de cobertura
bun run test:e2e          # Testes E2E com Playwright
```

### Regras TDD

- **NUNCA** implemente código sem teste primeiro
- Cada nova feature deve ter testes correspondentes
- Cada bugfix deve ter teste que reproduz o bug
- Mantenha cobertura mínima de 80%
- Testes devem ser independentes e determinísticos

### Regras Invioláveis

- **NUNCA** burle ou desative testes para fazer código "funcionar"
- **NUNCA** use `.skip()`, `.only()`, ou comente testes para evitar falhas
- **NUNCA** modifique assertions para que passem artificialmente
- **NUNCA** faça commit com testes falhando
- Se um teste falha, **CORRIJA A IMPLEMENTAÇÃO**, não o teste
- Implementações devem ser feitas corretamente desde o início
- Prefira falhar rápido e corrigir do que entregar código quebrado
- Testes existem para garantir qualidade, não para serem contornados

### Ferramentas

| Ferramenta | Uso |
|------------|-----|
| Vitest | Testes unitários/integração |
| @testing-library/react | Testes de componentes |
| MSW | Mock de API/network |
| Playwright | Testes E2E |

## UI/UX

Sempre use a skill `/frontend-design` ao criar ou modificar interfaces de usuário. Isso garante componentes com alta qualidade visual, evitando estética genérica de AI.

## Security

Use the `security-auditor` agent after implementing features to audit APIs, database RLS, authentication, and data exposure.
