# The Better GreenShare - Implementation Plan

> **A comprehensive roadmap for building an industrial-standard community goods exchange platform**


## 🎯 Project Overview

**The Better GreenShare** is a complete rebuild of the GreenShare platform using the T3 Stack. GreenShare is a not-for-profit community platform that enables local communities to exchange goods sustainably.

The original implementation uses a Python Flask backend with in-memory data structures and a Next.js frontend (Pages Router). This rebuild creates a unified, type-safe, production-ready application using modern best practices.

### Current State
- ✅ Fresh T3 Stack scaffold with sample `Post` model/router
- ⏳ All GreenShare domain logic needs to be implemented
- ⏳ Database schema needs to be designed
- ⏳ Authentication needs to be configured

---

## 🔍 Original Application Analysis

### Core Features

**User Management**
- Registration/login with email+password or Google OAuth
- Password reset via email (SMTP)
- User profiles with first/last name

**Item Listings**
- Users list items for exchange or free giveaway
- Item details: title, description, condition, location, category, type
- Multiple images per item (via Imgur API)
- Auto-categorization using Google Gemini AI
- Location validation via Google Places API

**Exchange System**
- Users make exchange offers on items
- Offer includes: selected items from inventory + message
- State machine: pending → accepted → completed → confirmed (or cancelled)
- Both parties can cancel pending offers
- Exact location revealed only after acceptance

**Search & Discovery**
- Browse items by category, condition, type
- Full-text search with trigram similarity
- Filter and sort capabilities

### Critical Weaknesses in Original

| Issue | Impact | Solution in Better GreenShare |
|-------|--------|------------------------------|
| In-memory data storage | Data lost on restart | PostgreSQL with Prisma ORM |
| Hand-rolled session management | Security vulnerabilities | NextAuth.js 5.0 |
| Split Flask + Next.js architecture | No type safety | Unified T3 Stack with tRPC |
| All pages `"use client"` | Poor SEO, slow initial load | React Server Components + SSR |
| 3-second polling | Inefficient, high server load | TanStack Query with smart caching |
| No test coverage for frontend | Hard to maintain | Vitest + Playwright tests |
| No rate limiting | Vulnerable to abuse | Rate limiting on sensitive endpoints |
| CSRF token in localStorage | XSS vulnerability | HttpOnly cookies via NextAuth |
| No pagination | Performance issues at scale | Cursor-based pagination |

---

## 🛠 Technology Stack

### Framework & Core
- **Next.js 15.5+** - App Router with React Server Components
- **TypeScript** - Strict mode with `noUncheckedIndexedAccess`
- **React 19** - Latest features including Server Components

### API & Data
- **tRPC 11+** - End-to-end type-safe APIs
- **Prisma 7** - Modern ORM with PostgreSQL
- **TanStack Query 5+** - Client-side data fetching and caching
- **Zod** - Schema validation (shared client/server)

### Authentication & Security
- **NextAuth.js 5.0** - Authentication with credentials + Google OAuth
- **bcryptjs** - Password hashing
- **Rate limiting** - Protection against abuse

### Styling & UI
- **Tailwind CSS 4.1+** - Utility-first styling
- **shadcn/ui** (recommended) - Accessible component library
- **Lucide React** - Icon library

### External Services
- **Google Gemini AI** - Automatic item categorization
- **Google Places API** - Location autocomplete and validation
- **Imgur API** or **Cloud Storage** - Image hosting

### Testing
- **Vitest** - Unit and integration tests
- **Testing Library** - React component testing
- **Playwright** - End-to-end testing

### DevOps
- **Docker** - Containerization
- **GitHub Actions** - CI/CD pipeline
- **Vercel** - Deployment platform (or Docker-based)

---

## 🚀 Implementation Phases

### Phase 1: Database Schema and Core Models

**Priority:** 🔴 CRITICAL | **Effort:** 1 day

#### 1.1 Design the Prisma Schema

- [ ] Remove the sample `Post` model from `prisma/schema.prisma`
- [ ] Extend NextAuth `User` model with `firstName` and `lastName` fields
- [ ] Add `Item` model with fields:
  ```prisma
  model Item {
    id          String   @id @default(cuid())
    userId      String
    user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
    title       String   @db.VarChar(100)
    description String   @db.Text
    condition   ItemCondition
    status      ItemStatus @default(AVAILABLE)
    location    String   @db.VarChar(500)
    category    ItemCategory
    type        ItemType
    createdAt   DateTime @default(now())
    updatedAt   DateTime @updatedAt

    images      ItemImage[]
    offersReceived ExchangeOffer[] @relation("RequestedItem")
    offersIncluded OfferedItem[]

    @@index([userId])
    @@index([category])
    @@index([condition])
    @@index([type])
    @@index([status])
  }

  enum ItemCondition {
    NEW
    LIKE_NEW
    USED_GOOD
    USED_FAIR
    POOR
  }

  enum ItemStatus {
    AVAILABLE
    EXCHANGED
    DELETED
  }

  enum ItemCategory {
    ESSENTIALS
    LIVING
    TOOLS_TECH
    STYLE_EXPRESSION
    LEISURE_LEARNING
  }

  enum ItemType {
    FREE
    EXCHANGE
  }
  ```
- [ ] Add `ItemImage` model:
  ```prisma
  model ItemImage {
    id      String @id @default(cuid())
    itemId  String
    item    Item   @relation(fields: [itemId], references: [id], onDelete: Cascade)
    url     String
    order   Int

    @@index([itemId])
  }
  ```
- [ ] Add `ExchangeOffer` model:
  ```prisma
  model ExchangeOffer {
    id                String       @id @default(cuid())
    offeredById       String
    offeredBy         User         @relation(fields: [offeredById], references: [id], onDelete: Cascade)
    requestedItemId   String
    requestedItem     Item         @relation("RequestedItem", fields: [requestedItemId], references: [id], onDelete: Cascade)
    message           String       @db.Text
    status            OfferStatus  @default(PENDING)
    cancellationReason String?     @db.Text
    createdAt         DateTime     @default(now())
    updatedAt         DateTime     @updatedAt

    offeredItems      OfferedItem[]

    @@index([offeredById])
    @@index([requestedItemId])
    @@index([status])
  }

  enum OfferStatus {
    PENDING
    ACCEPTED
    COMPLETED
    CONFIRMED
    CANCELLED
  }
  ```
- [ ] Add `OfferedItem` join model:
  ```prisma
  model OfferedItem {
    id      String        @id @default(cuid())
    offerId String
    offer   ExchangeOffer @relation(fields: [offerId], references: [id], onDelete: Cascade)
    itemId  String
    item    Item          @relation(fields: [itemId], references: [id], onDelete: Cascade)

    @@index([offerId])
    @@index([itemId])
  }
  ```
- [ ] Run `pnpm db:push` to apply schema to development database
- [ ] Verify schema with `pnpm db:studio`

#### 1.2 Seed Data Script

- [ ] Create `prisma/seed.ts` with realistic test data
- [ ] Add seed script to `package.json`: `"db:seed": "tsx prisma/seed.ts"`
- [ ] Include test users, items across all categories, and sample offers in various states
- [ ] Add `tsx` as dev dependency: `pnpm add -D tsx`

---

### Phase 2: Authentication and Authorization

**Priority:** 🔴 CRITICAL | **Effort:** 2 days | **Depends on:** Phase 1

#### 2.1 Extend NextAuth Configuration

- [ ] Update `src/server/auth/config.ts`:
  - [ ] Add Credentials provider with email + password
  - [ ] Add Google OAuth provider (replace Discord)
  - [ ] Configure session strategy as JWT
  - [ ] Add `firstName` and `lastName` to session callbacks
  - [ ] Add user ID to session
- [ ] Update `src/env.js`:
  - [ ] Replace `AUTH_DISCORD_ID`/`AUTH_DISCORD_SECRET` with `AUTH_GOOGLE_ID`/`AUTH_GOOGLE_SECRET`
  - [ ] Add `GOOGLE_PLACES_API_KEY` (for location validation)
  - [ ] Add `IMGUR_CLIENT_ID` (or cloud storage credentials)
  - [ ] Add `GEMINI_API_KEY` (for AI categorization)
  - [ ] Add `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD` (for email)
- [ ] Update `.env.example` with new environment variables

