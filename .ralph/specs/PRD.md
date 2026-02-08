# PRD: Enhanced Bartering Flow for GreenShare

## 1. Overview

This project enhances GreenShare's item exchange system to support flexible multi-item-to-multi-item bartering with formal negotiation states. Currently, the system only supports one-to-one exchanges (user offers multiple items for ONE item). The new system will support both directed offers (User A offers to specific User B) and broadcast offers (User A posts items available for trade, anyone can respond), with structured counter-offers, negotiation history, and integrated chat for coordination.

**Core Philosophy**: More structured than Facebook Marketplace's informal messaging, less rigid than eBay's fixed-price listings. Think "contract negotiation" with clear offer/counter-offer/acceptance states, but flexible item bundles on both sides.

## 2. Target Users

| User Type | Description | Primary Need |
|-----------|-------------|--------------|
| Active Trader | User who frequently exchanges items, manages multiple concurrent negotiations | Clear view of offer states, negotiation history, ability to counter-offer efficiently |
| Casual User | Occasional trader listing a few items, responding to offers | Simple flow to accept/reject offers, easy chat to arrange pickup |
| Browser | User exploring available items, making opportunistic trades | Discover broadcast offers, send interest, start negotiations |

## 3. User Stories

### Epic: Multi-Item Offers

- **US-001**: As a user, I want to offer multiple of my items for multiple items from another user, so that I can negotiate fair bundle trades
  - Acceptance Criteria:
    - [ ] Can select multiple items from my inventory to offer
    - [ ] Can select multiple items from another user's inventory (or leave open for them to choose)
    - [ ] Offer record stores both offered and requested item ID arrays
    - [ ] UI shows all items in the bundle clearly on both sides

- **US-002**: As a user receiving an offer, I want to see all items being offered and requested in one view, so that I can evaluate the trade at a glance
  - Acceptance Criteria:
    - [ ] Offer detail page shows offered items (with images, titles, conditions)
    - [ ] Shows requested items (if directed) or "open request" indicator
    - [ ] Displays offer message/context from sender
    - [ ] Clear accept/reject/counter-offer action buttons

### Epic: Broadcast Offers

- **US-003**: As a user, I want to post a "broadcast offer" listing items I have available for trade without targeting a specific user, so that anyone interested can respond
  - Acceptance Criteria:
    - [ ] Can create an offer with `targetUserId = null` (broadcast mode)
    - [ ] Broadcast offers appear in a public feed/discovery page
    - [ ] Other users can view broadcast offers and see "send interest" option
    - [ ] Broadcast offers are clearly distinguished from directed offers in UI

- **US-004**: As a user browsing broadcast offers, I want to send interest in someone's items, so that we can start a negotiation
  - Acceptance Criteria:
    - [ ] Can click "interested" on a broadcast offer
    - [ ] Creates an Interest record linking me to the broadcast offer
    - [ ] Broadcast offer owner sees list of interested users
    - [ ] Can start a negotiation from an interest (creates a directed offer/chat)

### Epic: Negotiation Flow

- **US-005**: As a user who receives an offer, I want to make a counter-offer with different items, so that we can negotiate toward a mutually beneficial trade
  - Acceptance Criteria:
    - [ ] From offer detail page, can click "counter-offer"
    - [ ] Counter-offer form pre-fills with current offer context
    - [ ] Can modify offered items list, requested items list, and message
    - [ ] Counter-offer creates a NEW offer record linked to original via `parentOfferId`
    - [ ] Original offer status remains visible in history (not deleted)

- **US-006**: As a user involved in a negotiation, I want to see the full offer history (initial offer, counters, who made each), so that I understand how we got to the current state
  - Acceptance Criteria:
    - [ ] Negotiation view shows all offers in thread (parent → counter → counter)
    - [ ] Each offer shows: who sent, items offered, items requested, timestamp, status
    - [ ] Current/active offer is clearly highlighted
    - [ ] Can collapse/expand historical offers

- **US-007**: As a user, I want both parties to explicitly accept the final offer before the trade is considered agreed, so that there's no ambiguity
  - Acceptance Criteria:
    - [ ] When User A accepts User B's offer, offer status becomes "ACCEPTED"
    - [ ] Accepted offer is locked (cannot be modified/countered)
    - [ ] Both users can see trade is agreed and move to completion phase

### Epic: Chat Integration

- **US-008**: As a user negotiating a trade, I want to chat with the other party within the offer context, so that we can discuss details and arrange pickup
  - Acceptance Criteria:
    - [ ] Each offer has an associated chat thread (Conversation)
    - [ ] Chat is accessible from offer detail page
    - [ ] Messages in chat show timestamp and sender
    - [ ] Real-time or near-real-time message delivery (polling acceptable for MVP)

