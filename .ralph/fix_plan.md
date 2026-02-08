# Fix Plan: Enhanced Bartering Flow
Derived from: .ralph/specs/PRD.md

## Critical (Data Model & Infrastructure)

- [ ] **Task 1.1**: Update Prisma schema - Add DELETED to ItemStatus enum, add images String[] to Item model, remove ItemImage table
  - Verify: `prisma/schema.prisma` has ItemStatus.DELETED
  - Verify: Item model has `images String[]` field
  - Verify: ItemImage model removed, all relations cleaned up

- [ ] **Task 1.2**: Update Prisma schema - Replace ExchangeOffer and OfferedItem with new Offer model
  - Verify: New Offer model has: initiatorId, targetUserId (nullable), offeredItemIds String[], requestedItemIds String[], message, status, parentOfferId (nullable), cancellationReason
  - Verify: OfferStatus enum updated to: PENDING, ACCEPTED, COMPLETED, CANCELLED
  - Verify: ExchangeOffer and OfferedItem models removed

- [ ] **Task 1.3**: Create Conversation and Message models in Prisma schema
  - Verify: Conversation model with id, offerId (nullable), participantIds String[], lastMessageAt
  - Verify: Message model with id, conversationId, senderId, content, sentAt
  - Verify: Relations set up (User has conversations/messages, Conversation has messages)

- [ ] **Task 1.4**: Create Interest model in Prisma schema
  - Verify: Interest model with id, offerId, userId, createdAt
  - Verify: Relations set up (User has interests, Offer has interests)

- [ ] **Task 1.5**: Run database migration and generate Prisma client
  - Verify: `pnpm db:generate` runs successfully
  - Verify: New types available in generated Prisma client
  - Verify: No TypeScript errors in `src/server/db.ts`

## High (Core API - Offers)

- [ ] **Task 2.1**: Create offers router - setup and types (US-001, US-002)
  - Verify: `src/server/api/routers/offers.ts` created
  - Verify: Input/output Zod schemas defined for all procedures
  - Verify: Router exported and added to `src/server/api/root.ts`

- [ ] **Task 2.2**: Implement offers.create procedure (US-001)
  - Verify: Accepts targetUserId (nullable), offeredItemIds, requestedItemIds, message
  - Verify: Creates Offer record with status PENDING
  - Verify: Returns offer with related item data populated

- [ ] **Task 2.3**: Implement offers.getById procedure (US-002, US-006)
  - Verify: Fetches offer with initiator, target, items (offered and requested)
  - Verify: If offer has parentOfferId, recursively loads parent chain
  - Verify: Returns full negotiation history

- [ ] **Task 2.4**: Implement offers.listMy procedure
  - Verify: Filter by 'sent' (initiatorId = currentUser) or 'received' (targetUserId = currentUser)
  - Verify: Returns offers with item previews and other party info
  - Verify: Orders by updatedAt DESC

- [ ] **Task 2.5**: Implement offers.counterOffer procedure (US-005)
  - Verify: Accepts parentOfferId, new offeredItemIds, requestedItemIds, message
  - Verify: Creates new Offer with parentOfferId set, swaps initiator/target
  - Verify: Parent offer remains PENDING (not modified)

- [ ] **Task 2.6**: Implement offers.accept procedure (US-007)
  - Verify: Updates offer status to ACCEPTED
  - Verify: Only target user (recipient) can accept
  - Verify: Cannot accept already ACCEPTED/COMPLETED/CANCELLED offers

- [ ] **Task 2.7**: Implement offers.complete procedure (US-010)
  - Verify: Updates offer status to COMPLETED
  - Verify: Updates all involved items' status to EXCHANGED
  - Verify: Only works on ACCEPTED offers
  - Verify: Either party can mark complete

- [ ] **Task 2.8**: Implement offers.cancel procedure (US-011)
  - Verify: Updates offer status to CANCELLED
  - Verify: Stores cancellationReason
  - Verify: Either party can cancel

## High (Broadcast Offers & Interests)