#### 2.2 Password Hashing Utility

- [ ] Install bcryptjs: `pnpm add bcryptjs && pnpm add -D @types/bcryptjs`
- [ ] Create `src/server/auth/password.ts`:
  ```typescript
  import bcrypt from 'bcryptjs';

  export async function hashPassword(plainPassword: string): Promise<string> {
    return bcrypt.hash(plainPassword, 12);
  }

  export async function verifyPassword(
    plainPassword: string,
    hashedPassword: string
  ): Promise<boolean> {
    return bcrypt.compare(plainPassword, hashedPassword);
  }
  ```

#### 2.3 Password Reset Flow

- [ ] Install nodemailer: `pnpm add nodemailer && pnpm add -D @types/nodemailer`
- [ ] Create `src/server/email.ts` for sending emails
- [ ] Create `src/server/auth/reset-token.ts` for generating/verifying reset tokens
- [ ] Add tRPC procedures for `forgotPassword` and `resetPassword`

#### 2.4 Authorization Middleware

- [ ] Verify `protectedProcedure` in `src/server/api/trpc.ts` enforces authentication
- [ ] Create `ownerProcedure` middleware that checks resource ownership

---

### Phase 3: Core tRPC API Routers

**Priority:** 🔴 CRITICAL | **Effort:** 3 days | **Depends on:** Phase 1, 2

#### 3.1 Item Router

- [ ] Create `src/server/api/routers/item.ts`
- [ ] Implement procedures:
  - [ ] `getAll` (publicProcedure) - Paginated list with filters
  - [ ] `getById` (publicProcedure) - Single item details
  - [ ] `getByUser` (protectedProcedure) - Current user's items
  - [ ] `create` (protectedProcedure) - Create new item with validation
  - [ ] `update` (protectedProcedure) - Update item (verify ownership)
  - [ ] `delete` (protectedProcedure) - Soft delete (verify ownership)
  - [ ] `search` (publicProcedure) - Trigram similarity search
- [ ] Register in `src/server/api/root.ts`

#### 3.2 Offer Router

- [ ] Create `src/server/api/routers/offer.ts`
- [ ] Implement procedures:
  - [ ] `create` (protectedProcedure) - Create offer with business rules validation
  - [ ] `getByUser` (protectedProcedure) - Returns `{ incoming: [], outgoing: [] }`
  - [ ] `getById` (publicProcedure) - Single offer details
  - [ ] `accept` (protectedProcedure) - Only item owner can accept
  - [ ] `complete` (protectedProcedure) - Only offer creator can mark complete
  - [ ] `confirm` (protectedProcedure) - Only item owner can confirm
  - [ ] `cancel` (protectedProcedure) - Either party can cancel with reason
- [ ] Implement offer state machine with strict transitions:
  - PENDING → ACCEPTED (by item owner)
  - PENDING → CANCELLED (by either party)
  - ACCEPTED → COMPLETED (by offer creator)
  - COMPLETED → CONFIRMED (by item owner)
- [ ] Register in `src/server/api/root.ts`

#### 3.3 External Service Router

- [ ] Create `src/server/api/routers/external.ts`
- [ ] Implement procedures:
  - [ ] `autocompleteAddress` (protectedProcedure) - Google Places API proxy
  - [ ] `validateAddress` (protectedProcedure) - Address validation
  - [ ] `categoriseItem` (protectedProcedure) - Gemini AI categorization
- [ ] Create `src/server/services/imgur.ts` for image upload
- [ ] Register in `src/server/api/root.ts`

#### 3.4 Auth Router

- [ ] Create `src/server/api/routers/auth.ts`
- [ ] Implement procedures:
  - [ ] `register` (publicProcedure) - User registration
  - [ ] `forgotPassword` (publicProcedure) - Send reset email
  - [ ] `resetPassword` (publicProcedure) - Reset password with token
- [ ] Register in `src/server/api/root.ts`

#### 3.5 Clean Up Sample Code

- [ ] Delete `src/server/api/routers/post.ts`
- [ ] Delete `src/app/_components/post.tsx`
- [ ] Remove `postRouter` from `src/server/api/root.ts`

---

### Phase 4: Shared Validation and Utilities

**Priority:** 🔴 CRITICAL | **Effort:** 0.5 day

#### 4.1 Zod Schemas (Shared Between Client and Server)

- [ ] Create `src/lib/validators/item.ts`:
  ```typescript
  import { z } from 'zod';

  export const createItemSchema = z.object({
    title: z.string().min(3).max(100),
    description: z.string().min(10).max(1000),
    condition: z.enum(['NEW', 'LIKE_NEW', 'USED_GOOD', 'USED_FAIR', 'POOR']),
    type: z.enum(['FREE', 'EXCHANGE']),
    location: z.string().min(1).max(500),
    category: z.enum(['ESSENTIALS', 'LIVING', 'TOOLS_TECH', 'STYLE_EXPRESSION', 'LEISURE_LEARNING']),
    imageUrls: z.array(z.string().url()).min(1).max(10),
  });

  export const updateItemSchema = createItemSchema.partial();

  export const itemFilterSchema = z.object({
    category: z.enum(['ESSENTIALS', 'LIVING', 'TOOLS_TECH', 'STYLE_EXPRESSION', 'LEISURE_LEARNING']).optional(),
    condition: z.enum(['NEW', 'LIKE_NEW', 'USED_GOOD', 'USED_FAIR', 'POOR']).optional(),
    type: z.enum(['FREE', 'EXCHANGE']).optional(),
    search: z.string().optional(),
    cursor: z.string().optional(),
    limit: z.number().min(1).max(100).default(20),
  });
  ```

- [ ] Create `src/lib/validators/offer.ts`:
  ```typescript
  import { z } from 'zod';

  export const createOfferSchema = z.object({
    requestedItemId: z.string().cuid(),
    offeredItemIds: z.array(z.string().cuid()).min(0).max(10),
    message: z.string().min(10).max(2000),
  });

  export const cancelOfferSchema = z.object({
    offerId: z.string().cuid(),
    reason: z.string().min(10).max(500),
  });
  ```

- [ ] Create `src/lib/validators/auth.ts`:
  ```typescript
  import { z } from 'zod';

  const emailRegex = /^[0-9a-z]+([0-9a-z]*[-._+])*[0-9a-z]+@[0-9a-z]+([-.][0-9a-z]+)*([0-9a-z]*[.])[a-z]{2,8}$/;
  const passwordRegex = /^(?=.*\d)(?=.*[a-z])(?=.*[A-Z])(?=.*[!@#$%^&*])(?!.*\s).{8,32}$/;
  const nameRegex = /^[a-zA-Z\s\-'.]{2,50}$/;

  export const registerSchema = z.object({
    firstName: z.string().regex(nameRegex, 'Invalid first name'),
    lastName: z.string().regex(nameRegex, 'Invalid last name'),
    email: z.string().regex(emailRegex, 'Invalid email address'),
    password: z.string().regex(passwordRegex, 'Password must be 8-32 characters with uppercase, lowercase, digit, and special character'),
    confirmPassword: z.string(),
  }).refine((data) => data.password === data.confirmPassword, {
    message: "Passwords don't match",
    path: ["confirmPassword"],
  });

  export const loginSchema = z.object({
    email: z.string().email(),
    password: z.string().min(1),
  });

  export const forgotPasswordSchema = z.object({
    email: z.string().email(),
  });

  export const resetPasswordSchema = z.object({
    token: z.string(),
    password: z.string().regex(passwordRegex),
    confirmPassword: z.string(),
  }).refine((data) => data.password === data.confirmPassword, {
    message: "Passwords don't match",
    path: ["confirmPassword"],
  });
  ```

#### 4.2 Constants and Enums

- [ ] Create `src/lib/constants.ts`:
  ```typescript
  export const ITEM_CONDITIONS = ['NEW', 'LIKE_NEW', 'USED_GOOD', 'USED_FAIR', 'POOR'] as const;
  export const ITEM_TYPES = ['FREE', 'EXCHANGE'] as const;
  export const ITEM_CATEGORIES = [
    'ESSENTIALS',
    'LIVING',
    'TOOLS_TECH',
    'STYLE_EXPRESSION',
    'LEISURE_LEARNING',
  ] as const;
  export const OFFER_STATUSES = ['PENDING', 'ACCEPTED', 'COMPLETED', 'CONFIRMED', 'CANCELLED'] as const;

  export const PAGINATION_DEFAULTS = {
    ITEMS_PER_PAGE: 20,
    MAX_ITEMS_PER_PAGE: 100,
  } as const;
  ```

