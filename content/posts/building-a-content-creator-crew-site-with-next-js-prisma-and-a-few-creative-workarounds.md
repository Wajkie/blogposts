---
title: "Building a Content Creator Crew Site with Next.js, Prisma, and a Few Creative Workarounds"
date: "2026-05-18"
excerpt: "A in-depth explanation and reasoning for building https://chuugs.vercel.app/"
slug: "building-a-content-creator-crew-site-with-next-js-prisma-and-a-few-creative-workarounds"
---

# Building a Content Creator Crew Site with Next.js, Prisma, and a Few Creative Workarounds

When a gaming collective needs a home on the web, a generic social media page stops being enough pretty quickly. You want live stream status, member profiles, a rank hierarchy, embedded short-form video, and a way to manage all of it without touching code every time someone new joins the crew. This post walks through how the Chuugs crew site is built, the architectural decisions behind it, and some of the less obvious problems we had to solve.

---

## The Stack

The site is a **Next.js 16** app using the App Router, backed by **PostgreSQL** via **Prisma ORM**, styled with **Tailwind CSS v4**, and deployed on **Vercel**. Nothing exotic there. The interesting part is in how the pieces fit together.

```
Next.js 16 (App Router)
├── React 19
├── Tailwind CSS v4
├── TanStack Query (client-side data)
└── Prisma 7 + @prisma/adapter-pg → PostgreSQL (Neon)
```

External integrations:
- **Twitch Helix API** — live stream status and profile images
- **TikTok oEmbed** — video metadata without an API key
- **Vercel Blob** — screenshot storage

---

## Project Structure

```
src/
├── app/
│   ├── page.tsx              # Homepage — composes all sections
│   ├── members/[crewName]/   # Dynamic member profile pages
│   └── api/                  # REST API (crew, characters, videos, ranks, twitch, auth)
├── actions/                  # Server Actions: DB queries, Twitch, TikTok
├── components/               # UI sections + shadcn/ui primitives
├── hooks/                    # React Query hooks
├── lib/
│   ├── db.ts                 # Prisma singleton + DBHandler class
│   └── cors.ts               # CORS helper
prisma/
├── schema.prisma
└── seed.ts
scripts/                      # Token generation utilities
discord-bot/                  # Optional standalone Discord bot
```

The homepage is essentially a composition root — it imports each section as a component (`<TwitchLive />`, `<MeetTheCrew />`, `<TikTokFeed />`, etc.) and the server fetches everything before the page lands. This keeps individual sections focused and easy to swap out or disable.

---

## Data Layer

All database access goes through a single `DBHandler` class in `src/lib/db.ts`. It wraps Prisma and exposes named methods rather than spreading `prisma.` calls across the codebase.

```ts
export class DBHandler {
  static prisma = prisma

  static async getCrewMembers(): Promise<CrewMemberWithRelations[]> {
    const [members, ranks] = await Promise.all([
      prisma.crewMember.findMany({ include: { characters: true, videos: true } }),
      prisma.rankConfig.findMany({ orderBy: { order: 'asc' } }),
    ])
    return orderByRankConfig(members, ranks, (m) => m.rank)
  }
  // ...
}
```

This also handles one non-obvious requirement: **rank-aware ordering**. Ranks are data-driven — there is no hard-coded enum in the UI. The `RankConfig` table stores each rank with an `order` integer, and `orderByRankConfig` uses that to sort any list consistently.

```ts
function orderByRankConfig<T>(
  items: T[],
  ranks: { key: string }[],
  getRank: (item: T) => string | null | undefined,
): T[] {
  const buckets = new Map<string, T[]>(ranks.map((r) => [r.key, []]))
  const unranked: T[] = []
  for (const item of items) {
    const key = getRank(item)
    if (key && buckets.has(key)) buckets.get(key)!.push(item)
    else unranked.push(item)
  }
  return [...ranks.flatMap((r) => buckets.get(r.key)!), ...unranked]
}
```

Any unranked member falls to the bottom. The same function works for crew members, roleplay characters, and anything else that carries a rank string — no duplication.

The Prisma client itself follows the standard singleton pattern to survive Next.js hot reloads in development:

```ts
const globalForPrisma = globalThis as unknown as { prisma: PrismaClient | undefined }
export const prisma = globalForPrisma.prisma ?? new PrismaClient({ adapter })
if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

---

## The Public Site vs. the CMS — Two URLs, One API

The site has no built-in admin panel. Content management lives in a **separate CMS** hosted at a different URL. This is an intentional architectural boundary: the public site is read-heavy and optimised for visitors; the CMS is write-heavy and only accessible to crew managers. They never need to be deployed together.

The connection between them is the REST API baked into the Next.js app. All routes in `src/app/api/` respond to both the Next.js server-rendered frontend (via Server Actions) and the external CMS (via fetch from a different origin).

**CORS** is how the API allows the CMS in while keeping random origins out:

```ts
// src/lib/cors.ts
export function corsHeaders(): Record<string, string> {
  return {
    'Access-Control-Allow-Origin': process.env.CMS_URL ?? '*',
    'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
    'Access-Control-Allow-Headers': 'Authorization, Content-Type',
  }
}
```

When `CMS_URL` is set in production, only that origin is allowed. During local development it falls back to `*` so you can hit the API from anywhere without friction.

Every route exports an `OPTIONS` handler for preflight requests, and every response is wrapped with `withCors()`:

```ts
export async function OPTIONS() {
  return preflight()
}

export async function GET() {
  const crewMembers = await DBHandler.getCrewMembers()
  return withCors(NextResponse.json(crewMembers))
}
```

**Authentication** on write endpoints uses bcrypt-hashed tokens stored in the database. The CMS sends a `Bearer` token in the `Authorization` header; the API hashes the incoming value and compares it against the stored hash. There is no session, no cookie, no JWT — just a long random string hashed with bcrypt. Simple, stateless, and hard to brute-force.

```ts
const token = authHeader.substring(7)
const isValid = await DBHandler.verifyAdminToken(token)
if (!isValid) return withCors(NextResponse.json({ error: 'Invalid token' }, { status: 403 }))
```

The trade-off: a compromised token has to be manually rotated via `npm run db:update-token`. There is no expiry. For a low-traffic crew site managed by a handful of people this is fine; for anything with rotating staff you would want short-lived JWTs.

When a write succeeds, the API calls `revalidatePath('/', 'layout')` so the Next.js cache is purged and the public site reflects the change on the next request — no manual rebuild needed.

---

## Twitch Live Status

The Twitch widget shows which crew members are currently live, with viewer count, stream title, game name, and a thumbnail. It uses the **Twitch Helix API** with the client credentials (app-to-app) OAuth flow — no user login required.

The token exchange only needs to happen once and the token is valid for days, so it is cached in a module-level variable:

```ts
let cachedToken: { value: string; expiresAt: number } | null = null