- [ ] **Task 3.1**: Implement offers.listBroadcast procedure (US-003)
  - Verify: Filters for offers where targetUserId IS NULL
  - Verify: Only returns PENDING broadcasts (not accepted/completed)
  - Verify: Returns with initiator info and item details
  - Verify: Supports optional category/condition filters

- [ ] **Task 3.2**: Create interests router (US-004)
  - Verify: `src/server/api/routers/interests.ts` created
  - Verify: Schemas and types defined
  - Verify: Exported and added to root router

- [ ] **Task 3.3**: Implement interests.create procedure (US-004)
  - Verify: Accepts offerId
  - Verify: Validates offer is a broadcast (targetUserId IS NULL)
  - Verify: Creates Interest record (unique per user per offer)

- [ ] **Task 3.4**: Implement interests.listForOffer procedure (US-004)
  - Verify: Accepts offerId
  - Verify: Returns list of users who sent interest
  - Verify: Only offer owner can see interests

## High (Chat System)

- [ ] **Task 4.1**: Create chat router (US-008, US-009)
  - Verify: `src/server/api/routers/chat.ts` created
  - Verify: Schemas for conversations and messages
  - Verify: Exported and added to root router

- [ ] **Task 4.2**: Implement chat.getOrCreateConversation procedure (US-008, US-009)
  - Verify: If offerId provided, finds/creates conversation for that offer
  - Verify: If otherUserId provided, finds/creates direct user-to-user conversation
  - Verify: Returns conversation with participant info

- [ ] **Task 4.3**: Implement chat.sendMessage procedure (US-008, US-009)
  - Verify: Accepts conversationId, content
  - Verify: Creates Message record
  - Verify: Updates conversation.lastMessageAt
  - Verify: Returns created message

- [ ] **Task 4.4**: Implement chat.listMessages procedure (US-008, US-009)
  - Verify: Accepts conversationId, optional limit/cursor for pagination
  - Verify: Returns messages ordered by sentAt DESC
  - Verify: Only conversation participants can read messages

- [ ] **Task 4.5**: Implement chat.listConversations procedure (US-008, US-009)
  - Verify: Returns all conversations for current user (participantIds contains user.id)
  - Verify: Includes last message preview, other participant(s) info, offer info if applicable
  - Verify: Orders by lastMessageAt DESC

## High (Item Availability)

- [ ] **Task 5.1**: Update items router - soft delete (US-012)
  - Verify: Existing delete procedure updated to set status = DELETED instead of hard delete
  - Verify: Returns updated item

- [ ] **Task 5.2**: Implement items.checkAvailability procedure (US-012)
  - Verify: Accepts itemIds array
  - Verify: Returns availability status for each item (AVAILABLE, EXCHANGED, DELETED)
  - Verify: Used by offer creation to validate items

## Medium (Frontend - Offers UI)

- [ ] **Task 6.1**: Create Offers page layout and tabs (sent/received)
  - Verify: `/app/offers/page.tsx` created
  - Verify: Tabs to switch between sent and received offers
  - Verify: Calls `api.offers.listMy` with appropriate filter

- [ ] **Task 6.2**: Create Offer card component
  - Verify: Displays item thumbnails (offered and requested sides)
  - Verify: Shows status badge (PENDING, ACCEPTED, etc.)
  - Verify: Shows other party name and timestamp
  - Verify: Links to offer detail page

- [ ] **Task 6.3**: Create Offer detail page (US-002, US-006)
  - Verify: `/app/offers/[id]/page.tsx` created
  - Verify: Displays all offered items (images, titles, conditions)
  - Verify: Displays all requested items
  - Verify: Shows offer message
  - Verify: If counter-offers exist, displays negotiation history timeline

- [ ] **Task 6.4**: Add offer actions to detail page (US-005, US-007, US-011)
  - Verify: Accept button (if user is target and status PENDING)
  - Verify: Counter-offer button (if status PENDING)
  - Verify: Cancel button with reason modal
  - Verify: Complete button (if status ACCEPTED)

- [ ] **Task 6.5**: Create offer form modal/page (US-001)
  - Verify: Target user selector (optional for broadcast)
  - Verify: My items multi-select (checkboxes with previews)
  - Verify: Requested items multi-select (if directed)
  - Verify: Message textarea
  - Verify: Submit calls `api.offers.create`