#### 4.3 Sanitization

- [ ] Install sanitization library: `pnpm add isomorphic-dompurify`
- [ ] Create `src/lib/sanitize.ts`:
  ```typescript
  import DOMPurify from 'isomorphic-dompurify';

  export function sanitizeHtml(dirty: string): string {
    return DOMPurify.sanitize(dirty, { ALLOWED_TAGS: [] });
  }
  ```

---

### Phase 5: Frontend Pages and Components

**Priority:** 🟠 HIGH | **Effort:** 4 days | **Depends on:** Phase 3, 4

#### 5.1 Layout and Navigation

- [ ] Update `src/app/layout.tsx`:
  - [ ] Add comprehensive metadata (title template, description, Open Graph)
  - [ ] Add viewport configuration
  - [ ] Verify TRPCReactProvider is set up
  - [ ] Add SessionProvider from NextAuth
- [ ] Create `src/app/_components/header.tsx`:
  - [ ] Logo/branding linking to home
  - [ ] Search bar with debounced input
  - [ ] Auth status (login button or user menu with logout)
  - [ ] Navigation links
- [ ] Create `src/app/_components/sidebar.tsx`:
  - [ ] Home link
  - [ ] Category links (5 categories)
  - [ ] Manage Items (authenticated only)
  - [ ] Manage Offers (authenticated only)

#### 5.2 UI Component Library

- [ ] Install shadcn/ui or create custom components in `src/app/_components/ui/`:
  - [ ] `button.tsx` - Button component with variants
  - [ ] `input.tsx` - Input field
  - [ ] `textarea.tsx` - Textarea field
  - [ ] `select.tsx` - Select dropdown
  - [ ] `card.tsx` - Card container
  - [ ] `badge.tsx` - Badge for tags/status
  - [ ] `modal.tsx` - Modal dialog
  - [ ] `skeleton.tsx` - Loading skeleton
  - [ ] `toast.tsx` - Toast notifications (or use `sonner`)

#### 5.3 Home Page (Browse Items)

- [ ] Rewrite `src/app/page.tsx` as Server Component:
  - [ ] Use `api.item.getAll` server caller for SSR
  - [ ] Display welcome banner with CTA
  - [ ] Render filter bar
  - [ ] Render item grid with pagination
- [ ] Create `src/app/_components/item-card.tsx` (client component):
  - [ ] Display item image, title, condition, type badges
  - [ ] Link to item detail page
  - [ ] Show category icon
- [ ] Create `src/app/_components/filter-bar.tsx` (client component):
  - [ ] Condition dropdown
  - [ ] Type dropdown
  - [ ] Category filter
  - [ ] Search input

#### 5.4 Category Pages

- [ ] Create `src/app/category/[slug]/page.tsx`:
  - [ ] Dynamic route for category slug
  - [ ] Server Component with SSR
  - [ ] Filter items by category
  - [ ] Reuse item grid and filter bar
  - [ ] Generate metadata for SEO

#### 5.5 Item Detail Page

- [ ] Create `src/app/items/[id]/page.tsx`:
  - [ ] Server Component with SSR using `api.item.getById`
  - [ ] Image carousel/gallery (use `next/image`)
  - [ ] Item details: title, description, condition, location, type, category
  - [ ] "Make an Offer" button (authenticated) or "Login to Make an Offer"
  - [ ] Show approximate location only (exact location after offer acceptance)
  - [ ] Generate dynamic metadata for SEO
  - [ ] Structured data (JSON-LD) for Product schema

#### 5.6 Authentication Pages

- [ ] Create `src/app/login/page.tsx`:
  - [ ] Email/password form with validation
  - [ ] Google OAuth button
  - [ ] Links to register and forgot password
  - [ ] Error display for failed login
  - [ ] Redirect to home or intended page after login
- [ ] Create `src/app/register/page.tsx`:
  - [ ] Form: firstName, lastName, email, password, confirmPassword
  - [ ] Client-side validation matching Zod schemas
  - [ ] Auto-login after successful registration
  - [ ] Link to login page
- [ ] Create `src/app/forgot-password/page.tsx`:
  - [ ] Email input form
  - [ ] Success message after submission
- [ ] Create `src/app/reset-password/page.tsx`:
  - [ ] Extract token from URL query params
  - [ ] New password + confirm password form
  - [ ] Success message and redirect to login

#### 5.7 Manage Items Page

- [ ] Create `src/app/manage/items/page.tsx` (protected):
  - [ ] Server Component fetching user's items
  - [ ] List items with status badges
  - [ ] "Add New Item" button opening modal
  - [ ] Edit and Delete buttons per item
  - [ ] Confirm dialog before deletion
- [ ] Create `src/app/_components/item-form.tsx`:
  - [ ] Shared form for create and edit
  - [ ] Fields: title, description, condition, type, location, images
  - [ ] Image upload with preview (max 10 images)
  - [ ] Location autocomplete using Google Places
  - [ ] Client-side validation
  - [ ] Submit to tRPC `item.create` or `item.update`

#### 5.8 Make Offer Page

- [ ] Create `src/app/items/[id]/offer/page.tsx` (protected):
  - [ ] Display requested item details
  - [ ] Multi-select from user's available items
  - [ ] Message textarea (10-2000 chars)
  - [ ] Validation: cannot offer on own items
  - [ ] Validation: must offer items for EXCHANGE type
  - [ ] Submit button

#### 5.9 Manage Offers Page

- [ ] Create `src/app/manage/offers/page.tsx` (protected):
  - [ ] Toggle between incoming and outgoing offers
  - [ ] Offer cards showing:
    - Items involved with images
    - Offer message
    - Current status with visual pipeline/stepper
    - Location (approximate or exact based on status)
  - [ ] Action buttons based on state and user role:
    - Item owner: Accept (pending), Confirm (completed), Cancel (pending)
    - Offer creator: Complete (accepted), Cancel (pending)
  - [ ] Cancellation dialog with reason input

---

### Phase 6: Image Upload

**Priority:** 🟠 HIGH | **Effort:** 1 day | **Depends on:** Phase 3

#### 6.1 Image Upload Pipeline

- [ ] Choose image storage solution:
  - [ ] Option A: Imgur API (carry over from original)
  - [ ] Option B (recommended): Cloud storage (AWS S3, Cloudflare R2, Vercel Blob)
- [ ] Create `src/server/services/image-upload.ts`:
  - [ ] Upload function with error handling
  - [ ] Image validation (file type, size)
  - [ ] Return uploaded URL
- [ ] Create API route `src/app/api/upload/route.ts`:
  - [ ] Accept multipart form data
  - [ ] Validate: JPEG, PNG, WebP, GIF only
  - [ ] Validate: max 5MB per image
  - [ ] Call upload service
  - [ ] Return JSON with URL

#### 6.2 Client-Side Upload Component

- [ ] Create `src/app/_components/image-upload.tsx`:
  - [ ] Drag and drop zone
  - [ ] Click to select files
  - [ ] Image preview grid with remove button
  - [ ] Upload progress indicator
  - [ ] Enforce max 10 images
  - [ ] Display upload errors

---

### Phase 7: AI Categorisation

**Priority:** 🟡 MEDIUM | **Effort:** 0.5 day | **Depends on:** Phase 3

#### 7.1 Gemini Integration

- [ ] Install Google Generative AI: `pnpm add @google/generative-ai`
- [ ] Create `src/server/services/categorisation.ts`:
  - [ ] Initialize Gemini AI client
  - [ ] Create categorization prompt with context
  - [ ] Input: item title + description
  - [ ] Output: one of 5 category enums
  - [ ] Parse response and validate
  - [ ] Fallback to "ESSENTIALS" on error
  - [ ] Cache results to reduce API costs
- [ ] Integrate into `external.categoriseItem` tRPC procedure
- [ ] Add manual category override in item form

---

### Phase 8: Search

**Priority:** 🟠 HIGH | **Effort:** 1 day | **Depends on:** Phase 1

#### 8.1 Full-Text and Trigram Search

