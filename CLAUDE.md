# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**GreenShare** is a freebies platform for giving away free items and building sustainable communities. Users list items they no longer need, others claim them, and coordinate pickup through chat.

**Key Philosophy:** No exchanges, no bartering—just free stuff and generosity.

## Technology Stack

- **Framework**: Next.js 15.5+ (App Router with React Server Components)
- **Language**: TypeScript (strict mode enabled)
- **API**: tRPC 11+ for end-to-end type-safe APIs
- **Database**: PostgreSQL with Prisma 7 ORM
- **Auth**: NextAuth.js 5.0 (beta) with Google OAuth
- **Styling**: Tailwind CSS 4.1+
- **State Management**: TanStack Query (React Query) 5+
- **Package Manager**: pnpm

## Development Commands

### Setup & Installation
```bash
pnpm install                    # Install dependencies
./start-database.sh             # Start local PostgreSQL container
pnpm db:push                    # Push schema to database (development)
```

### Development
```bash
pnpm dev                        # Start dev server (http://localhost:3000)
pnpm db:studio                  # Open Prisma Studio for database GUI
```

### Database Operations
```bash
pnpm db:generate                # Generate Prisma Client and create migration
pnpm db:migrate                 # Deploy migrations to production database
pnpm db:push                    # Push schema changes without migration (dev only)
pnpm db:studio                  # Open Prisma Studio
```

### Code Quality
```bash
pnpm check                      # Run linter + type checking
pnpm lint                       # Run ESLint
pnpm lint:fix                   # Fix ESLint issues automatically
pnpm typecheck                  # Run TypeScript compiler in check mode
pnpm format:check               # Check code formatting with Prettier
pnpm format:write               # Format code with Prettier
```

### Build & Deploy
```bash
pnpm build                      # Build for production
pnpm start                      # Start production server
pnpm preview                    # Build and start production server
```

## Project Structure

### Key Directories

- **`src/app/`**: Next.js App Router pages and layouts
  - `src/app/api/`: API routes (tRPC handler, NextAuth)
  - `src/app/_components/`: Page-specific client components
- **`src/server/`**: Server-side code
  - `src/server/api/routers/`: tRPC router definitions
  - `src/server/api/trpc.ts`: tRPC context, middleware, and procedures
  - `src/server/auth/`: NextAuth.js configuration
  - `src/server/db.ts`: Prisma client singleton
- **`src/trpc/`**: tRPC client setup
  - `src/trpc/react.tsx`: Client-side tRPC provider
  - `src/trpc/server.ts`: Server-side tRPC caller for React Server Components
- **`prisma/`**: Database schema and migrations
- **`generated/prisma/`**: Generated Prisma Client (gitignored)

### Critical Files

- **`src/env.js`**: Environment variable validation using `@t3-oss/env-nextjs` - update when adding new env vars
- **`prisma/schema.prisma`**: Database schema definition
- **`prisma.config.ts`**: Prisma 7 configuration file
- **`src/server/api/root.ts`**: Main tRPC router that combines all sub-routers

## Architecture Patterns

### tRPC Request Flow

1. **Client-side (RSC)**: `src/trpc/server.ts` → Creates server caller with context
2. **Client-side (Client Component)**: `src/trpc/react.tsx` → HTTP request to `/api/trpc`
3. **API Route**: `src/app/api/trpc/[trpc]/route.ts` → Handles tRPC requests
4. **Router**: `src/server/api/root.ts` → Routes to specific router (e.g., `itemRouter`)
5. **Procedure**: Individual query/mutation in `src/server/api/routers/*.ts`

### Database Access

- Prisma Client is imported from `generated/prisma/client` (custom output path)
- Uses `@prisma/adapter-pg` for PostgreSQL connection pooling
- Singleton pattern in `src/server/db.ts` prevents multiple instances
- Always access via `db` import from `@/server/db`

### Authentication

- NextAuth.js config in `src/server/auth/config.ts`
- Google OAuth provider configured (requires `AUTH_GOOGLE_ID` and `AUTH_GOOGLE_SECRET`)
- Session accessible in tRPC context via `ctx.session`
- Use `protectedProcedure` for authenticated endpoints (enforces non-null user)
- Use `publicProcedure` for unauthenticated endpoints (session may be null)

