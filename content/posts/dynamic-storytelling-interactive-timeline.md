---
title: "Dynamic Storytelling: Building an Interactive Timeline"
date: "2026-01-13"
excerpt: "How I replaced static CVs with a dynamic, editable journey timeline that tells my story in real-time - from education to career milestones."
slug: "dynamic-storytelling-interactive-timeline"
---

# Dynamic Storytelling: Building an Interactive Timeline

## The Problem with Static CVs

Traditional CVs are:
- **Outdated** the moment you finish them
- **One-size-fits-all** (same document for every application)
- **Boring** (walls of text nobody reads)
- **Hard to update** (PDF hell)

I wanted something different. Something **alive**.

## The Vision: A Living Story

Instead of a PDF that collects dust, I built a **dynamic journey timeline** that:
- Updates in real-time
- Shows my progression visually
- Lets me add new milestones instantly
- Tells a story, not just lists facts

## The Architecture

### Database Schema
```typescript
interface JourneyEntry {
  id: number;
  type: 'education' | 'work' | 'project' | 'achievement';
  title: string;
  organization: string;
  description: string;
  startDate: Date;
  endDate?: Date; // null = "Present"
  technologies: string[];
  orderIndex: number;
}
```

### Admin Panel for Real-Time Editing
```typescript
// src/lib/actions/journey.ts
'use server';

export async function createJourneyEntry(data: JourneyFormData) {
  const session = await getSession();
  if (!session.isAuthenticated) redirect('/auth/signin');
  
  const maxOrder = await db
    .selectFrom('journeyEntries')
    .select((eb) => eb.fn.max('orderIndex').as('maxOrder'))
    .executeTakeFirst();
  
  const newEntry = await db
    .insertInto('journeyEntries')
    .values({
      ...data,
      orderIndex: (maxOrder?.maxOrder || 0) + 1,
    })
    .returningAll()
    .executeTakeFirst();
  
  revalidatePath('/journey');
  return newEntry;
}
```

## Visual Timeline Component

The timeline uses a **vertical layout** with clear visual separation:

```tsx
<div className="relative border-l-2 border-primary pl-6">
  {entries.map((entry) => (
    <div key={entry.id} className="mb-8 relative">
      {/* Timeline dot */}
      <div className="absolute -left-[1.6rem] top-2 w-4 h-4 rounded-full bg-primary border-4 border-background" />
      
      {/* Content card */}
      <Card>
        <h3>{entry.title}</h3>
        <p className="text-muted-foreground">{entry.organization}</p>
        <div className="flex gap-2 mt-2">
          {entry.technologies.map(tech => (
            <Badge key={tech}>{tech}</Badge>
          ))}
        </div>
      </Card>
    </div>
  ))}
</div>
```

## Smart Date Formatting

```typescript
function formatDateRange(start: Date, end?: Date) {
  const formatter = new Intl.DateTimeFormat('sv-SE', {
    year: 'numeric',
    month: 'short',
  });
  
  const startStr = formatter.format(start);
  const endStr = end ? formatter.format(end) : 'Present';
  
  return `${startStr} - ${endStr}`;
}
```

## Why This Matters

### For Visitors
- **Engaging**: Visual timeline > wall of text
- **Scannable**: See progression at a glance
- **Current**: Always up-to-date

### For Me
- **Fast updates**: Add new entry in 30 seconds
- **Flexible**: Reorder, edit, delete anytime
- **Portfolio integration**: Same database powers multiple views

### For Employers
- **Credibility**: Live system = I actually build things
- **Transparency**: See my tech stack choices
- **Story**: Understand my journey, not just jobs

## Admin Features

### Drag-and-Drop Reordering
```typescript
async function updateOrder(entries: Array<{id: number, orderIndex: number}>) {
  await Promise.all(
    entries.map(entry =>
      db.updateTable('journeyEntries')
        .set({ orderIndex: entry.orderIndex })
        .where('id', '=', entry.id)
        .execute()
    )
  );
  revalidatePath('/journey');
}
```

### Bulk Actions
- Archive old entries
- Highlight featured items
- Export to PDF (for traditional applications)

## The Bio Connection

The journey timeline connects with **dynamic bio** section:

```typescript
interface Bio {
  id: number;
  name: string;
  title: string;
  description: string;
  skills: string[];
  values: string[];
  contactEmail: string;
  avatarUrl?: string;
}
```

Same pattern:
- Editable via admin panel
- Stored in database
- Type-safe with Kysely
- Server actions for mutations

## Tech Stack Benefits

**TypeScript** everywhere means I can't mess up field names

**Server Actions** eliminate API boilerplate

**Kysely** gives me SQL power with type safety

**Revalidation** updates pages instantly after edits

## Use Cases Beyond Portfolio

This pattern works for:
- **Company history** timelines
- **Product roadmaps**
- **Personal journals**
- **Project milestones**
- **Learning logs**

## Performance

- **SSR**: Timeline renders server-side
- **Caching**: 60s revalidation for public views
- **No hydration delay**: Pure HTML until interaction
- **Small bundle**: No heavy charting libraries

## The Result

A portfolio section that:
- ✅ Updates in seconds, not hours
- ✅ Tells a story, not just facts
- ✅ Looks professional and modern
- ✅ Demonstrates full-stack skills

And when someone asks "Can you update X on your CV?" - I do it live during the call.

---

**Database entries**: Dynamic (currently 8+)
**Admin edit time**: 30 seconds per entry
**Page load time**: <100ms
**Maintenance**: Zero