- [ ] Enable `pg_trgm` PostgreSQL extension:
  - [ ] Create Prisma migration: `pnpm db:generate`
  - [ ] Add SQL: `CREATE EXTENSION IF NOT EXISTS pg_trgm;`
- [ ] Create GIN trigram index on `Item.title`:
  - [ ] Add to migration: `CREATE INDEX idx_item_title_trgm ON "Item" USING gin (title gin_trgm_ops);`
  - [ ] Run migration: `pnpm db:migrate`
- [ ] Implement `item.search` procedure using `$queryRaw`:
  ```typescript
  const results = await ctx.db.$queryRaw`
    SELECT * FROM "Item"
    WHERE similarity(title, ${query}) > 0.3
    ORDER BY similarity(title, ${query}) DESC
    LIMIT ${limit}
  `;
  ```

#### 8.2 Search UI

- [ ] Create `src/app/_components/search-bar.tsx`:
  - [ ] Debounced input (300ms delay)
  - [ ] Autocomplete dropdown with suggestions
  - [ ] Navigate to search results page on Enter
  - [ ] Use tRPC `item.search` procedure
- [ ] Create `src/app/search/page.tsx`:
  - [ ] Display search query
  - [ ] Show search results in item grid
  - [ ] Allow filtering by category/condition/type
  - [ ] Show "No results" message when empty

---

### Phase 9: Security Hardening

**Priority:** 🟠 HIGH | **Effort:** 1.5 days | **Depends on:** Phase 2, 3

#### 9.1 Input Validation

- [ ] All tRPC inputs validated with Zod schemas ✓ (enforced by tRPC)
- [ ] Server-side validation before all database operations
- [ ] HTML sanitization for user text fields (title, description, message)
- [ ] No client-side data trusted without server validation

#### 9.2 Authentication Security

- [ ] Password hashing with bcrypt (cost factor 12)
- [ ] Install rate limiting: `pnpm add @upstash/ratelimit @upstash/redis` (or use in-memory for dev)
- [ ] Create `src/server/rate-limit.ts`:
  ```typescript
  import { Ratelimit } from '@upstash/ratelimit';
  import { Redis } from '@upstash/redis';

  export const authRateLimit = new Ratelimit({
    redis: Redis.fromEnv(),
    limiter: Ratelimit.slidingWindow(5, '1 h'),
  });

  export const loginRateLimit = new Ratelimit({
    redis: Redis.fromEnv(),
    limiter: Ratelimit.slidingWindow(10, '1 m'),
  });
  ```
- [ ] Apply rate limiting to auth endpoints:
  - [ ] Register: 5 attempts per hour per IP
  - [ ] Login: 10 attempts per minute per IP
  - [ ] Forgot password: 3 attempts per hour per email
- [ ] Verify session cookies are: HttpOnly, Secure (production), SameSite=Lax
- [ ] Configure JWT token expiry in NextAuth (7 days default)

#### 9.3 Authorization

- [ ] Verify resource ownership on all mutations:
  - [ ] Item edit/delete: check `userId === session.user.id`
  - [ ] Offer actions: check user is item owner or offer creator
- [ ] Validate offer state transitions (no invalid transitions)
- [ ] Prevent users from offering on their own items
- [ ] Prevent duplicate offers (same user + item combination)

#### 9.4 HTTP Security Headers

- [ ] Update `next.config.js` with security headers:
  ```javascript
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-XSS-Protection', value: '1; mode=block' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          {
            key: 'Strict-Transport-Security',
            value: 'max-age=31536000; includeSubDomains',
          },
          {
            key: 'Content-Security-Policy',
            value: "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline'; style-src 'self' 'unsafe-inline';",
          },
        ],
      },
    ];
  },
  ```

#### 9.5 Additional Rate Limiting

- [ ] Apply rate limiting to:
  - [ ] Item create: 10 per hour per user
  - [ ] Offer create: 20 per hour per user
  - [ ] Image upload: 50 per hour per user
  - [ ] External API proxies: 60 per hour per user

#### 9.6 Environment Variable Security

- [ ] All secrets in `.env` (gitignored) ✓
- [ ] Zod validation in `src/env.js` ✓
- [ ] No `NEXT_PUBLIC_*` secrets exposed to client
- [ ] Use separate credentials for development and production

---

### Phase 10: Error Handling and Loading States

**Priority:** 🟠 HIGH | **Effort:** 1 day | **Depends on:** Phase 5

#### 10.1 Error Boundaries

- [ ] Create `src/app/error.tsx` (global error boundary):
  - [ ] User-friendly error message
  - [ ] "Try again" button
  - [ ] Log error to monitoring service
- [ ] Create `src/app/not-found.tsx` (404 page):
  - [ ] Custom 404 message
  - [ ] Link back to home
  - [ ] Search bar
- [ ] Create per-route error boundaries for critical pages:
  - [ ] `src/app/items/[id]/error.tsx`
  - [ ] `src/app/manage/items/error.tsx`
  - [ ] `src/app/manage/offers/error.tsx`

#### 10.2 Loading States

- [ ] Create `src/app/loading.tsx` (global loading UI):
  - [ ] Skeleton for header + content
- [ ] Create skeleton components:
  - [ ] `src/app/_components/skeletons/item-card-skeleton.tsx`
  - [ ] `src/app/_components/skeletons/item-detail-skeleton.tsx`
  - [ ] `src/app/_components/skeletons/offer-card-skeleton.tsx`
- [ ] Use Suspense boundaries for data-heavy sections:
  - [ ] Item grid on home page
  - [ ] Offer lists on manage offers page

#### 10.3 tRPC Error Handling

- [ ] Use consistent error codes in tRPC procedures:
  - `BAD_REQUEST` - Invalid input
  - `NOT_FOUND` - Resource doesn't exist
  - `UNAUTHORIZED` - Not authenticated
  - `FORBIDDEN` - Not authorized (authenticated but no permission)
  - `INTERNAL_SERVER_ERROR` - Server errors
- [ ] User-friendly error messages (not raw DB errors)
- [ ] Install toast library: `pnpm add sonner`
- [ ] Create `src/app/_components/toaster.tsx` for notifications
- [ ] Display errors as toasts in client components

---

### Phase 11: Performance Optimizations

**Priority:** 🟡 MEDIUM | **Effort:** 1.5 days | **Depends on:** Phase 5

#### 11.1 Server-Side Rendering

- [ ] Use React Server Components for all data-fetching pages ✓
- [ ] Only mark interactive components with `"use client"`
- [ ] Prefetch linked pages using Next.js `<Link prefetch>`
- [ ] Use `loading.tsx` and Suspense for streaming SSR

#### 11.2 Database Performance

- [ ] Verify all necessary indexes are created (from Phase 1) ✓
- [ ] Implement cursor-based pagination:
  - [ ] Use `cursor` (item ID) instead of `offset`
  - [ ] Return `nextCursor` in response
- [ ] Use Prisma `select` to fetch only needed fields:
  ```typescript
  const items = await ctx.db.item.findMany({
    select: {
      id: true,
      title: true,
      images: { take: 1, select: { url: true } },
      condition: true,
      type: true,
      category: true,
    },
  });
  ```
- [ ] Connection pooling verified via `@prisma/adapter-pg` ✓

#### 11.3 Image Optimization

- [ ] Use Next.js `<Image>` component for all images
- [ ] Configure image domains in `next.config.js`:
  ```javascript
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'i.imgur.com' },
      // Add other image domains
    ],
  },
  ```
- [ ] Set appropriate `sizes` attribute for responsive images
- [ ] Use `priority` for above-the-fold images
- [ ] Use `placeholder="blur"` with `blurDataURL` for better UX
- [ ] Lazy load below-the-fold images (default behavior)

#### 11.4 Bundle Optimization

- [ ] Analyze bundle size: `pnpm add -D @next/bundle-analyzer`
- [ ] Configure analyzer in `next.config.js`
- [ ] Use dynamic imports for heavy components:
  ```typescript
  const ImageCarousel = dynamic(() => import('./image-carousel'), {
    loading: () => <Skeleton />,
  });
  ```
- [ ] Consider lighter alternatives for heavy dependencies
- [ ] Tree-shake unused imports

#### 11.5 Caching Strategy