- [ ] **Task 6.6**: Create counter-offer form (US-005)
  - Verify: Pre-fills with current offer context
  - Verify: Allows editing offered/requested items
  - Verify: Submit calls `api.offers.counterOffer`

## Medium (Frontend - Broadcast & Interests)

- [ ] **Task 7.1**: Create Broadcast feed page (US-003)
  - Verify: `/app/broadcast/page.tsx` created
  - Verify: Calls `api.offers.listBroadcast`
  - Verify: Displays broadcast offers in grid/list
  - Verify: Optional filters for category/condition

- [ ] **Task 7.2**: Add interest button to broadcast offers (US-004)
  - Verify: "Send Interest" button on broadcast offer cards
  - Verify: Calls `api.interests.create`
  - Verify: Shows success feedback

- [ ] **Task 7.3**: Show interests on user's broadcast offers (US-004)
  - Verify: On offer detail page (if user is offer owner), show interested users
  - Verify: Calls `api.interests.listForOffer`
  - Verify: Link to start negotiation (create directed offer)

## Medium (Frontend - Chat)

- [ ] **Task 8.1**: Create Chat page with conversation list (US-009)
  - Verify: `/app/chat/page.tsx` created
  - Verify: Calls `api.chat.listConversations`
  - Verify: Displays conversations with last message preview, timestamp
  - Verify: Badge/icon to distinguish offer chats vs direct chats

- [ ] **Task 8.2**: Create Conversation detail page (US-008, US-009)
  - Verify: `/app/chat/[id]/page.tsx` created
  - Verify: Calls `api.chat.listMessages` for conversation
  - Verify: Displays messages in scrollable container
  - Verify: If offer-related, shows offer context header with link

- [ ] **Task 8.3**: Add message input and send (US-008, US-009)
  - Verify: Input field and send button at bottom of conversation page
  - Verify: Calls `api.chat.sendMessage`
  - Verify: Optimistically adds message to UI
  - Verify: Polling or manual refresh to fetch new messages

- [ ] **Task 8.4**: Embed chat in offer detail page (US-008)
  - Verify: Offer detail page includes embedded chat component
  - Verify: Uses same conversation as standalone chat page
  - Verify: Auto-creates conversation on first message

- [ ] **Task 8.5**: Add "Message" button to user profiles/item listings (US-009)
  - Verify: Button to initiate direct chat with item owner
  - Verify: Calls `api.chat.getOrCreateConversation` with otherUserId

## Low (Polish & Edge Cases)

- [ ] **Task 9.1**: Add item unavailability indicator in offers (US-012)
  - Verify: If item status = DELETED, show "no longer available" badge
  - Verify: Offer UI explains item was removed
  - Verify: User can remove item from offer or cancel

- [ ] **Task 9.2**: Add empty states
  - Verify: No offers yet (sent/received)
  - Verify: No broadcast offers
  - Verify: No messages in conversation

- [ ] **Task 9.3**: Add loading states
  - Verify: Skeleton loaders for offer lists
  - Verify: Spinner when sending message
  - Verify: Loading state for offer actions (accept/cancel)

- [ ] **Task 9.4**: Update navigation to include new pages
  - Verify: Offers tab in main navigation
  - Verify: Broadcast tab or accessible from home
  - Verify: Chat tab in main navigation

- [ ] **Task 9.5**: Add error handling and validation
  - Verify: Cannot create offer with no items
  - Verify: Cannot accept own offer
  - Verify: Graceful error messages for API failures

## Verification Gate
After all tasks complete, verify:
- [ ] All user stories (US-001 through US-012) acceptance criteria met
- [ ] All PRD screens from §5 implemented
- [ ] Database schema matches PRD §4 Data Model
- [ ] All tRPC procedures from PRD §4 API table exist and work
- [ ] Tests passing (if tests exist)
- [ ] Can complete full flow: create broadcast offer → receive interest → negotiate via counters → accept → chat → complete
- [ ] Can complete directed offer flow: select items → send offer → counter-offer → accept → complete
- [ ] Item soft-delete works correctly with active offers
