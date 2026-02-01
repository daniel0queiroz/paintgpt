# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

All commands run from `paintgpt/` directory using `bun`:

```bash
bun run dev      # Start dev server (localhost:3000)
bun run build    # Build for production
bun run start    # Run production server
bun run lint     # Run ESLint
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

## Security

Use the `security-auditor` agent after implementing features to audit APIs, database RLS, authentication, and data exposure.