- [ ] Configure TanStack Query cache times:
  ```typescript
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // 1 minute
        gcTime: 5 * 60 * 1000, // 5 minutes
      },
    },
  });
  ```
- [ ] Server-side caching for public pages:
  ```typescript
  export const revalidate = 60; // Revalidate every 60 seconds
  ```
- [ ] Cache external API responses (address autocomplete, AI categorization)
- [ ] Use Redis for rate limiting and caching (production)

---

### Phase 12: Testing Strategy

**Priority:** 🟠 HIGH | **Effort:** 3 days | **Depends on:** Phase 3, 5

#### 12.1 Setup Testing Infrastructure

- [ ] Install testing dependencies:
  ```bash
  pnpm add -D vitest @vitest/ui @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom vitest-mock-extended @playwright/test
  ```
- [ ] Create `vitest.config.ts`:
  ```typescript
  import { defineConfig } from 'vitest/config';
  import react from '@vitejs/plugin-react';

  export default defineConfig({
    plugins: [react()],
    test: {
      environment: 'jsdom',
      setupFiles: ['./vitest.setup.ts'],
    },
    resolve: {
      alias: {
        '@': '/src',
      },
    },
  });
  ```
- [ ] Create `vitest.setup.ts`:
  ```typescript
  import '@testing-library/jest-dom';
  ```
- [ ] Create `playwright.config.ts`
- [ ] Add test scripts to `package.json`:
  ```json
  {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui"
  }
  ```

#### 12.2 Unit Tests

- [ ] Test Zod validation schemas:
  - [ ] `src/lib/validators/item.test.ts` - Test all edge cases
  - [ ] `src/lib/validators/offer.test.ts` - Test all edge cases
  - [ ] `src/lib/validators/auth.test.ts` - Test regex patterns
- [ ] Test utility functions:
  - [ ] `src/lib/sanitize.test.ts` - Test HTML sanitization
  - [ ] `src/server/auth/password.test.ts` - Test hashing/verification
- [ ] Test tRPC procedures with mocked Prisma:
  - [ ] `src/server/api/routers/item.test.ts`
  - [ ] `src/server/api/routers/offer.test.ts`
  - [ ] `src/server/api/routers/auth.test.ts`

#### 12.3 Integration Tests

- [ ] Test authentication flow:
  - [ ] User registration
  - [ ] Email/password login
  - [ ] Google OAuth login
  - [ ] Password reset
- [ ] Test item lifecycle:
  - [ ] Create item
  - [ ] Update item
  - [ ] Delete item
  - [ ] List items with filters
  - [ ] Search items
- [ ] Test offer state machine:
  - [ ] Create offer
  - [ ] Accept offer → mark items as exchanged
  - [ ] Complete offer
  - [ ] Confirm offer
  - [ ] Cancel offer at each stage
  - [ ] Invalid state transitions throw errors
- [ ] Test authorization:
  - [ ] Users cannot edit others' items
  - [ ] Users cannot accept offers on items they don't own
  - [ ] Users cannot offer on their own items

#### 12.4 End-to-End Tests

- [ ] E2E test scenarios:
  - [ ] **Happy path**: User A registers → creates item → User B makes offer → User A accepts → User B completes → User A confirms
  - [ ] **Search and filter**: Search for items, filter by category/condition/type
  - [ ] **Authentication**: Login, logout, register, password reset
  - [ ] **Authorization**: Attempt to access protected pages without login (redirect to login)
  - [ ] **Error states**: Submit invalid forms, handle network errors
  - [ ] **Cancellation**: Cancel offer at pending stage by both parties

#### 12.5 Coverage Goals

- [ ] Aim for 80%+ code coverage on server-side code
- [ ] Aim for 70%+ code coverage on client-side code
- [ ] Critical paths (auth, offer state machine) must have 100% coverage

---

### Phase 13: DevOps and CI/CD

**Priority:** 🟡 MEDIUM | **Effort:** 1.5 days | **Depends on:** Phase 12

#### 13.1 GitHub Actions - CI Pipeline

- [ ] Create `.github/workflows/ci.yml`:
  ```yaml
  name: CI

  on:
    push:
      branches: [main]
    pull_request:
      branches: [main]

  jobs:
    test:
      runs-on: ubuntu-latest

      services:
        postgres:
          image: postgres:16
          env:
            POSTGRES_PASSWORD: postgres
          options: >-
            --health-cmd pg_isready
            --health-interval 10s
            --health-timeout 5s
            --health-retries 5
          ports:
            - 5432:5432

      steps:
        - uses: actions/checkout@v4
        - uses: pnpm/action-setup@v2
        - uses: actions/setup-node@v4
          with:
            node-version: 20
            cache: 'pnpm'

        - name: Install dependencies
          run: pnpm install

        - name: Run linter
          run: pnpm lint

        - name: Run type check
          run: pnpm typecheck

        - name: Run unit tests
          run: pnpm test
          env:
            DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test

        - name: Build
          run: pnpm build
  ```

#### 13.2 GitHub Actions - E2E Tests

- [ ] Create `.github/workflows/e2e.yml`:
  ```yaml
  name: E2E Tests

  on:
    pull_request:
      branches: [main]

  jobs:
    e2e:
      runs-on: ubuntu-latest

      steps:
        - uses: actions/checkout@v4
        - uses: pnpm/action-setup@v2
        - uses: actions/setup-node@v4
          with:
            node-version: 20
            cache: 'pnpm'

        - name: Install dependencies
          run: pnpm install

        - name: Install Playwright browsers
          run: pnpm exec playwright install --with-deps

        - name: Run E2E tests
          run: pnpm test:e2e

        - name: Upload test artifacts
          if: failure()
          uses: actions/upload-artifact@v4
          with:
            name: playwright-report
            path: playwright-report/
  ```

#### 13.3 Docker Setup

- [ ] Create `Dockerfile`:
  ```dockerfile
  FROM node:20-alpine AS base
  RUN corepack enable pnpm

  FROM base AS deps
  WORKDIR /app
  COPY package.json pnpm-lock.yaml ./
  RUN pnpm install --frozen-lockfile

  FROM base AS builder
  WORKDIR /app
  COPY --from=deps /app/node_modules ./node_modules
  COPY . .
  RUN pnpm build

  FROM base AS runner
  WORKDIR /app
  ENV NODE_ENV=production

  RUN addgroup --system --gid 1001 nodejs
  RUN adduser --system --uid 1001 nextjs

  COPY --from=builder /app/public ./public
  COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
  COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

  USER nextjs
  EXPOSE 3000
  ENV PORT=3000

  CMD ["node", "server.js"]
  ```

- [ ] Create `docker-compose.yml`:
  ```yaml
  version: '3.8'

  services:
    app:
      build: .
      ports:
        - '3000:3000'
      environment:
        DATABASE_URL: postgresql://postgres:postgres@db:5432/greenshare
      depends_on:
        - db

    db:
      image: postgres:16-alpine
      environment:
        POSTGRES_USER: postgres
        POSTGRES_PASSWORD: postgres
        POSTGRES_DB: greenshare
      volumes:
        - postgres_data:/var/lib/postgresql/data
      ports:
        - '5432:5432'

  volumes:
    postgres_data:
  ```

- [ ] Create `.dockerignore`:
  ```
  node_modules
  .next
  .git
  .env
  .env.local
  README.md
  ```

#### 13.4 Health Check Endpoint

- [ ] Create `src/app/api/health/route.ts`:
  ```typescript
  import { db } from '@/server/db';
  import { NextResponse } from 'next/server';

  export async function GET() {
    try {
      await db.$queryRaw`SELECT 1`;
      return NextResponse.json({
        status: 'ok',
        timestamp: new Date().toISOString(),
        uptime: process.uptime(),
      });
    } catch (error) {
      return NextResponse.json(
        { status: 'error', message: 'Database connection failed' },
        { status: 503 }
      );
    }
  }
  ```

---

### Phase 14: Code Quality and Developer Experience

**Priority:** 🟡 MEDIUM | **Effort:** 1 day

#### 14.1 Linting and Formatting

- [ ] Verify ESLint configuration covers:
  - [ ] TypeScript strict rules ✓
  - [ ] React hooks rules ✓
  - [ ] Import ordering
  - [ ] No unused variables
  - [ ] No console.logs in production
