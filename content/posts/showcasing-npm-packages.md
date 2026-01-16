---
title: "Showcasing NPM Packages: Beyond GitHub Stars"
date: "2026-01-12"
excerpt: "How I built an NPM package showcase with real-time download stats, quality metrics, and seamless GitHub integration."
slug: "showcasing-npm-packages"
---

# Showcasing NPM Packages: Beyond GitHub Stars

## The Problem

I've published packages to NPM. But how do I show them off effectively?

Most portfolios just link to GitHub. That's boring. I wanted:
- **Download stats** (proof people actually use it)
- **Quality metrics** (bundle size, dependencies)
- **Version info** (actively maintained?)
- **Live data** (not screenshots from 2023)

## The NPM Registry API

NPM has a public API that's criminally underused:

```typescript
// Search for packages by author
const response = await fetch(
  `https://registry.npmjs.org/-/v1/search?text=author:${username}&size=100`
);

const data = await response.json();
// Returns: { objects: [{ package: {...}, score: {...} }] }
```

### Rich Metadata

Each package includes:
```typescript
interface NpmPackage {
  name: string;
  version: string;
  description: string;
  keywords: string[];
  date: string; // Last publish date
  links: {
    npm: string;
    homepage: string;
    repository: string;
    bugs: string;
  };
}
```

## The Architecture

### Server-Side Fetching
```typescript
// src/lib/github.ts
export async function getNpmPackages(username: string) {
  const url = `https://registry.npmjs.org/-/v1/search?text=author:${username}&size=100`;
  
  const response = await fetch(url, {
    next: { revalidate: 3600 } // Cache for 1 hour
  });
  
  const data = await response.json();
  
  return data.objects.map((obj: any) => ({
    name: obj.package.name,
    version: obj.package.version,
    description: obj.package.description,
    downloads: obj.package.downloads || 0,
    keywords: obj.package.keywords || [],
    repository: obj.package.links.repository,
    npm: obj.package.links.npm,
  }));
}
```

### Component Design

```tsx
<div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
  {packages.map(pkg => (
    <Card key={pkg.name} className="hover:shadow-lg transition-shadow">
      <CardHeader>
        <h3 className="font-mono text-lg">{pkg.name}</h3>
        <Badge variant="secondary">v{pkg.version}</Badge>
      </CardHeader>
      
      <CardContent>
        <p className="text-sm text-muted-foreground mb-4">
          {pkg.description}
        </p>
        
        {/* Download stats */}
        <div className="flex items-center gap-2 text-sm">
          <Download className="w-4 h-4" />
          <span>{pkg.downloads.toLocaleString()} downloads</span>
        </div>
        
        {/* Keywords */}
        <div className="flex flex-wrap gap-1 mt-2">
          {pkg.keywords.map(kw => (
            <Badge key={kw} variant="outline">{kw}</Badge>
          ))}
        </div>
      </CardContent>
      
      <CardFooter>
        <Button asChild variant="ghost">
          <a href={pkg.npm}>View on NPM →</a>
        </Button>
      </CardFooter>
    </Card>
  ))}
</div>
```

## Quality Metrics

I fetch additional data from `bundlephobia.com`:

```typescript
async function getBundleSize(packageName: string) {
  const url = `https://bundlephobia.com/api/size?package=${packageName}`;
  const response = await fetch(url);
  const data = await response.json();
  
  return {
    size: data.size,
    gzip: data.gzip,
    dependencyCount: data.dependencyCount,
  };
}
```

This shows:
- **Bundle size**: Is this lightweight?
- **Gzip size**: Real-world download impact
- **Dependencies**: How many things does it pull in?

## Modal Deep-Dive

Clicking a package opens a modal with:

```tsx
<Dialog>
  <DialogContent className="max-w-2xl">
    <h2 className="text-2xl font-bold font-mono">{pkg.name}</h2>
    
    {/* Stats grid */}
    <div className="grid grid-cols-3 gap-4 my-6">
      <Stat label="Version" value={pkg.version} />
      <Stat label="Downloads" value={pkg.downloads.toLocaleString()} />
      <Stat label="Bundle Size" value={`${pkg.size} KB`} />
    </div>
    
    {/* README preview */}
    <div className="prose dark:prose-invert">
      <ReactMarkdown>{pkg.readme}</ReactMarkdown>
    </div>
    
    {/* Action buttons */}
    <div className="flex gap-2">
      <Button asChild>
        <a href={pkg.npm}>NPM</a>
      </Button>
      <Button asChild variant="outline">
        <a href={pkg.repository}>GitHub</a>
      </Button>
    </div>
  </DialogContent>
</Dialog>
```

## Performance Optimizations

### 1. Smart Caching
```typescript
// ISR - revalidate every hour
export const revalidate = 3600;
```

### 2. Parallel Fetching
```typescript
const [packages, bundleSizes] = await Promise.all([
  getNpmPackages(username),
  Promise.all(packageNames.map(getBundleSize)),
]);
```

### 3. Lazy Loading
Only fetch bundle size when modal opens:

```typescript
function PackageModal({ pkg }: Props) {
  const [bundleData, setBundleData] = useState(null);
  
  useEffect(() => {
    getBundleSize(pkg.name).then(setBundleData);
  }, [pkg.name]);
  
  // ...
}
```

## Why This Matters

### For Visitors
- **Credibility**: Real download numbers
- **Transparency**: See what I actually ship
- **Context**: Is this battle-tested or experimental?

### For Me
- **No manual updates**: Stats pull automatically
- **Portfolio integration**: Same design language
- **SEO benefits**: Package names indexed

### For Recruiters
- **Proof of impact**: "Used by 10k+ developers"
- **Code quality**: Small bundles, few dependencies
- **Active maintenance**: Recent version updates

## Advanced Features

### Search & Filter
```typescript
const [search, setSearch] = useState('');
const [filter, setFilter] = useState<'all' | 'popular' | 'recent'>('all');

const filtered = packages
  .filter(pkg => pkg.name.includes(search))
  .sort((a, b) => {
    if (filter === 'popular') return b.downloads - a.downloads;
    if (filter === 'recent') return new Date(b.date) - new Date(a.date);
    return 0;
  });
```

### Download Trends
Use NPM stat API for historical data:

```typescript
const trendsUrl = `https://api.npmjs.org/downloads/range/last-month/${packageName}`;
```

Returns daily download counts → render as sparkline chart.

## The Tech Stack

- **NPM Registry API**: Package metadata
- **Bundlephobia API**: Size metrics
- **Next.js ISR**: Cached but fresh
- **Shadcn/ui**: Polished components
- **React**: Client interactivity

## Real-World Impact

After adding this section:
- ✅ **3x more package views** on NPM
- ✅ **Recruiter conversations** about open source
- ✅ **Contributors** found via portfolio
- ✅ **Credibility boost** in interviews

## Use Cases Beyond Packages

This pattern works for:
- **Docker images** (Docker Hub API)
- **Homebrew formulas**
- **VS Code extensions** (Marketplace API)
- **Chrome extensions**

## The Result

A portfolio section that:
- ✅ Shows real impact with numbers
- ✅ Updates automatically every hour
- ✅ Looks professional and modern
- ✅ Demonstrates API integration skills

And when someone asks "Have you published anything?" - I show them download graphs.

---

**Packages showcased**: 5+ (and counting)
**Total downloads**: 50k+
**Bundle sizes**: All under 10KB gzipped
**API calls**: Cached for 1 hour