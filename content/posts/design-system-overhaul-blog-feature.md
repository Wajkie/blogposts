---
title: "Design System Overhaul & Blog Feature"
date: "2026-01-13"
excerpt: "Portfoliobyggande, dag 1"
slug: "design-system-overhaul-blog-feature"
---

# Design System Overhaul & Blog Feature

Idag implementerades 2 stora saker: byggt ett komplett liquid glass design system och implementerat en fullständig blogg-feature med GitHub som backend.

## 🎨 Design System - Liquid Glass & Accessibility

### Liquid Glass Design
Hela designen bygger nu på glassmorfism med:
- **Backdrop blur** och saturate för äkta glass-effekt
- **Subtle shine gradient** på alla cards
- **Transparenta lager** som skapar depth

### Muted Color Palette
Bytte från vibrant blå/turkos till en professionell muted palette:
- **Primary**: Desaturerad cool gray med hint av purple (`oklch(0.75 0.08 285)`)
- **Accent**: Subtle warm gray (`oklch(0.55 0.03 350)`)
- **Background**: Deep dark (`oklch(0.06 0.01 264)`)
- **Chroma reducerat** från 0.14-0.16 till 0.03-0.08

### Shadow System för Depth
Implementerat 6-nivåers skuggsystem istället för färgmättnad:
- `--shadow-sm`: Inputs och subtle elements
- `--shadow-md`: Hover states
- `--shadow-lg`: Cards och sections
- `--shadow-xl`: Modals och overlays
- `--shadow-glow`: Primary buttons
- `--shadow-glow-lg`: Enhanced glow på hover

### UI Components
Byggde komplett komponentbibliotek:
- **Card**: Polymorfisk med `as` prop för semantic HTML
- **Button**: `asChild` pattern för link-rendering
- **Input/Textarea**: Animated border transitions (1s smooth trailing light)
- **Label**: Fullt accessible

### Accessibility First
- **Semantic HTML**: `<main>`, `<section>`, `<nav>`, `<article>`, `<time>`
- **ARIA labels**: Proper labelledby, describedby, busy states
- **Keyboard navigation**: Focus-visible med ring
- **Screen readers**: sr-only headings där behövs
- **Reduced motion**: Respekterar prefers-reduced-motion
- **WCAG AA compliant**: Alla kontraster uppfyller krav
- **0 WAVE alerts**: Från 5 → 0 accessibility issues

## 📝 Blog Feature - GitHub as CMS

### Arkitekturen
Istället för traditionell CMS använder vi GitHub som backend:

### Hur det fungerar

**1. Skriva inlägg** (`/admin`)
- Markdown editor med live preview
- GitHub Flavored Markdown support
- Syntax highlighting med rehype-highlight
- Toolbar för snabb formatting

**2. Publicera**
När du klickar "Publicera":
1. Sparas till **Neon PostgreSQL** (metadata: title, slug, date, excerpt)
2. Pushas till **GitHub repo** (`content/posts/slug.md`)
3. GitHub Actions (CI/CD) detekteras och visas på projektsidan
4. Inlägget syns direkt på `/blog`

**3. Visa publikt** (`/blog`, `/blog/[slug]`)
- Hämtar metadata från PostgreSQL (snabbt!)
- Läser markdown-innehåll från GitHub (runtime)
- **ISR (Incremental Static Regeneration)**: Cachar i 60 sekunder
- Ingen rebuild behövs - uppdateras automatiskt!

### Tech Stack
- **Database**: Neon PostgreSQL + Kysely (type-safe queries)
- **Content**: GitHub API (Octokit)
- **Rendering**: ReactMarkdown + remark-gfm + rehype plugins
- **Auth**: TOTP (Google Authenticator) för admin-access
- **Styling**: Tailwind CSS v4 med okLCH color space

### Fördelar med detta flöde
✅ **Git history**: All content är versionshanterad  
✅ **Backup**: GitHub är din backup  
✅ **No CMS lock-in**: Äger dina `.md` filer  
✅ **Fast queries**: PostgreSQL för metadata  
✅ **CI/CD tracking**: Ser deploy-status på projektsidan  
✅ **Offline editing**: Kan jobba lokalt i Git

## 🚀 Projects Page Integration

Projektsidan visar nu även blogg-relaterad aktivitet:
- GitHub repos med automatisk CI/CD detection
- Senaste commit-meddelanden
- Deploy-status från GitHub Actions
- Direktlänkar till både GitHub och live-projekt

## 📊 Resultat

**Design:**
- 99% TypeScript coverage
- 0 accessibility alerts
- Konsistent liquid glass theme
- Smooth 1s transitions på inputs

**Blog:**
- Runtime content updates (ingen rebuild)
- 60s ISR cache för performance
- Full markdown support med code highlighting
- Type-safe queries till databas

Nästa steg: Fortsätta polera detaljer och lägga till animations på hover för extra sauce! 🚀