- [ ] Verify Prettier handles Tailwind class sorting ✓
- [ ] Install husky and lint-staged:
  ```bash
  pnpm add -D husky lint-staged
  pnpm exec husky init
  ```
- [ ] Create `.husky/pre-commit`:
  ```bash
  #!/bin/sh
  pnpm lint-staged
  ```
- [ ] Add to `package.json`:
  ```json
  {
    "lint-staged": {
      "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],
      "*.{json,md,yml,yaml}": ["prettier --write"]
    }
  }
  ```

#### 14.2 Commit Conventions

- [ ] Install commitlint:
  ```bash
  pnpm add -D @commitlint/cli @commitlint/config-conventional
  ```
- [ ] Create `.commitlintrc.json`:
  ```json
  {
    "extends": ["@commitlint/config-conventional"]
  }
  ```
- [ ] Create `.husky/commit-msg`:
  ```bash
  #!/bin/sh
  pnpm commitlint --edit $1
  ```

#### 14.3 GitHub Templates

- [ ] Create `.github/pull_request_template.md`:
  ```markdown
  ## Description
  <!-- Brief description of changes -->

  ## Type of Change
  - [ ] Bug fix
  - [ ] New feature
  - [ ] Breaking change
  - [ ] Documentation update

  ## Checklist
  - [ ] Code follows style guidelines
  - [ ] Self-review completed
  - [ ] Tests added/updated
  - [ ] Documentation updated
  - [ ] No new warnings
  ```

- [ ] Create `.github/ISSUE_TEMPLATE/bug_report.md`
- [ ] Create `.github/ISSUE_TEMPLATE/feature_request.md`

#### 14.4 Code Organization Standards

- [ ] File naming: `kebab-case.tsx` for files, `PascalCase` for components
- [ ] Server-only code stays in `src/server/`
- [ ] Client components in `src/app/_components/`
- [ ] Shared code in `src/lib/`
- [ ] No barrel exports (use direct imports)
- [ ] Maximum file size: 300 lines (split if larger)

---

### Phase 15: Accessibility (WCAG 2.1 AA)

**Priority:** 🟠 HIGH | **Effort:** 2 days | **Depends on:** Phase 5

#### 15.1 Semantic HTML

- [ ] Use proper heading hierarchy (h1 → h2 → h3, no skipping)
- [ ] Use semantic landmarks:
  - [ ] `<header>` for site header
  - [ ] `<nav>` for navigation
  - [ ] `<main>` for main content
  - [ ] `<footer>` for site footer
  - [ ] `<section>` for content sections
  - [ ] `<article>` for item cards
- [ ] Use `<button>` for actions, `<Link>` for navigation
- [ ] All form inputs have associated `<label>` elements
- [ ] Use `<fieldset>` and `<legend>` for grouped form controls

#### 15.2 ARIA and Keyboard Navigation

- [ ] All interactive elements keyboard accessible (Tab, Enter, Escape, Arrow keys)
- [ ] Modal dialogs:
  - [ ] Trap focus within modal
  - [ ] Restore focus on close
  - [ ] Close on Escape key
- [ ] Add `aria-label` to icon-only buttons
- [ ] Add `aria-live` regions for dynamic updates:
  - [ ] Toast notifications
  - [ ] Filter results count
  - [ ] Offer status changes
- [ ] Add skip navigation link:
  ```tsx
  <a href="#main-content" className="sr-only focus:not-sr-only">
    Skip to main content
  </a>
  ```
- [ ] Dropdown menus navigable with arrow keys

#### 15.3 Visual Accessibility

- [ ] Color contrast ratio meets WCAG AA:
  - [ ] Normal text: 4.5:1
  - [ ] Large text: 3:1
  - [ ] UI components: 3:1
- [ ] Do not rely solely on color to convey information:
  - [ ] Use icons + text for status badges
  - [ ] Use patterns in addition to colors
- [ ] Focus indicators visible on all interactive elements (outline or ring)
- [ ] Responsive design works at 200% zoom
- [ ] Text remains readable at 200% zoom without horizontal scrolling

#### 15.4 Image Accessibility

- [ ] All images have meaningful `alt` text describing the item
- [ ] Decorative images use `alt=""` or `aria-hidden="true"`
- [ ] Image carousels:
  - [ ] Keyboard navigable
  - [ ] Announce current slide (e.g., "Image 2 of 5")
  - [ ] Provide text alternative for image content

#### 15.5 Form Accessibility

- [ ] Error messages:
  - [ ] Associated with form controls via `aria-describedby`
  - [ ] Announced to screen readers
  - [ ] Visible and clear
- [ ] Required fields indicated with `aria-required="true"` and visual indicator
- [ ] Input purpose identified with `autocomplete` attributes

#### 15.6 Accessibility Testing

- [ ] Run automated tests with axe-core or similar
- [ ] Manual keyboard navigation testing
- [ ] Screen reader testing (NVDA, JAWS, VoiceOver)

---

### Phase 16: SEO Optimization

**Priority:** 🟡 MEDIUM | **Effort:** 1 day | **Depends on:** Phase 5

#### 16.1 Metadata Configuration

- [ ] Update `src/app/layout.tsx` with comprehensive metadata:
  ```typescript
  export const metadata: Metadata = {
    title: {
      template: '%s | GreenShare',
      default: 'GreenShare - Sustainable Community Goods Exchange',
    },
    description: 'Exchange and give away items sustainably within your local community. Join GreenShare to reduce waste and support sustainable living.',
    keywords: ['sustainability', 'exchange', 'community', 'green', 'sharing economy', 'circular economy'],
    authors: [{ name: 'GreenShare Team' }],
    openGraph: {
      title: 'GreenShare',
      description: 'Sustainable Community Goods Exchange',
      type: 'website',
      locale: 'en_US',
      siteName: 'GreenShare',
    },
    twitter: {
      card: 'summary_large_image',
      title: 'GreenShare',
      description: 'Sustainable Community Goods Exchange',
    },
  };
  ```

- [ ] Generate dynamic metadata for item detail pages:
  ```typescript
  export async function generateMetadata({ params }): Promise<Metadata> {
    const item = await api.item.getById({ id: params.id });
    return {
      title: item.title,
      description: item.description,
      openGraph: {
        title: item.title,
        description: item.description,
        images: item.images.map(img => ({ url: img.url })),
      },
    };
  }
  ```

- [ ] Generate dynamic metadata for category pages

#### 16.2 Technical SEO

