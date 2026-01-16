---
title: "Real-Time Stats Without WebSockets"
date: "2026-01-14"
excerpt: "Building a live dashboard with polling, auto-refresh, and smooth UX - no WebSockets required."
slug: "real-time-stats-without-websockets"
---

# Real-Time Stats Without WebSockets

## The Goal

I wanted a **live analytics widget** on my landing page showing:
- Total visits
- Unique visitors  
- Recent activity (last 7 days)
- Popular pages

And it needed to feel real-time without complicated infrastructure.

## Why Not WebSockets?

WebSockets are great, but they add complexity:
- Need a persistent connection server (not serverless-friendly)
- More expensive on platforms like Vercel
- Requires connection management, reconnection logic, heartbeats
- Overkill for stats that update every 30 seconds

## The Simple Solution: Polling

```typescript
useEffect(() => {
  const loadStats = async () => {
    const data = await getPublicStats();
    setStats(data);
  };

  // Initial load
  loadStats();

  // Auto-refresh every 30 seconds
  const interval = setInterval(loadStats, 30000);

  return () => clearInterval(interval);
}, []);
```

That's it. No libraries, no connection state, no edge servers.

## Server-Side Optimization

The `getPublicStats()` server action is optimized:

```typescript
'use server';

export async function getPublicStats() {
  const [totalVisits, uniqueIPs, popularPages, recentActivity] = 
    await Promise.all([
      db.selectFrom('pageVisits')
        .select((eb) => eb.fn.count('id').as('total'))
        .executeTakeFirst(),
      
      db.selectFrom('pageVisits')
        .select('ipHash')
        .distinct()
        .execute(),
      
      db.selectFrom('pageVisits')
        .select(['pagePath', (eb) => eb.fn.count('id').as('visits')])
        .groupBy('pagePath')
        .orderBy('visits', 'desc')
        .limit(5)
        .execute(),
      
      db.selectFrom('pageVisits')
        .select((eb) => eb.fn.count('id').as('visits'))
        .where('timestamp', '>', sevenDaysAgo)
        .executeTakeFirst(),
    ]);

  return {
    totalVisits: Number(totalVisits?.total || 0),
    uniqueVisitors: uniqueIPs.length,
    recentVisits: Number(recentActivity?.visits || 0),
    popularPages: popularPages.slice(0, 5),
  };
}
```

All queries run in parallel with `Promise.all()`. Total query time: ~50ms.

## UX Polish

### 1. Loading State
```tsx
if (loading) {
  return (
    <Card className="p-6 animate-pulse">
      <div className="h-4 bg-gray-200 rounded w-1/3 mb-4" />
      <div className="space-y-2">
        <div className="h-3 bg-gray-200 rounded" />
        <div className="h-3 bg-gray-200 rounded w-5/6" />
      </div>
    </Card>
  );
}
```

### 2. Smooth Number Updates
Use `.toLocaleString()` for comma separators:
```typescript
{stats.totalVisits.toLocaleString()} // 2,375 instead of 2375
```

### 3. Visual Hierarchy
- Large numbers (2xl font)
- Small labels (xs font, muted color)
- Grid layout for scannability

## Real-Time Tracking API

Visitors are tracked via a simple fetch:

```typescript
fetch('/api/visitor/track', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-API-Token': process.env.NEXT_PUBLIC_ANALYTICS_TOKEN,
  },
  body: JSON.stringify({
    action: 'visit',
    pagePath: window.location.pathname,
  }),
});
```

The API:
1. Validates token
2. Hashes visitor IP
3. Checks rate limit
4. Inserts to database

All in <10ms.

## Database Indexes

Critical for fast queries:

```sql
CREATE INDEX idx_page_visits_timestamp ON page_visits(timestamp);
CREATE INDEX idx_page_visits_token_id ON page_visits(token_id);
CREATE INDEX idx_page_visits_page_path ON page_visits(page_path);
```

## The Result

A dashboard that:
- Updates every 30 seconds automatically
- Shows live visitor counts
- Requires zero WebSocket infrastructure
- Works perfectly on serverless platforms
- Costs nothing extra

## When to Use Polling vs WebSockets

**Use Polling when**:
- Updates every 10+ seconds are acceptable
- Data is read-heavy (dashboards, stats)
- Running on serverless/edge
- Simplicity matters

**Use WebSockets when**:
- Sub-second updates required (chat, games)
- Server needs to push to specific clients
- Bidirectional real-time communication

## Performance Impact

- **Bandwidth**: ~1KB per request × 2 requests/min = negligible
- **Database load**: 4 optimized queries every 30s = totally fine
- **User experience**: Feels real-time, no lag

## Conclusion

Real-time doesn't always mean WebSockets. For dashboards, analytics, and periodic updates, **polling with smart intervals** is simpler, cheaper, and just as effective.

Sometimes the best solution is the boring one.

---

**Polling interval**: 30 seconds
**Query time**: ~50ms
**Bandwidth per update**: 1KB
**Complexity added**: 5 lines of code