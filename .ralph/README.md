# Spidey Setup Complete

## Overview

Spidey has been configured for the GreenShare Bartering Flow Enhancement project. This setup includes a comprehensive PRD derived from your requirements interview and a detailed implementation plan.

## What Was Created

### 1. Product Requirements Document (PRD)
**Location:** `.ralph/specs/PRD.md`

A complete PRD with:
- **12 User Stories** across 6 epics (Multi-Item Offers, Broadcast Offers, Negotiation Flow, Chat Integration, Trade Completion, Item Availability)
- **Full data model** specifying the new schema (Offer, Conversation, Message, Interest models)
- **API specification** with all tRPC procedures needed
- **Screen map** showing UI structure
- **Success metrics** and MVP scope definition

### 2. Implementation Plan
**Location:** `.ralph/fix_plan.md`

A task-by-task breakdown with **52 tasks** organized by priority:
- **Critical (9 tasks)**: Data model changes, Prisma schema updates
- **High (23 tasks)**: Core API implementation (offers, interests, chat)
- **Medium (15 tasks)**: Frontend UI (offer pages, broadcast feed, chat)
- **Low (5 tasks)**: Polish, error handling, navigation

Each task includes verification criteria linked to PRD acceptance criteria.

### 3. Ralph Loop Configuration

- **`.ralphrc`**: Loop settings (rate limits, timeouts, allowed tools)
- **`.ralph/PROMPT.md`**: Instructions for Claude during autonomous execution
- **`.ralph/AGENT.md`**: Build, test, and run commands for the project

### 4. Git Setup

- Ralph state files added to `.gitignore`
- Initial commit created with all setup files

## Key Design Decisions from Your Requirements

Based on your responses during the interview:

1. **Multi-item bundles**: Offers support multiple items on both sides (offeredItemIds[], requestedItemIds[])
2. **Counter-offers**: New offer records linked via parentOfferId (full audit trail)
3. **Broadcast + Directed**: targetUserId nullable for broadcast mode
4. **Interest system**: Users send interest to broadcasts, owner starts negotiation
5. **Dual chat**: Both offer-specific chats and general user-to-user messaging
6. **Simplified schema**: Removed OfferedItem junction table, removed ItemImage table (images now String[] on Item)
7. **Offer states**: PENDING, ACCEPTED, COMPLETED, CANCELLED
8. **Soft delete**: Items in offers become DELETED status (not hard-deleted)
9. **Completion flow**: After acceptance, use chat to arrange pickup, either party marks complete

## How to Run the Autonomous Loop

### Option 1: Monitor Mode (Recommended)
```bash
ralph --monitor
```

This launches:
- The autonomous execution loop
- A live dashboard in tmux showing progress, file changes, circuit breaker state

### Option 2: Loop Only
```bash
ralph
```

Runs the loop without the dashboard (quieter, but less visibility).

### Option 3: Dashboard Separately
```bash
# In one terminal:
ralph

# In another terminal:
ralph-monitor
```

## What the Loop Will Do

Ralph will work through `fix_plan.md` from top to bottom:

1. **Phase 1 (Critical)**: Update Prisma schema with new models (Offer, Conversation, Message, Interest)
2. **Phase 2 (High)**: Implement all tRPC procedures for offers, interests, and chat
3. **Phase 3 (Medium)**: Build frontend UI components (offer pages, broadcast feed, chat)
4. **Phase 4 (Low)**: Add polish (error handling, empty states, navigation)

Each iteration:
- Picks the highest priority unchecked task
- Implements it
- Runs type checks
- Updates fix_plan.md (checks off completed tasks)
- Outputs RALPH_STATUS

The loop stops when all tasks are complete AND the verification gate passes.

## Circuit Breaker Safety

Ralph has built-in circuit breakers to prevent infinite loops:

- **No Progress**: Halts if 3 loops with no file changes
- **Same Error**: Halts if 5 loops with identical errors
- **Output Decline**: Monitoring if 2 loops with minimal output

If blocked, Ralph will stop and explain the issue.

## Session Continuity

- Sessions persist for 24 hours via `--continue` flag
- Claude remembers context across iterations (no repeated explanations)

## Reviewing Progress

While the loop runs:
- Check `fix_plan.md` to see completed tasks (checked boxes)
- Review `.ralph/logs/` for full execution logs
- Use `ralph-monitor` dashboard for real-time status

## Editing the Plan

You can edit `fix_plan.md` or `.ralph/specs/PRD.md` at any time:
- Add/remove tasks
- Change priorities (reorder tasks)
- Update acceptance criteria

Ralph will pick up changes on the next loop iteration.

## Manual Intervention

If you need to step in:
1. Stop Ralph (Ctrl+C)
2. Make your changes
3. Commit to git (important for circuit breaker file tracking)
4. Restart Ralph with `ralph --monitor`

## Next Steps

**Review the PRD** (`.ralph/specs/PRD.md`) and **fix_plan** (`.ralph/fix_plan.md`). If you want to make any changes to requirements or task priorities, edit those files now before running the loop.

When ready:
```bash
ralph --monitor
```

Let Spidey handle the implementation autonomously!

---

**Questions or Issues?**
- Review PRD: `.ralph/specs/PRD.md`
- Review plan: `.ralph/fix_plan.md`
- Edit loop config: `.ralphrc`
- Edit instructions: `.ralph/PROMPT.md`