- **US-009**: As a user, I want to also message other users outside of specific offers, so that I can ask general questions or build rapport
  - Acceptance Criteria:
    - [ ] Can initiate a direct message to any user (not tied to an offer)
    - [ ] User-to-user conversations are separate from offer-specific chats
    - [ ] Conversation list shows both offer chats and direct chats clearly labeled

### Epic: Trade Completion

- **US-010**: As a user with an accepted offer, I want to mark the exchange as complete after we meet up and swap items, so that the trade is finalized
  - Acceptance Criteria:
    - [ ] Accepted offers show "complete trade" button
    - [ ] Clicking complete updates offer status to "COMPLETED"
    - [ ] Involved items' status changes from AVAILABLE to EXCHANGED
    - [ ] Completion is reflected in both users' trade history

- **US-011**: As a user, I want to cancel an offer (or accepted trade) if circumstances change, so that I'm not locked into something I can't fulfill
  - Acceptance Criteria:
    - [ ] Can cancel offer at any status (PENDING, ACCEPTED)
    - [ ] Must provide cancellation reason (text field)
    - [ ] Cancellation notifies other party
    - [ ] Cancelled offers show reason in history

### Epic: Item Availability Management

- **US-012**: As a user whose item is in an active offer, I want the system to handle item deletion gracefully, so that I understand the impact
  - Acceptance Criteria:
    - [ ] If item in active offer (PENDING/ACCEPTED) is deleted, item status becomes DELETED (soft delete)
    - [ ] Offer still shows item but marked as "no longer available"
    - [ ] User can remove unavailable item from offer or cancel offer
    - [ ] Completed offers preserve item details even if item deleted later

## 4. Technical Requirements

### Stack
| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Framework | Next.js 15.5 (App Router) | Already in use, RSC support |
| Database | PostgreSQL + Prisma 7 | Already in use, strong relational model needed |
| API | tRPC 11 | Already in use, end-to-end type safety |
| Auth | NextAuth.js 5.0 | Already in use, session management |
| Real-time | Polling (MVP) / Pusher or Ably (future) | Chat delivery mechanism |

### Architecture

**Core Principle**: Flatten the offer structure. Remove `OfferedItem` junction table, use arrays of item IDs directly on the `Offer` model.

**Key Decisions**:
1. **Offer as negotiation thread**: Each counter-offer is a new Offer record linked via `parentOfferId`, creating a linked list of negotiation states
2. **Broadcast via nullable target**: `targetUserId` can be null (broadcast) or set (directed)
3. **Chat dual-mode**: Conversations can be offer-specific (`offerId` set) or user-to-user (`offerId` null)
4. **Soft delete items**: Items in offers become DELETED status, not hard-deleted, preserving referential integrity
5. **Item arrays**: `offeredItemIds` and `requestedItemIds` as String[] on Offer (Prisma supports array types on PostgreSQL)

### Data Model

| Entity | Key Fields | Relationships |
|--------|-----------|---------------|
| User | id, email, name, firstName, lastName | has_many: items, sentOffers, receivedOffers, conversations, messages, interests |
| Item | id, userId, title, description, condition, status, category, type, images (String[]) | belongs_to: user; appears_in: offers via ID arrays |
| Offer | id, initiatorId, targetUserId (nullable), offeredItemIds (String[]), requestedItemIds (String[]), message, status, parentOfferId (nullable), cancellationReason | belongs_to: initiator (User), target (User), parentOffer; has_many: counterOffers, conversation, interests |
| Conversation | id, offerId (nullable), participantIds (String[]), lastMessageAt | belongs_to: offer (optional); has_many: messages |
| Message | id, conversationId, senderId, content, sentAt | belongs_to: conversation, sender (User) |
| Interest | id, offerId, userId, createdAt | belongs_to: offer (must be broadcast), user |

**Removed Models**: `OfferedItem`, `ItemImage` (images now String[] on Item)

**Key Enums**:
- `OfferStatus`: PENDING, ACCEPTED, COMPLETED, CANCELLED
- `ItemStatus`: AVAILABLE, EXCHANGED, DELETED (added DELETED)
- `ItemCondition`, `ItemCategory`, `ItemType`: unchanged

### API Endpoints / Procedures (tRPC)

**Offers Router** (`src/server/api/routers/offers.ts`):
| Procedure | Purpose | Auth | Input |
|-----------|---------|------|-------|
| create | Create new offer (directed or broadcast) | Protected | { targetUserId?, offeredItemIds, requestedItemIds, message } |
| getById | Get offer details with items and history | Protected | { offerId } |
| listMy | List offers I initiated or received | Protected | { filter?: 'sent'\|'received'\|'all' } |
| listBroadcast | List public broadcast offers | Protected | { filters? } |
| counterOffer | Create counter-offer linked to parent | Protected | { parentOfferId, offeredItemIds, requestedItemIds, message } |
| accept | Accept an offer | Protected | { offerId } |
| complete | Mark accepted offer as completed | Protected | { offerId } |
| cancel | Cancel offer with reason | Protected | { offerId, reason } |