- [ ] Create `src/app/sitemap.ts`:
  ```typescript
  import { MetadataRoute } from 'next';
  import { api } from '@/trpc/server';

  export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
    const items = await api.item.getAll({ limit: 1000 });

    return [
      {
        url: 'https://greenshare.com',
        lastModified: new Date(),
        changeFrequency: 'daily',
        priority: 1,
      },
      ...items.map((item) => ({
        url: `https://greenshare.com/items/${item.id}`,
        lastModified: item.updatedAt,
        changeFrequency: 'weekly' as const,
        priority: 0.8,
      })),
    ];
  }
  ```

- [ ] Create `src/app/robots.ts`:
  ```typescript
  import { MetadataRoute } from 'next';

  export default function robots(): MetadataRoute.Robots {
    return {
      rules: {
        userAgent: '*',
        allow: '/',
        disallow: ['/manage/', '/api/'],
      },
      sitemap: 'https://greenshare.com/sitemap.xml',
    };
  }
  ```

- [ ] Ensure SSR for all public pages ✓
- [ ] Use canonical URLs in metadata
- [ ] Structured data (JSON-LD) for item listings:
  ```typescript
  const structuredData = {
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: item.title,
    description: item.description,
    image: item.images.map(img => img.url),
    offers: {
      '@type': 'Offer',
      price: '0',
      priceCurrency: 'USD',
      availability: 'https://schema.org/InStock',
    },
  };
  ```

---

### Phase 17: Monitoring, Logging, and Observability

**Priority:** 🔵 LOW | **Effort:** 1 day | **Depends on:** Phase 13

#### 17.1 Error Tracking

- [ ] Install Sentry: `pnpm add @sentry/nextjs`
- [ ] Run Sentry wizard: `pnpm exec sentry-wizard -i nextjs`
- [ ] Configure `sentry.client.config.ts` and `sentry.server.config.ts`
- [ ] Add `SENTRY_DSN` to environment variables
- [ ] Configure source maps upload for production debugging
- [ ] Add custom error context (user ID, route, etc.)

#### 17.2 Logging

- [ ] Install logging library: `pnpm add pino pino-pretty`
- [ ] Create `src/server/logger.ts`:
  ```typescript
  import pino from 'pino';

  export const logger = pino({
    level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
    ...(process.env.NODE_ENV !== 'production' && {
      transport: {
        target: 'pino-pretty',
        options: { colorize: true },
      },
    }),
  });
  ```

- [ ] Log all tRPC procedure calls with timing:
  ```typescript
  const loggingMiddleware = t.middleware(async ({ path, type, next }) => {
    const start = Date.now();
    const result = await next();
    const duration = Date.now() - start;
    logger.info({ path, type, duration }, 'tRPC call');
    return result;
  });
  ```

- [ ] Log authentication events:
  - [ ] Successful login
  - [ ] Failed login attempts
  - [ ] User registration
  - [ ] Password reset requests
- [ ] Never log sensitive data (passwords, tokens, personal info)

#### 17.3 Performance Monitoring

- [ ] Consider Vercel Analytics (automatic if deployed on Vercel)
- [ ] Or install web-vitals: `pnpm add web-vitals`
- [ ] Track Core Web Vitals: LCP, FID, CLS, FCP, TTFB

---

### Phase 18: Documentation

**Priority:** 🟡 MEDIUM | **Effort:** 1 day | **Depends on:** All phases

#### 18.1 README

- [ ] Rewrite `README.md` with:
  - [ ] Project overview and purpose
  - [ ] Features list
  - [ ] Screenshots/demo
  - [ ] Tech stack
  - [ ] Prerequisites (Node.js 20+, pnpm, Docker/Podman)
  - [ ] Setup instructions:
    - Clone repository
    - Install dependencies
    - Environment setup
    - Database setup
    - Run development server
  - [ ] Available scripts
  - [ ] Folder structure
  - [ ] Architecture overview
  - [ ] Contributing guidelines
  - [ ] License

#### 18.2 API Documentation

- [ ] Add JSDoc comments to all tRPC procedures:
  ```typescript
  /**
   * Creates a new item listing.
   * @requires Authentication
   * @throws {TRPCError} BAD_REQUEST if validation fails
   */
  create: protectedProcedure
    .input(createItemSchema)
    .mutation(async ({ ctx, input }) => {
      // ...
    });
  ```

- [ ] Create `docs/API.md` with:
  - [ ] Overview of tRPC API
  - [ ] Authentication
  - [ ] Available routers and procedures
  - [ ] Request/response examples
  - [ ] Error codes

- [ ] Document offer state machine:
  ```markdown
  ## Offer State Machine

  ```mermaid
  stateDiagram-v2
      [*] --> PENDING
      PENDING --> ACCEPTED: Item owner accepts
      PENDING --> CANCELLED: Either party cancels
      ACCEPTED --> COMPLETED: Offer creator marks complete
      COMPLETED --> CONFIRMED: Item owner confirms
      CONFIRMED --> [*]
      CANCELLED --> [*]
  ```
  ```

#### 18.3 Architecture Decision Records

- [ ] Create `docs/adr/` directory
- [ ] Create ADRs:
  - [ ] `001-t3-stack-over-flask-nextjs.md`
  - [ ] `002-trpc-over-rest.md`
  - [ ] `003-nextauth-over-custom-auth.md`
  - [ ] `004-image-storage-strategy.md`
  - [ ] `005-search-implementation.md`

#### 18.4 Developer Onboarding

- [ ] Create `docs/DEVELOPMENT.md`:
  - [ ] Development workflow
  - [ ] Coding standards
  - [ ] Git workflow
  - [ ] Testing guidelines
  - [ ] How to add new features

---

### Phase 19: Production Deployment Readiness

**Priority:** 🔴 CRITICAL | **Effort:** 1 day | **Depends on:** All phases

#### 19.1 Production Configuration

- [ ] Update `next.config.js`:
  - [ ] Configure image domains for production
  - [ ] Add security headers ✓
  - [ ] Remove development-only settings
  - [ ] Configure output: 'standalone' for Docker
- [ ] Verify artificial tRPC delay is disabled in production ✓ (gated by `NODE_ENV`)
- [ ] Create production environment variables checklist:
  - [ ] `DATABASE_URL` (production database)
  - [ ] `AUTH_SECRET`
  - [ ] `AUTH_GOOGLE_ID` / `AUTH_GOOGLE_SECRET`
  - [ ] `GOOGLE_PLACES_API_KEY`
  - [ ] `IMGUR_CLIENT_ID` (or cloud storage credentials)
  - [ ] `GEMINI_API_KEY`
  - [ ] `SMTP_*` credentials
  - [ ] `SENTRY_DSN`
  - [ ] `NEXTAUTH_URL` (production URL)

#### 19.2 Database Production Setup

- [ ] Choose managed PostgreSQL provider:
  - [ ] Option A: Neon (serverless, free tier)
  - [ ] Option B: Supabase (includes auth, storage)
  - [ ] Option C: AWS RDS
  - [ ] Option D: Vercel Postgres
- [ ] Configure connection pooling (PgBouncer or Prisma Accelerate)
- [ ] Set up automated backups
- [ ] Run migrations: `pnpm db:migrate`
- [ ] Migration deployment strategy:
  - [ ] Run migrations in CI/CD before deployment
  - [ ] Or run manually before each deploy
- [ ] Seed production database with initial data (if needed)

#### 19.3 Deployment Platform

**Option A: Vercel (Recommended)**

- [ ] Connect GitHub repository to Vercel
- [ ] Configure environment variables in Vercel dashboard
- [ ] Set build command: `pnpm build`
- [ ] Set install command: `pnpm install`
- [ ] Enable preview deployments for PRs
- [ ] Configure custom domain
- [ ] Set up production/preview environments

**Option B: Docker-based Deployment**

- [ ] Build Docker image: `docker build -t greenshare .`
- [ ] Push to container registry (Docker Hub, AWS ECR, GCP Artifact Registry)
- [ ] Deploy to hosting platform (AWS ECS, GCP Cloud Run, Railway, Fly.io)
- [ ] Configure environment variables
- [ ] Set up load balancer
- [ ] Configure SSL/TLS certificate

#### 19.4 Post-Deployment Verification

- [ ] Health check endpoint returns 200 OK
- [ ] Database migrations applied successfully
- [ ] Authentication works (credentials + Google OAuth)
- [ ] Image upload works
- [ ] All external APIs functional (Google Places, Gemini, Imgur/storage)
- [ ] Email sending works (SMTP)
- [ ] Error tracking works (Sentry receives test error)
- [ ] Performance monitoring active
- [ ] SEO metadata rendering correctly
- [ ] Sitemap accessible at `/sitemap.xml`
- [ ] Robots.txt accessible at `/robots.txt`

#### 19.5 Launch Checklist

- [ ] All environment variables set
- [ ] Database backed up
- [ ] SSL certificate configured
- [ ] DNS configured
- [ ] Analytics/monitoring configured
- [ ] Error tracking configured
- [ ] Rate limiting active
- [ ] Security headers verified
- [ ] Load testing completed
- [ ] E2E tests passing in production-like environment
- [ ] Rollback plan documented

---

## 📊 Priority & Timeline

### Priority Legend
- 🔴 **CRITICAL** - Must be completed for MVP
- 🟠 **HIGH** - Important for production readiness
- 🟡 **MEDIUM** - Improves quality and maintainability
- 🔵 **LOW** - Nice to have, can be deferred

### Timeline Estimate

| Phase | Priority | Effort | Dependencies |
|-------|----------|--------|--------------|
| Phase 1: Database Schema | 🔴 CRITICAL | 1 day | None |
| Phase 2: Authentication | 🔴 CRITICAL | 2 days | Phase 1 |
| Phase 3: Core API Routers | 🔴 CRITICAL | 3 days | Phase 1, 2 |
| Phase 4: Shared Validation | 🔴 CRITICAL | 0.5 day | None |
| Phase 5: Frontend Pages | 🟠 HIGH | 4 days | Phase 3, 4 |
| Phase 6: Image Upload | 🟠 HIGH | 1 day | Phase 3 |
| Phase 7: AI Categorisation | 🟡 MEDIUM | 0.5 day | Phase 3 |
| Phase 8: Search | 🟠 HIGH | 1 day | Phase 1 |
| Phase 9: Security Hardening | 🟠 HIGH | 1.5 days | Phase 2, 3 |
| Phase 10: Error Handling | 🟠 HIGH | 1 day | Phase 5 |
| Phase 11: Performance | 🟡 MEDIUM | 1.5 days | Phase 5 |
| Phase 12: Testing | 🟠 HIGH | 3 days | Phase 3, 5 |
| Phase 13: DevOps/CI/CD | 🟡 MEDIUM | 1.5 days | Phase 12 |
| Phase 14: Code Quality | 🟡 MEDIUM | 1 day | None |
| Phase 15: Accessibility | 🟠 HIGH | 2 days | Phase 5 |
| Phase 16: SEO | 🟡 MEDIUM | 1 day | Phase 5 |
| Phase 17: Monitoring | 🔵 LOW | 1 day | Phase 13 |
| Phase 18: Documentation | 🟡 MEDIUM | 1 day | All phases |
| Phase 19: Production Readiness | 🔴 CRITICAL | 1 day | All phases |

**Total Estimated Effort:** ~28 days

### Recommended Execution Order

**Sprint 1: Foundation (6.5 days)**
1. Phase 1: Database Schema
2. Phase 2: Authentication
3. Phase 3: Core API Routers
4. Phase 4: Shared Validation

**Sprint 2: Frontend & Features (6.5 days)**
5. Phase 5: Frontend Pages
6. Phase 6: Image Upload
7. Phase 8: Search
8. Phase 7: AI Categorisation

**Sprint 3: Quality & Security (5.5 days)**
9. Phase 9: Security Hardening
10. Phase 10: Error Handling
11. Phase 12: Testing (start)
12. Phase 15: Accessibility

**Sprint 4: Optimization & DevOps (5 days)**
13. Phase 12: Testing (complete)
14. Phase 11: Performance
15. Phase 13: DevOps/CI/CD
16. Phase 14: Code Quality

**Sprint 5: Launch Preparation (4.5 days)**
17. Phase 16: SEO
18. Phase 17: Monitoring
19. Phase 18: Documentation
20. Phase 19: Production Readiness

---

## ✅ Success Criteria

The Better GreenShare will be considered complete when all of the following criteria are met:

### Functional Requirements
- [ ] All original GreenShare features replicated with improved UX
- [ ] End-to-end type safety from database to UI (Prisma → tRPC → React)
- [ ] Authentication works with both credentials and Google OAuth
- [ ] Users can create, edit, delete items
- [ ] Users can make, accept, complete, confirm, and cancel offers
- [ ] Offer state machine enforces correct transitions
- [ ] Image upload works with proper validation (type, size, count)
- [ ] Item search with trigram similarity returns relevant results
- [ ] AI categorization suggests appropriate categories
- [ ] Location autocomplete and validation work

### Technical Requirements
- [ ] All public pages are server-side rendered (SSR)
- [ ] All inputs validated with Zod on both client and server
- [ ] No TypeScript errors in production build
- [ ] No ESLint errors or warnings
- [ ] Test coverage above 80% for server-side code
- [ ] Test coverage above 70% for client-side code
- [ ] All E2E tests passing
- [ ] CI pipeline runs on every PR (lint, typecheck, test, build)
- [ ] Application handles errors gracefully with user-friendly messages
- [ ] No sensitive data exposed in client-side bundles or logs

### Security Requirements
- [ ] Password hashing with bcrypt (cost factor 12+)
- [ ] Rate limiting on auth endpoints
- [ ] CSRF protection via NextAuth (built-in)
- [ ] Session cookies are HttpOnly, Secure, SameSite
- [ ] All user inputs sanitized
- [ ] Resource ownership verified on all mutations
- [ ] Security headers configured
- [ ] No XSS, SQL injection, or CSRF vulnerabilities

### Performance Requirements
- [ ] First Contentful Paint (FCP) < 1.5s
- [ ] Largest Contentful Paint (LCP) < 2.5s
- [ ] Time to Interactive (TTI) < 3.5s
- [ ] Cumulative Layout Shift (CLS) < 0.1
- [ ] Images optimized (Next.js Image, lazy loading)
- [ ] Bundle size < 300KB (initial load)
- [ ] Database queries optimized with indexes
- [ ] Cursor-based pagination for large lists

### Accessibility Requirements
- [ ] All pages pass WCAG 2.1 AA automated audit
- [ ] All interactive elements keyboard accessible
- [ ] All images have meaningful alt text
- [ ] Color contrast ratios meet WCAG AA standards
- [ ] Focus indicators visible on all elements
- [ ] Screen reader friendly (tested with NVDA/VoiceOver)

### SEO Requirements
- [ ] All public pages have unique, descriptive titles and meta descriptions
- [ ] Sitemap generated and accessible at `/sitemap.xml`
- [ ] Robots.txt configured
- [ ] Structured data (JSON-LD) for item listings
- [ ] Open Graph and Twitter Card metadata
- [ ] Canonical URLs configured

### Production Readiness
- [ ] Environment variables documented and configured
- [ ] Database migrations applied successfully
- [ ] Health check endpoint working
- [ ] Error tracking configured (Sentry)
- [ ] Logging configured (Pino)
- [ ] Deployment automated (Vercel or Docker)
- [ ] SSL/TLS certificate configured
- [ ] Domain configured
- [ ] Backups configured
- [ ] Rollback plan documented

---

## 📦 Dependencies to Install

### Production Dependencies

```bash
# Authentication
pnpm add bcryptjs @google/generative-ai

