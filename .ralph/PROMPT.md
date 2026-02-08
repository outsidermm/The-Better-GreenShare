# Project: GreenShare Bartering Flow Enhancement

## Your Mission
You are working autonomously in a Ralph loop. Each iteration:
1. Read `.ralph/specs/PRD.md` for full requirements context
2. Read `.ralph/fix_plan.md` for the current priority list
3. Implement the HIGHEST PRIORITY unchecked item
4. Run tests after implementation (if applicable)
5. Update fix_plan.md (check off completed items, note any verify: criteria met)
6. Output a RALPH_STATUS block (REQUIRED)

## Project Context

This is a Next.js 15.5+ application using the T3 Stack (TypeScript, tRPC 11, Prisma 7, NextAuth.js 5.0, Tailwind CSS). The project enhances GreenShare's item exchange system to support multi-item-to-multi-item bartering with formal negotiation states, broadcast offers, counter-offers, and integrated chat.

**Technology Stack:**
- Framework: Next.js 15.5 (App Router with React Server Components)
- Database: PostgreSQL with Prisma 7 ORM (custom output path: `generated/prisma/client`)
- API: tRPC 11 for type-safe procedures
- Auth: NextAuth.js 5.0 (beta)
- Styling: Tailwind CSS 4.1+
- Package Manager: pnpm

**Current Architecture:**
- Client-side tRPC: `src/trpc/react.tsx`
- Server-side tRPC: `src/trpc/server.ts` (for RSC)
- API route handler: `src/app/api/trpc/[trpc]/route.ts`
- Routers: `src/server/api/routers/` (combined in `src/server/api/root.ts`)
- Database: `src/server/db.ts` (Prisma client singleton)
- Prisma schema: `prisma/schema.prisma`

## Specifications
- Full PRD: `.ralph/specs/PRD.md`
- Task plan: `.ralph/fix_plan.md`

## Key Architecture Rules

1. **Prisma Schema Changes:**
   - Custom output path is `generated/prisma` (not default `node_modules/.prisma/client`)
   - After schema changes, run `pnpm db:push` (dev) or `pnpm db:generate` (migration)
   - Always import from `@prisma/client` which resolves to the custom output path

2. **tRPC Procedures:**
   - Use `createTRPCRouter` and `publicProcedure` / `protectedProcedure`
   - `protectedProcedure` enforces authentication (session required)
   - Input validation with Zod schemas
   - New routers must be added to `src/server/api/root.ts`

3. **Database Access:**
   - Always import `db` from `@/server/db`
   - Never create new Prisma client instances
   - Use transactions for multi-step operations

4. **App Router (Next.js 15):**
   - Server Components by default (no 'use client' unless needed)
   - Use `src/trpc/server.ts` for RSC data fetching
   - Use `src/trpc/react.tsx` for client components with mutations

5. **Environment Variables:**
   - All env vars must be in `.env` AND `.env.example` AND validated in `src/env.js`

## Build & Test Instructions

See `.ralph/AGENT.md` for how to build, test, and run the project.

## Rules

- ONE task per loop iteration (stay focused)
- Always run relevant checks after changes (`pnpm typecheck`, `pnpm lint`)
- Never skip the RALPH_STATUS block
- If blocked, set STATUS: BLOCKED and explain why
- If all tasks are done, set EXIT_SIGNAL: true
- Reference the PRD for acceptance criteria — don't guess
- When implementing a user story, check ALL its acceptance criteria
- Do NOT create frontend components until the backend API is fully implemented and tested
- Do NOT skip database schema changes - they are prerequisites for everything else

## Required Output Format

At the END of every response, output EXACTLY:

---RALPH_STATUS---
STATUS: IN_PROGRESS | COMPLETE | BLOCKED
TASKS_COMPLETED_THIS_LOOP: <number>
FILES_MODIFIED: <number>
TESTS_STATUS: PASSING | FAILING | NOT_RUN
WORK_TYPE: IMPLEMENTATION | TESTING | DOCUMENTATION | REFACTORING
EXIT_SIGNAL: false
RECOMMENDATION: <one line summary of what was done and what's next>
---END_RALPH_STATUS---

Set EXIT_SIGNAL: true ONLY when ALL tasks in fix_plan.md are complete
AND the Verification Gate at the bottom of fix_plan.md passes.
