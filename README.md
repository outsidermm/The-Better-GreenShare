# GreenShare - Free Stuff, Zero Waste

> **A modern platform for giving away free items and building sustainable communities**

## 🌱 What is GreenShare?

GreenShare is a freebies platform where people can **list items they no longer need** and others can **claim them for free**. No exchanges, no bartering—just pure generosity and sustainability.

Think of it as a digital "free stuff" board for your community, with built-in chat to coordinate pickups.

### Why GreenShare?

- ♻️ **Reduce waste** - Give items a second life instead of throwing them away
- 🤝 **Build community** - Connect with neighbors and help each other out
- 💚 **Zero cost** - Everything is free, no transactions or exchanges
- 📱 **Modern UX** - Clean, fast, accessible interface built with Next.js 15

---

## 🚀 Features

### Phase 1: Items Backend (Current)
- ✅ List free items with photos, description, condition, location
- ✅ Browse items by category (Essentials, Living, Tools/Tech, Style, Leisure)
- ✅ Filter by condition (New, Like New, Used Good, etc.)
- ✅ User authentication with Google OAuth
- ✅ Full-stack type safety (TypeScript + tRPC + Prisma)

### Phase 2: Real-Time Chat (Coming Soon)
- 💬 Direct messaging between users
- ⚡ Real-time chat for coordinating pickup
- 📍 Location sharing after initial contact
- 🔔 Notifications for new messages

---

## 🛠 Tech Stack

**Framework & Runtime**
- Next.js 15.5+ (App Router with React Server Components)
- TypeScript (strict mode)
- Node.js 20+

**API & Database**
- tRPC 11 - Type-safe API layer
- Prisma 7 - PostgreSQL ORM
- PostgreSQL 16 - Database

**Authentication**
- NextAuth.js 5.0 (beta) - Google OAuth

**Styling**
- Tailwind CSS 4.1+

**Package Manager**
- pnpm

---

## 📦 Getting Started

### Prerequisites

- Node.js 20+ ([install](https://nodejs.org/))
- pnpm ([install](https://pnpm.io/installation))
- Docker or Podman (for local PostgreSQL)

### Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/yourusername/greenshare.git
   cd greenshare
   ```

2. **Install dependencies**
   ```bash
   pnpm install
   ```

3. **Set up environment variables**
   ```bash
   cp .env.example .env
   ```

   Edit `.env` and fill in:
   - `AUTH_SECRET` - Generate with `npx auth secret`
   - `AUTH_GOOGLE_ID` - From Google Cloud Console
   - `AUTH_GOOGLE_SECRET` - From Google Cloud Console
   - `DATABASE_URL` - PostgreSQL connection string (see next step)

4. **Start local PostgreSQL database**
   ```bash
   ./start-database.sh
   ```

5. **Push database schema**
   ```bash
   pnpm db:push
   ```

6. **Run development server**
   ```bash
   pnpm dev
   ```

7. **Open in browser**
   ```
   http://localhost:3000
   ```

---

## 📜 Available Scripts

### Development
```bash
pnpm dev          # Start dev server (http://localhost:3000)
pnpm build        # Build for production
pnpm start        # Start production server
pnpm preview      # Build and start production server
```

### Database
```bash
pnpm db:push      # Push schema changes (dev only)
pnpm db:generate  # Create migration + generate Prisma Client
pnpm db:migrate   # Deploy migrations (production)
pnpm db:studio    # Open Prisma Studio (database GUI)
```

### Code Quality
```bash
pnpm check        # Run linter + type checking
pnpm lint         # Run ESLint
pnpm lint:fix     # Fix ESLint issues
pnpm typecheck    # Run TypeScript compiler
pnpm format:check # Check code formatting
pnpm format:write # Format code with Prettier
```

---

## 📁 Project Structure

```
greenshare/
├── prisma/
│   ├── schema.prisma          # Database schema
│   └── migrations/            # Database migrations
├── src/
│   ├── app/                   # Next.js App Router
│   │   ├── _components/       # Client components
│   │   ├── api/               # API routes (tRPC handler, NextAuth)
│   │   ├── layout.tsx         # Root layout
│   │   └── page.tsx           # Home page
│   ├── server/
│   │   ├── api/
│   │   │   ├── routers/       # tRPC routers
│   │   │   ├── root.ts        # Main tRPC router
│   │   │   └── trpc.ts        # tRPC setup
│   │   ├── auth/              # NextAuth config
│   │   └── db.ts              # Prisma client
│   ├── trpc/
│   │   ├── react.tsx          # Client-side tRPC
│   │   └── server.ts          # Server-side tRPC (RSC)
│   ├── env.js                 # Environment variable validation
│   └── styles/                # Global styles
├── .env                       # Environment variables (gitignored)
├── .env.example               # Environment template
└── package.json
```

---

## 🗄️ Database Schema

### Current Models (Phase 1)

**Item** - Free items listed by users
- `id`, `userId`, `title`, `description`
- `condition` - NEW | LIKE_NEW | USED_GOOD | USED_FAIR | POOR
- `status` - AVAILABLE | CLAIMED | DELETED
- `category` - ESSENTIALS | LIVING | TOOLS_TECH | STYLE_EXPRESSION | LEISURE_LEARNING
- `location` - User-provided location string
- `images[]` - Array of image URLs
- `createdAt`, `updatedAt`

**User** - User accounts (via NextAuth)
- `id`, `email`, `name`, `firstName`, `lastName`
- `emailVerified`, `image`
- Relations: `items[]`

### Coming in Phase 2

**Conversation** - Chat between users
- `id`, `participantIds[]`, `lastMessageAt`

**Message** - Chat messages
- `id`, `conversationId`, `senderId`, `content`, `sentAt`

---

## 🔒 Environment Variables

See `.env.example` for full list. Key variables:

| Variable | Description | Required |
|----------|-------------|----------|
| `DATABASE_URL` | PostgreSQL connection string | ✅ |
| `AUTH_SECRET` | NextAuth.js secret (generate with `npx auth secret`) | ✅ |
| `AUTH_GOOGLE_ID` | Google OAuth client ID | ✅ |
| `AUTH_GOOGLE_SECRET` | Google OAuth client secret | ✅ |
| `NODE_ENV` | `development` or `production` | Auto-set |

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes using conventional commits:
   - `features: add item filtering by category`
   - `fix: resolve image upload bug`
   - `refactor: simplify item card component`
   - `chores: update dependencies`
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## 🎯 Roadmap

- [x] **Phase 1: Items Backend** - Database schema, authentication, item CRUD
- [ ] **Phase 2: Real-Time Chat** - User messaging, pickup coordination
- [ ] **Phase 3: Frontend Polish** - Image upload, search, filters, responsive design
- [ ] **Phase 4: Production** - Deployment, monitoring, analytics

---

## 📄 License

MIT

---

## 🙏 Acknowledgments

Built with the [T3 Stack](https://create.t3.gg/) - the best way to start a full-stack, type-safe Next.js app.

**Tech:** Next.js • tRPC • Prisma • NextAuth.js • Tailwind CSS • TypeScript

---

**Questions or feedback?** Open an issue or start a discussion!