# Email
pnpm add nodemailer

# UI/UX
pnpm add sonner

# Sanitization
pnpm add isomorphic-dompurify

# Logging
pnpm add pino pino-pretty

# Error Tracking
pnpm add @sentry/nextjs

# Rate Limiting (production)
pnpm add @upstash/ratelimit @upstash/redis
```

### Development Dependencies

```bash
# TypeScript types
pnpm add -D @types/bcryptjs @types/nodemailer

# Testing
pnpm add -D vitest @vitest/ui @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom vitest-mock-extended @playwright/test

# Code quality
pnpm add -D husky lint-staged @commitlint/cli @commitlint/config-conventional

# Build tools
pnpm add -D tsx @next/bundle-analyzer
```

---

## 📝 Notes

### Key Architectural Decisions

1. **T3 Stack over Flask + Next.js**
   - **Rationale:** Unified type safety, single deployment, better DX
   - **Tradeoff:** Requires learning tRPC and Prisma

2. **tRPC over REST**
   - **Rationale:** End-to-end type safety, automatic API documentation, better DX
   - **Tradeoff:** Vendor lock-in (harder to use with non-TypeScript clients)

3. **NextAuth.js over custom auth**
   - **Rationale:** Battle-tested, secure, supports multiple providers
   - **Tradeoff:** Less flexibility for custom auth flows

4. **Prisma over raw SQL**
   - **Rationale:** Type-safe queries, migrations, great DX
   - **Tradeoff:** Learning curve, abstraction overhead

5. **React Server Components**
   - **Rationale:** Better performance, SEO, reduced client bundle
   - **Tradeoff:** New mental model, debugging can be harder

### Common Pitfalls to Avoid

- ❌ Using `"use client"` everywhere (kills SSR benefits)
- ❌ Trusting client-side validation without server validation
- ❌ Exposing secrets in `NEXT_PUBLIC_*` env vars
- ❌ Not handling loading and error states
- ❌ Forgetting to verify resource ownership in mutations
- ❌ Using offset-based pagination (slow at scale)
- ❌ Not sanitizing user inputs before storing in database
- ❌ Skipping accessibility testing
- ❌ Not testing the offer state machine thoroughly
- ❌ Deploying without running migrations

### Resources

- [T3 Stack Documentation](https://create.t3.gg/)
- [tRPC Documentation](https://trpc.io/)
- [Prisma Documentation](https://www.prisma.io/docs)
- [NextAuth.js Documentation](https://authjs.dev/)
- [Next.js Documentation](https://nextjs.org/docs)
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)

---

**Last Updated:** 2026-02-08
**Version:** 1.0
**Status:** Ready for Implementation

---

*This implementation plan is a living document and should be updated as the project evolves.*