### Environment Variables

All environment variables must be:
1. Defined in `.env` (gitignored)
2. Added to `.env.example` template
3. Validated in `src/env.js` with Zod schema
4. Server vars in `server` object, client vars in `client` object with `NEXT_PUBLIC_` prefix

## Important Conventions

### TypeScript Configuration

- Strict mode enabled with `noUncheckedIndexedAccess`
- Path alias: `@/*` maps to `src/*`
- ESM module system (`"type": "module"` in package.json)
- `checkJs` is enabled - JavaScript files are type-checked

### tRPC Procedures

- Development timing middleware adds 100-400ms artificial delay
- SuperJSON transformer enables serialization of Dates, Maps, Sets, etc.
- `publicProcedure`: Available to all users (session may be null)
- `protectedProcedure`: Requires authentication (throws UNAUTHORIZED if not logged in)

### Prisma Schema

- Custom output path: `generated/prisma` (not default `node_modules/.prisma/client`)
- Models use NextAuth.js adapter schema (Account, Session, User, VerificationToken)
- PostgreSQL-specific features available (arrays, etc.)
- After schema changes: run `pnpm db:push` (dev) or `pnpm db:generate` (migration)

### Adding New tRPC Routers

1. Create router file in `src/server/api/routers/[name].ts`
2. Export a router using `createTRPCRouter`
3. Import and add to `appRouter` in `src/server/api/root.ts`
4. Type safety is automatic - no manual type exports needed

### Database Development Workflow

- **Development**: Use `pnpm db:push` for quick schema iterations (no migration files)
- **Production/Team**: Use `pnpm db:generate` to create migration files
- Always commit migration files in `prisma/migrations/`
- Prisma Client regenerates automatically on `pnpm install` (postinstall hook)

## Database Schema (Phase 1)

### Enums

```prisma
enum ItemCondition {
  NEW
  LIKE_NEW
  USED_GOOD
  USED_FAIR
  POOR
}

enum ItemStatus {
  AVAILABLE    // Item is available for claiming
  CLAIMED      // Item has been claimed
  DELETED      // Item was deleted by owner
}

enum ItemCategory {
  ESSENTIALS          // Food, toiletries, basic necessities
  LIVING              // Furniture, home goods, appliances
  TOOLS_TECH          // Tools, gadgets, electronics
  STYLE_EXPRESSION    // Clothing, accessories, art
  LEISURE_LEARNING    // Books, games, sports equipment
}
```

### Models

**Item** - Free items listed by users
- All items are FREE (no "exchange" type)
- Users list items, others claim them
- Status tracks: AVAILABLE → CLAIMED or DELETED

**User** - User accounts via NextAuth
- Extended with `firstName` and `lastName`
- Relation to items they've listed

## Phase 1 Scope

### Completed
✅ Database schema with Item and User models
✅ Prisma setup with custom output path
✅ NextAuth.js with Google OAuth (replacing Discord)
✅ Type-safe environment variable validation
✅ tRPC foundation with example router

### Next Steps (Phase 2)
- [ ] Real-time chat system (Conversation + Message models)
- [ ] User-to-user messaging
- [ ] Pickup coordination
- [ ] Notifications

## Commit Conventions

Use conventional commits with these prefixes:
- `features:` - New features or capabilities
- `fix:` - Bug fixes
- `refactor:` - Code restructuring without behavior change
- `chores:` - Maintenance tasks (deps, config, etc.)

**Examples:**
```
features: add item listing API endpoint
fix: resolve authentication redirect loop
refactor: simplify item card component
chores: update Next.js to 15.5.1
```

## Environment Setup

Required environment variables (see `.env.example`):
- `DATABASE_URL`: PostgreSQL connection string
- `AUTH_SECRET`: NextAuth.js secret (generate with `npx auth secret`)
- `AUTH_GOOGLE_ID`: Google OAuth application ID
- `AUTH_GOOGLE_SECRET`: Google OAuth application secret

Use `./start-database.sh` to spin up a local PostgreSQL Docker/Podman container automatically configured from your `DATABASE_URL`.