**Interests Router** (`src/server/api/routers/interests.ts`):
| Procedure | Purpose | Auth | Input |
|-----------|---------|------|-------|
| create | Send interest in broadcast offer | Protected | { offerId } |
| listForOffer | List users interested in my broadcast offer | Protected | { offerId } |

**Chat Router** (`src/server/api/routers/chat.ts`):
| Procedure | Purpose | Auth | Input |
|-----------|---------|------|-------|
| getOrCreateConversation | Get conversation for offer or between users | Protected | { offerId?, otherUserId? } |
| sendMessage | Send message in conversation | Protected | { conversationId, content } |
| listMessages | Get messages in conversation | Protected | { conversationId, limit?, cursor? } |
| listConversations | List user's conversations | Protected | {} |

**Items Router** (enhance existing `src/server/api/routers/post.ts`):
| Procedure | Purpose | Auth | Input |
|-----------|---------|------|-------|
| delete | Soft-delete item (set status DELETED) | Protected | { itemId } |
| checkAvailability | Check if items are available for offer | Protected | { itemIds } |

## 5. Screens & Navigation

### Screen Map
```
App
├── Auth Stack (unauthenticated)
│   ├── Welcome
│   ├── Sign Up
│   ├── Login
│   └── Forgot Password
└── Main Tabs (authenticated)
    ├── Home / Browse Items
    ├── My Items
    ├── Offers (new)
    │   ├── Sent Offers
    │   ├── Received Offers
    │   ├── Offer Detail
    │   │   ├── Offer History (if countered)
    │   │   ├── Chat (embedded)
    │   │   └── Actions (Accept/Counter/Cancel/Complete)
    │   └── Create Offer (directed or broadcast)
    ├── Broadcast Feed (new)
    │   ├── Browse Broadcast Offers
    │   └── Send Interest
    ├── Chat (new)
    │   ├── Conversation List
    │   └── Conversation Detail
    └── Profile / Settings
```

### Screen Descriptions
| Screen | Purpose | Key Components |
|--------|---------|----------------|
| Offers List | View sent/received offers, filter by status | Tabs (sent/received), offer cards (items preview, status badge, other user) |
| Offer Detail | View offer details, negotiation history, take actions | Item grids (offered/requested), message, action buttons, embedded chat, history timeline |
| Create Offer | Form to create directed or broadcast offer | Target user selector (optional), my items multi-select, requested items multi-select, message field |
| Broadcast Feed | Browse public broadcast offers | Card grid, filters (category, condition), "interested" button |
| Chat List | View all conversations (offer-specific and direct) | Conversation cards (other user, last message, timestamp, badge if offer-related) |
| Conversation | Message thread | Message bubbles, input field, offer context header (if applicable) |

## 6. Non-Functional Requirements

### Performance
- Offer detail page load < 1s
- Chat message send/receive < 2s (polling interval)
- Broadcast feed initial load < 2s

### Security
- Users can only modify/cancel their own offers or offers directed at them
- Offer acceptance requires both parties' consent (initiator creates, target accepts)
- Chat messages only visible to participants
- RLS policies on all tables (Prisma middleware for now, native RLS future)

### Accessibility
- WCAG AA compliance target
- Screen reader support for offer states and chat
- Keyboard navigation for offer actions

### Offline Support
- MVP: requires connection (polling-based chat)
- Future: offline message queuing, optimistic UI updates

## 7. MVP Scope

### In Scope (MVP)
- Multi-item offers (directed and broadcast)
- Counter-offer negotiation flow
- Offer history (parent/counter chain)
- Offer-specific chat
- User-to-user direct chat
- Broadcast offer feed with interest system
- Offer states: PENDING, ACCEPTED, COMPLETED, CANCELLED
- Soft delete for items in active offers

### Out of Scope (Post-MVP)
- Real-time chat (use polling for MVP)
- Push notifications for new offers/messages
- Offer expiration (auto-cancel after X days)
- Image upload improvements (currently just URLs)
- Advanced search/filters on broadcast feed
- Rating/review system for completed trades
- Delivery/shipping coordination (in-person only for MVP)

## 8. Success Metrics
| Metric | Target | How Measured |
|--------|--------|--------------|
| Successful multi-item trades | 10+ in first month | Count COMPLETED offers with 2+ items on each side |
| Counter-offers per negotiation | 1.5 avg | Count offers with parentOfferId / total threads |
| Chat messages per trade | 5+ avg | Count messages in offer conversations |
| Broadcast offer engagement | 20%+ conversion (interest → accepted offer) | Interests created / broadcast offers |

## 9. Open Questions
- Should we add offer expiration (auto-cancel after 7 days of inactivity)?
- Image handling: keep as String[] or migrate to separate table later?
- Real-time chat: when to add (user feedback dependent)?
- Notifications: in-app only or email/push?
