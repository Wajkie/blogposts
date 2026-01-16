---
title: "How I Accidentally Built for Microservices"
date: "2026-01-11"
excerpt: "I just wanted clean code. Turns out I was building scalable architecture without realizing it. Here's what happened."
slug: "accidentally-built-for-microservices"
---

# How I Accidentally Built for Microservices

## The Starting Point

I didn't set out to build "microservices architecture" or "service-oriented design". I just wanted my portfolio code to be clean and maintainable.

But looking back at what I built, I realize something interesting happened.

## The Simple Rules I Followed

### 1. One folder per feature
```
src/lib/actions/
├── analytics/
├── auth/
├── bio/
├── email/
├── journey/
├── posts/
└── projects/
```

Each feature gets its own folder. That's it. Nothing fancy.

### 2. Server actions as boundaries
```typescript
// src/lib/actions/journey.ts
'use server';

export async function getJourneyEntries() {
  return db.selectFrom('journeyEntries').selectAll().execute();
}

export async function createJourneyEntry(data: JourneyFormData) {
  return db.insertInto('journeyEntries').values(data).execute();
}
```

Each action is self-contained. It doesn't import from other features.

### 3. No cross-feature dependencies
Analytics doesn't know Journey exists.  
Journey doesn't know Posts exists.  
They all just talk to the database.

## What I Noticed Later

After building this for a while, I realized something:

**I could move any feature to a separate server.**

Want to run analytics on its own VPS?
```typescript
// Before (local server action)
import { getVisitorStats } from '@/lib/actions/analytics';

// After (remote service)
const stats = await fetch('https://analytics.wajkie.dev/stats');
```

The interface stays the same. The UI doesn't care where the data comes from.

## Why This Works

### Loose Coupling
No feature imports code from another feature. They're already isolated.

### Clear Boundaries
Server actions define the contract. Change the implementation, keep the interface.

### Database as Integration Point
Features communicate through the database, not direct function calls.

```
┌─────────────┐
│  Analytics  │─┐
└─────────────┘ │
                ▼
┌─────────────┐ ┌──────────┐
│   Journey   │▶│ Database │
└─────────────┘ └──────────┘
                ▲
┌─────────────┐ │
│    Posts    │─┘
└─────────────┘
```

## The Accidental Benefits

### Easy to Extract
Want to turn visitor tracking into its own product?
1. Copy `src/lib/actions/analytics`
2. Add an HTTP layer
3. Done

### Easy to Replace
Don't like how Journey works? Rewrite it. Nothing else breaks.

### Easy to Scale
Analytics getting heavy traffic? Move it to a bigger server. The rest stays put.

### Easy to Test
Each feature is isolated. Mock the database, test the actions.

## What I'd Change

If I were rebuilding from scratch, I'd probably:

### Use a shared types package
```typescript
// @wajkie/types
export interface JourneyEntry { ... }
```

Keeps features independent but types consistent.

### Add an API layer upfront
Server actions are great for internal use, but an HTTP API would make extraction even easier.

### Document the boundaries
A simple `ARCHITECTURE.md` explaining the rules would help future me.

## The Lesson

Good architecture isn't always intentional. Sometimes it's just:
- Breaking things into small pieces
- Avoiding dependencies
- Keeping functions focused

I didn't study microservices patterns. I just wanted my code to make sense.

Turns out, that's most of the battle.

## Could This Actually Become Microservices?

Honestly? Yes. And it wouldn't be hard.

```typescript
// Current: Direct server action
const entries = await getJourneyEntries();

// Future: HTTP service
const entries = await fetch('https://api.wajkie.dev/journey/entries')
  .then(r => r.json());
```

The component code barely changes. The infrastructure does.

## Why I'm Not Doing It Yet

Because I don't need to.

Premature optimization is real. Right now:
- Everything is fast
- Deploys are simple
- Costs are low

When (if?) I need to scale, the path is clear. That's enough.

## The Takeaway

You don't need to architect for microservices on day one.

You just need to:
- ✅ Keep features isolated
- ✅ Define clear boundaries
- ✅ Avoid tight coupling

Do that, and scaling becomes a deployment problem, not a rewrite problem.

---

**Current architecture**: Monolith  
**Future-ready for**: Microservices  
**Time to extract a service**: ~1 day  
**Rewrites needed**: Zero