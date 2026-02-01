# PaintGPT - Product Requirements Document

## Overview

**Product:** PaintGPT - AI-powered coloring page generator for kids
**Target Users:** Parents with young children + Teachers/Schools
**Monetization:** Credit-based ($9 for 20 credits)

## Problem Statement

Parents and teachers need fresh, personalized coloring pages for children but:
- Existing coloring books are generic and repetitive
- Custom illustrations are expensive and slow to produce
- Kids want to color specific things (their pet, favorite character, etc.)

## Solution

An AI-powered web app that generates unique coloring pages based on any prompt, with a community gallery for sharing and discovering pages.

## Tech Stack

| Layer | Technology |
|-------|------------|
| Framework | Next.js 16 (App Router) |
| Auth | Clerk |
| Database | Neon (PostgreSQL) |
| Storage | Cloudflare R2 |
| Payments | Stripe |
| AI | OpenRouter (google/gemini-3-pro-image-preview) |
| Styling | Tailwind CSS 4 |

## Core Features

### 1. Image Generation
- **Prompt input:** User describes what they want (e.g., "a cat playing with yarn")
- **Style:** Simple line art optimized for coloring (ages 3-5)
- **Output:** Black and white line drawing, clean edges, no shading
- **Cost:** 1 credit per generation

### 2. Public Gallery
- Browse images shared by other users
- Filter by category/tags
- "Use this" button to save to personal collection
- Free to browse and save (no credit cost)

### 3. Personal Library
- View all generated images
- Download as PDF (A4, optimized for printing)
- Toggle public/private visibility
- Delete unwanted images

### 4. Authentication (Clerk)
- Sign up / Sign in (email, Google, Apple)
- User profile with credits balance
- Account management

### 5. Credits System
- **Package:** $9 = 20 credits
- **Usage:** 1 credit = 1 image generation
- Purchase via Stripe Checkout
- Credits never expire

### 6. PDF Download
- A4 format, portrait orientation
- High resolution for printing
- Minimal margins, maximum drawing area
- Included in credit cost (no extra charge)

## Database Schema

```sql
-- Users (synced from Clerk)
users (
  id UUID PRIMARY KEY,
  clerk_id TEXT UNIQUE NOT NULL,
  email TEXT NOT NULL,
  credits INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW()
)

-- Generated images
images (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  prompt TEXT NOT NULL,
  image_url TEXT NOT NULL,
  is_public BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT NOW()
)

-- Credit purchases
purchases (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  stripe_session_id TEXT UNIQUE,
  amount_cents INTEGER NOT NULL,
  credits INTEGER NOT NULL,
  status TEXT DEFAULT 'pending',
  created_at TIMESTAMP DEFAULT NOW()
)
```

## API Routes

| Method | Route | Description |
|--------|-------|-------------|
| POST | `/api/generate` | Generate new image (requires auth, costs 1 credit) |
| GET | `/api/images` | List user's images |
| GET | `/api/gallery` | List public images |
| PATCH | `/api/images/[id]` | Toggle public/private |
| DELETE | `/api/images/[id]` | Delete image |
| POST | `/api/checkout` | Create Stripe checkout session |
| POST | `/api/webhooks/stripe` | Handle Stripe webhooks |
| GET | `/api/download/[id]` | Generate and download PDF |

## Pages

| Route | Description |
|-------|-------------|
| `/` | Landing page + gallery preview |
| `/generate` | Image generation interface (auth required) |
| `/gallery` | Full public gallery |
| `/library` | User's personal images (auth required) |
| `/pricing` | Credits pricing page |
| `/account` | Account settings (auth required) |

## User Flows

### New User Flow
1. Land on homepage, see gallery preview
2. Click "Create Your Own"
3. Prompted to sign up (Clerk)
4. Redirected to pricing page
5. Purchase credits via Stripe
6. Start generating images

### Generation Flow
1. Enter prompt describing desired image
2. Click "Generate" (1 credit deducted)
3. Wait for AI generation (~10-15 seconds)
4. View result
5. Options: Download PDF, Share to Gallery, Generate Another

### Purchase Flow
1. Click "Buy Credits"
2. Redirected to Stripe Checkout
3. Complete payment
4. Webhook credits account
5. Redirected back to app with confirmation

## AI Prompt Engineering

Base prompt template for Gemini:
```
Generate a simple black and white coloring page for children ages 3-5.
The image should have:
- Clear, thick outlines
- No shading or gradients
- Simple shapes suitable for young children
- White background
- No text or letters

Subject: {user_prompt}
```

## MVP Scope

### Phase 1 (MVP)
- [ ] Landing page with value proposition
- [ ] Clerk authentication
- [ ] Single credit package ($9 = 20 credits)
- [ ] Basic image generation with one style
- [ ] Personal library
- [ ] PDF download
- [ ] Public gallery (view only)

### Phase 2
- [ ] Share to gallery toggle
- [ ] Gallery categories/tags
- [ ] Multiple drawing styles
- [ ] Bulk download

### Phase 3
- [ ] Subscription plans
- [ ] Teacher/classroom accounts
- [ ] Coloring progress tracking
- [ ] Mobile app

## Success Metrics

| Metric | Target (Month 1) |
|--------|------------------|
| Signups | 500 |
| Paid conversions | 5% |
| Images generated | 2,000 |
| Avg credits purchased/user | 40 |

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Inappropriate prompts | Content moderation via OpenRouter + prompt filtering |
| High AI costs | Monitor usage, adjust credit pricing if needed |
| Low conversion | A/B test pricing, add free trial credits |
| Image quality issues | Refine prompt engineering, allow regeneration |

## Open Questions

1. Should new users get free trial credits? How many?
2. Referral program for teachers?
3. Should we watermark gallery images?