async function getAppToken(): Promise<string> {
  if (cachedToken && Date.now() < cachedToken.expiresAt) {
    return cachedToken.value
  }
  // ... POST to id.twitch.tv/oauth2/token
  cachedToken = { value: data.access_token, expiresAt: Date.now() + (data.expires_in - 60) * 1000 }
  return cachedToken.value
}
```

When the page renders, `getCrewLiveStatus()` fires two Helix requests in parallel: one to `/helix/streams` (live data, `cache: 'no-store'`) and one to `/helix/users` (profile images, revalidated every 3 hours). The streams endpoint never caches because live status can flip at any second; profile images barely change, so they get a long TTL.

```ts
const [streamsRes, usersRes] = await Promise.all([
  fetch(`https://api.twitch.tv/helix/streams?${streamParams}`, {
    headers: { ... },
    cache: 'no-store',
  }),
  fetch(`https://api.twitch.tv/helix/users?${userParams}`, {
    headers: { ... },
    next: { revalidate: 3600 * 3 },
  }),
])
```

The response merges stream data, user data, and the database rank for each member into a single `CrewStream` shape. Members who are offline still appear — just without the live badge — because the DB is the source of truth for who is in the crew, not what Twitch currently sees.

---

## The TikTok Workaround

TikTok's official API is locked behind a lengthy approval process that makes it impractical for small projects. At the same time, simply embedding a TikTok `<iframe>` only works if the video URL is known ahead of time — you can't dynamically discover a user's latest videos without API access.

The solution is **TikTok's public oEmbed endpoint**:

```
https://www.tiktok.com/oembed?url=<encoded-video-url>
```

oEmbed is a simple open standard for rich embeds. TikTok exposes it without authentication. Given a video URL it returns title, author name, author handle, and a thumbnail — everything needed to render a card.

```ts
async function fetchOEmbed(url: string): Promise<OEmbedResponse | null> {
  try {
    const res = await fetch(
      `https://www.tiktok.com/oembed?url=${encodeURIComponent(url)}`,
      { next: { revalidate: 300 } }
    )
    if (!res.ok) return null
    return res.json()
  } catch {
    return null
  }
}
```

The video URLs themselves are stored in the `Video` table with `platform: 'tiktok'`. When you want to feature a new clip, you add its URL through the CMS. The site then enriches every stored URL with metadata at render time, with a 5-minute cache so it doesn't hammer oEmbed on every page view.

```ts
export async function getTikTokVideos(): Promise<TikTokVideoEnriched[]> {
  const videos = await DBHandler.getVideosByPlatform('tiktok')
  const results = await Promise.all(
    videos.map(async (video) => {
      const oembed = await fetchOEmbed(video.url)
      return {
        id: video.id,
        url: video.url,
        crewMemberId: video.crewMemberId,
        title: oembed?.title ?? '',
        authorHandle: oembed?.author_unique_id ?? '',
        thumbnail: oembed?.thumbnail_url ?? '',
      }
    })
  )
  return results
}
```

**Trade-offs:**
- You still have to manually add video URLs. There is no automatic "latest video" feed.
- oEmbed is undocumented and TikTok can change or rate-limit it without notice.
- If oEmbed returns `null` for a URL (e.g., deleted video, geo-block), the card renders without a thumbnail or title rather than crashing. Graceful degradation is built in.

For the crew's use case — featuring specific clips worth highlighting rather than a full feed — this is a good trade. The manual curation is intentional; not every video needs to be on the site.

---

## Schema Design

The data model is deliberately flat and simple:

```
CrewMember
  ├── RoleplayCharacter[]   (1-to-many, cascade delete)
  └── Video[]               (1-to-many, cascade delete)

RankConfig                  (rank labels + sort order)
SocialPlatform              (platform metadata)
AdminToken                  (bcrypt-hashed auth tokens)
```

`socials` on `CrewMember` is a `Json` column rather than a normalised table. Each member has a slightly different combination of platforms and the structure is flexible. Normalising it into a `CrewMemberSocialLink` table would add joins and migrations every time a new platform appears. A JSON blob sidesteps that entirely at the cost of losing foreign-key guarantees — fine for display data that's always read as a whole.

Cascade deletes on `RoleplayCharacter` and `Video` mean that removing a crew member also removes everything they own. No orphaned rows.

---

## What I'd Do Differently

**Rank as a foreign key.** `CrewMember.rank` is currently a plain `String?` that references `RankConfig.key` by convention rather than by a database constraint. If a rank key is renamed, existing members silently lose their rank ordering. A `rankId String? @relation(...)` would make the database enforce the link.

**oEmbed fan-out.** Fetching oEmbed for every TikTok URL in parallel on every page render works fine at small scale but will slow down as the video library grows. A background job that caches metadata in the database — refreshed periodically — would be more robust.

**Token expiry.** The admin token lives until manually rotated. Short-lived tokens with a refresh mechanism would be better for any team larger than two or three people.

**CMS as part of the monorepo.** The CMS lives at a separate URL and is maintained separately. A monorepo with the public site and CMS as distinct Next.js apps would make it easier to share types and keep the API contract in sync.

---

## Summary

The architecture prioritises simplicity over extensibility: one database, one deployed app, a handful of external API calls, and a clean separation between the read-facing public site and the write-facing CMS. The interesting constraints were the Twitch token caching (avoid unnecessary OAuth round-trips), the TikTok oEmbed workaround (get video metadata without an approved API), and the CORS + bcrypt auth pattern (let an external CMS talk to the API without shipping an admin panel).

For a crew site that needs to stay up, load fast, and be manageable by people who are not developers, this stack hits the right balance.
