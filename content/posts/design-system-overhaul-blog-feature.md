---
title: "Design System Overhaul & Blog Feature"
date: "2026-01-13"
excerpt: "Built a complete liquid glass design system and implemented a full-featured blog with GitHub as the backend. Here's how it works."
slug: "design-system-overhaul-blog-feature"
---

# Design System Overhaul & Blog Feature

Today I implemented 2 major things: built a complete liquid glass design system and implemented a full-featured blog with GitHub as the backend.

## 🎨 Design System - Liquid Glass & Accessibility

### Liquid Glass Design

The entire design now uses glassmorphism with:
- Backdrop blur and saturate for authentic glass effect
- Subtle shine gradient on all cards
- Transparent layers creating depth

### Muted Color Palette

Switched from vibrant blue/turquoise to a professional muted palette:
- **Primary**: Desaturated cool gray with hint of purple (`oklch(0.75 0.08 285)`)
- **Accent**: Subtle warm gray (`oklch(0.55 0.03 350)`)
- **Background**: Deep dark (`oklch(0.06 0.01 264)`)
- Chroma reduced from 0.14-0.16 to 0.03-0.08

### Shadow System for Depth

Implemented a 6-level shadow system instead of color saturation:

```css
--shadow-sm: Inputs and subtle elements
--shadow-md: Hover states
--shadow-lg: Cards and sections
--shadow-xl: Modals and overlays
--shadow-glow: Primary buttons
--shadow-glow-lg: Enhanced glow on hover
```

### UI Components

Built a complete component library:

**Card**: Polymorphic with `as` prop for semantic HTML
```tsx
<Card as="section" aria-labelledby="title">
  <CardHeader>
    <CardTitle id="title">Content</CardTitle>
  </CardHeader>
</Card>
```

**Button**: `asChild` pattern for link rendering
```tsx
<Button asChild>
  <Link href="/blog">View Blog →</Link>
</Button>
```

**Input/Textarea**: Animated border transitions (1s smooth trailing light)

**Label**: Fully accessible with proper associations

### Accessibility First

- ✅ **Semantic HTML**: `<main>`, `<section>`, `<nav>`, `<article>`, `<time>`
- ✅ **ARIA labels**: Proper `labelledby`, `describedby`, `busy` states
- ✅ **Keyboard navigation**: Focus-visible with ring
- ✅ **Screen readers**: `sr-only` headings where needed
- ✅ **Reduced motion**: Respects `prefers-reduced-motion`
- ✅ **WCAG AA compliant**: All contrasts meet requirements
- ✅ **0 WAVE alerts**: From 5 → 0 accessibility issues

## 📝 Blog Feature - GitHub as CMS

### The Architecture

Instead of a traditional CMS, I use GitHub as the backend:

```
┌──────────┐
│  Admin   │ Write post in markdown
└────┬─────┘
     │
     ▼
┌──────────┐
│PostgreSQL│ Save metadata (title, slug, excerpt)
└────┬─────┘
     │
     ▼
┌──────────┐
│  GitHub  │ Push .md file to repo
└────┬─────┘
     │
     ▼
┌──────────┐
│   Blog   │ Fetch from GitHub, render with ISR
└──────────┘
```

### How It Works

**1. Writing Posts (/admin)**
- Markdown editor with live preview
- GitHub Flavored Markdown support
- Syntax highlighting with `rehype-highlight`
- Toolbar for quick formatting

**2. Publishing**

When you click "Publish":
```typescript
// 1. Save metadata to PostgreSQL (fast queries)
await db.insertInto('posts').values({
  title, slug, excerpt, date
}).execute();

// 2. Push markdown to GitHub
await octokit.rest.repos.createOrUpdateFileContents({
  owner: GITHUB_OWNER,
  repo: GITHUB_REPO,
  path: `content/posts/${slug}.md`,
  content: Buffer.from(markdown).toString('base64'),
});

// 3. Post appears immediately on /blog
```

**3. Public Display (/blog, /blog/[slug])**
- Fetches metadata from PostgreSQL (fast!)
- Reads markdown content from GitHub (runtime)
- **ISR** (Incremental Static Regeneration): Caches for 60 seconds
- No rebuild needed - updates automatically!

### Tech Stack

- **Database**: Neon PostgreSQL + Kysely (type-safe queries)
- **Content**: GitHub API (Octokit)
- **Rendering**: ReactMarkdown + remark-gfm + rehype plugins
- **Auth**: TOTP (Google Authenticator) for admin access
- **Styling**: Tailwind CSS v4 with okLCH color space

### Benefits of This Flow

✅ **Git history**: All content is version-controlled  
✅ **Backup**: GitHub is your backup  
✅ **No CMS lock-in**: You own your .md files  
✅ **Fast queries**: PostgreSQL for metadata  
✅ **CI/CD tracking**: See deploy status on projects page  
✅ **Offline editing**: Can work locally in Git

## 🚀 Projects Page Integration

The projects page now also shows blog-related activity:
- GitHub repos with automatic CI/CD detection
- Latest commit messages
- Deploy status from GitHub Actions
- Direct links to both GitHub and live projects

## 📊 Results

**Design:**
- 99% TypeScript coverage
- 0 accessibility alerts
- Consistent liquid glass theme
- Smooth 1s transitions on inputs

**Blog:**
- Runtime content updates (no rebuild)
- 60s ISR cache for performance
- Full markdown support with code highlighting
- Type-safe queries to database

## Example: Writing This Post

I wrote this in the admin panel:

```markdown
---
title: "Design System Overhaul"
slug: "design-system-overhaul"
date: "2026-01-13"
excerpt: "Built a complete liquid glass design system..."
---

# Design System Overhaul

Content here...
```

Clicked **Publish**, and:
1. Metadata saved to PostgreSQL in ~10ms
2. Markdown pushed to GitHub in ~200ms
3. Post live on /blog instantly
4. ISR cache updates every 60s

No build. No deploy. Just publish.

## Next Steps

Continue polishing details and adding hover animations for extra sauce! 🚀

---

**Design system**: Liquid glass with muted okLCH palette  
**Blog posts**: Stored in GitHub, indexed in PostgreSQL  
**Accessibility**: WCAG AA compliant, 0 WAVE alerts  
**Performance**: 60s ISR, type-safe queries