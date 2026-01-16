---
title: "Building a Privacy-First Analytics System"
date: "2026-01-16"
excerpt: "How I built a custom analytics platform with IP hashing, token-based auth, and rate limiting - no cookies, no tracking scripts, just clean architecture."
slug: "building-privacy-first-analytics"
---

# Building a Privacy-First Analytics System

## The Problem with Traditional Analytics

Google Analytics, Mixpanel, and similar tools are powerful, but they come with baggage:
- Cookie consent banners
- GDPR compliance headaches
- Third-party scripts slowing down your site
- Data ownership concerns

## My Approach: Custom Analytics with Security First

I decided to build my own analytics system from scratch. Here's what makes it different:

### 1. **IP Hashing for Privacy**
Instead of storing actual IP addresses, I hash them using SHA-256:

```typescript
const ipHash = crypto
  .createHash('sha-256')
  .update(ipAddress + secret)
  .digest('hex');
```

This means I can track unique visitors without storing personally identifiable information.

### 2. **Token-Based Authentication**
Each external project gets its own tracking token:

```typescript
const token = crypto.randomBytes(32).toString('hex');
const hashedToken = await bcrypt.hash(token, 10);
```

Tokens are bcrypt-hashed (10 salt rounds) before storage. Only the project owner knows the actual token.

### 3. **Rate Limiting**
Each token has a configurable rate limit (e.g., 1000 requests/minute). This prevents abuse and keeps the database clean.

### 4. **Action Tracking**
Beyond simple page views, I track:
- `visit` - Page loads
- `click` - Button/link clicks
- `scroll` - Scroll depth
- `download` - File downloads
- `share` - Social shares

## The Tech Stack

- **Next.js 16** - Server actions for CRUD operations
- **Kysely** - Type-safe SQL queries
- **PostgreSQL** - Reliable, fast, ACID-compliant
- **bcryptjs** - Secure token hashing
- **crypto** - Built-in IP hashing

## Real-Time Dashboard

The admin panel shows:
- Total visits and unique visitors
- Top pages by traffic
- Action breakdown with visual bars
- Recent activity (last 24h)
- Per-project filtering

All queries are optimized with proper indexing on `tokenId`, `timestamp`, and `pagePath`.

## Why This Matters

**Performance**: No external scripts = faster page loads

**Privacy**: IP hashing means GDPR compliance without consent banners

**Ownership**: My data, my database, my control

**Cost**: $0/month (just PostgreSQL storage)

**Learning**: Building it myself taught me about security, hashing, and database optimization

## What's Next?

- [ ] WebSocket support for real-time updates
- [ ] Retention analysis (DAU/MAU)
- [ ] Funnel tracking
- [ ] Geographic data (city-level, no precise location)

Building your own analytics is a fantastic learning experience. You understand every query, every hash, every rate limit decision. And when someone asks "how do you handle user privacy?" - you can explain exactly how, because you built it.

---

**Tech stack**: Next.js 16, TypeScript, Kysely, PostgreSQL, bcrypt, crypto
**Lines of code**: ~500
**Time to build**: 1 day
**Third-party dependencies**: 0