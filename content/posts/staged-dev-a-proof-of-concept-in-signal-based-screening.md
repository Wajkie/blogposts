---
title: "Staged.dev — A Proof of Concept in Signal-Based Screening"
date: "2026-06-07"
excerpt: "ATS systems filter on keywords. A junior candidate with no employer history and no degree gets rejected in three seconds — not because they can't do the job, but because they don't have the paper trail the system expects.

The signal it needs is already there. It's just not on the CV.

This project asks a simple question: if you ignore the CV entirely and look only at what someone has built in public, how far can you get?"
slug: "staged-dev-a-proof-of-concept-in-signal-based-screening"
---


### The Problem

ATS systems filter on keywords. A junior candidate with no employer history and no degree gets rejected in three seconds — not because they can't do the job, but because they don't have the paper trail the system expects.

The signal it needs is already there. It's just not on the CV.

This project asks a simple question: if you ignore the CV entirely and look only at what someone has built in public, how far can you get?

---

### What It Does

Staged.dev pulls structured signals from public GitHub activity — repository architecture, dependency graphs, commit history, project diversity — and maps them against role-based expectations. Each candidate gets a fit score, a concept coverage breakdown, and a trajectory reading.

The score is deterministic and rule-based. AI is used only for generating the narrative explanation — not for scoring, not for decisions.

**Concept match** is a binary plus frequency-weighted mapping between detected dependencies and structures in the candidate's repos, and a role-defined concept taxonomy. A role file for `junior-frontend` might list concepts like `component-based architecture`, `client-side routing`, `state management`, and `testing`. The scorer checks which of those are evidenced by actual package usage and folder structure — not by keywords in a README. The result is a coverage percentage and a list of missing concepts, not a vague similarity score.

---

### The Role of AI

AI is the smallest part of this system.

Think of it as an iceberg. At the surface — the only part AI touches — is the narrative explanation: a human-readable summary of why a candidate scored the way they did. That's it.

Everything below the surface is deterministic. Trajectory is calculated from timestamped commit history across time buckets. Concept match is a binary mapping between detected package dependencies and a role-defined taxonomy. Complexity is derived from folder depth, module count, and architectural signals. Noise filtering strips school projects below a complexity floor. Run it twice on the same profile and you get the same number. Read the role file and you can predict the score before running it.

This is the opposite of how most "AI hiring tools" work. Those are opaque by design — a score emerges from a model no one can inspect, and there's no way to understand why a candidate landed where they did. Here, the logic is open source, the role definitions are readable markdown, and every weight is documented. You can audit every point of the score.

The AI is the translator. The engineering is underneath.

---

### Why GitHub

Resumes are self-reported and unverifiable. GitHub isn't.

That said, GitHub isn't truth either. It's a higher-resolution behavioral proxy than CV data for a specific class of role. The claim is narrower than "GitHub reveals ability" — it's that repository artifacts contain more structured signal than a list of keywords, for candidates whose work happens to be public.

Folder structure, `package.json` dependencies, CI config, commit cadence — these are artifacts with structure. They're not perfect proxies for engineering ability, but they're honest ones. You can't keyword-optimize your way to a well-structured monorepo.

The system reads:
- **Project structure** — architecture depth, module organization
- **Dependencies** — what concepts a developer has actually shipped, not claimed
- **Commit history** — learning trajectory over time
- **Activity recency** — momentum as an intent signal

---

### Trajectory Over Snapshot

The most important signal isn't where someone is — it's how fast they're moving.

Trajectory is weighted 45% of the total score. A junior developer with a steep upward curve is often the better hire for a team that wants someone who grows with them. A senior developer with a flat decade of tutorial repos is a known quantity — useful, but a different kind of bet.

In an informal calibration experiment during development, the tool was run against WebDevSimplified (1.1M YouTube subscribers, teaches Next.js professionally) and a newly graduated student. The student scored higher — not because he knew more, but because his recent work was denser and harder. WDS has more repos but a flatter curve. Fewer repos, steeper trajectory, higher complexity score.

This isn't a controlled evaluation. It's one data point that behaved the way the weighting model predicts it should. Whether that generalises is an open question.

---

### What "Junior" Actually Means Here

The threshold is set at upper junior — someone hitting the ceiling of junior and ready to grow into mid. Not someone who can follow a tutorial.

The concept lists reflect this: state management beyond `useState`, custom hooks, separation of concerns beyond a single file, meaningful commit messages. These are signals that someone has hit the wall and figured it out. The gap between junior and mid in the role definitions is small by design — the tool is looking for the ones who are about to cross it.

---

### The Demographic It's Built For

There's a class of candidate that ATS systems keep missing:

- The bootcamp grad
- The self-taught developer
- The career changer
- The student who went way beyond the curriculum

They exist in every cohort. They often have years of real engineering work — shipped things, debugged production systems, built tools people use. It lives on GitHub, not on a corporate org chart. The system is built to surface them.

---

### The Dogfooding Moment

The best work built during the week Staged.dev was created was the tool itself — 12 issues, 4 MCP tools, 257 tests, a full scoring engine. None of it was pushed to GitHub during that week. All local commits.

The tool couldn't see its own creator's best work.

*"I built a tool to surface hidden developer signal. My best work that week was hidden developer signal. That's the point."*

---

### Calibration and Configurability

The scoring weights represent one set of values:

```plaintext
score = 0.45 × trajectory
      + 0.35 × concept_match
      + 0.20 × complexity
```

Trajectory is weighted highest because the tool is optimized for early-career candidates, where momentum matters more than accumulated volume. Concept match is next — shipping the right ideas matters more than raw project count. Complexity is last, partly because it's the easiest to game. A startup CTO might push trajectory higher. A bootcamp-focused agency might reduce the complexity weight, since bootcamp grads have a short history by definition. A consultancy might care more about concept coverage.

Both dials — weights and role definitions — are configurable without touching the scoring engine. Two companies using the same tool can screen for completely different things.

---

### What It Can't See

Private and enterprise work is invisible. Team contributions are underrepresented. Soft skills don't appear in folder structure. Senior developers who've spent a decade in internal monorepos will score poorly for reasons that have nothing to do with their ability.

The tool works best for open-source-adjacent roles — frontend, backend, fullstack — where public GitHub activity is a reasonable proxy for what someone has built. For designers, PMs, data scientists, or anyone whose work doesn't leave a GitHub footprint, it's the wrong lens.

---

### Failure Modes

**False positives — the over-engineered tutorial repo.** A candidate who follows along with a complex boilerplate tutorial can end up with a repo that scores well on structure and dependencies without having made many real decisions. The complexity signal is partially gameable by copying sophisticated scaffolding.

**False negatives — the private contributor.** Someone who's spent two years doing serious work in a closed company repo, or contributing to internal tooling, is invisible to this system. The score will understate them, sometimes badly. This is the most common failure for experienced candidates.

**Adversarial inflation.** A candidate who knows the scoring model could create repos specifically designed to trigger concept matches — adding packages they've never actually used, generating folder structure that signals architectural thinking they don't have. It's a higher bar than keyword stuffing a CV, but it's not impossible.

**Domain bias.** Frontend and fullstack roles are heavily represented in the role definitions, because that's where public GitHub activity is a reasonable proxy for real work. Backend is thinner. Systems, embedded, ML, and data engineering are mostly unmodeled — not because the engine can't handle them, but because the role files don't exist yet.

---

### Integrity by Design

The scoring model is hard to manipulate quietly. The candidate's GitHub is public. The role definitions are open markdown files. The scoring engine is open source. A TA agency can't lower the bar on a role spec without the hiring company running their own scan and seeing the discrepancy. Score inflation requires corrupting all three simultaneously, with no black box to hide it in.

Community-maintained role files make it harder still — no single actor controls what "good" looks like for a given role.

---

### The API

The REST API is built with Hono and runs on Node. It's the core of the system — the UI and MCP server are both clients of it.

**Candidates** — full CRUD plus triggering analysis jobs. `POST /candidates/:id/jobs` kicks off a scoring run against the candidate's GitHub profile. Results stream back in real time over SSE at `/jobs/:id/stream`, so the UI can show live progress without polling.

**Reports** — a completed analysis stored as a snapshot. Each report includes the fit score, best-fit role, recommendation, trajectory data, concept coverage, Lighthouse scores (if the run included them), activity stats, and a breakdown of the top repositories that contributed to the score.

**Roles** — `GET /roles` returns every available role slug. `GET /roles/:role/candidates` returns a leaderboard of all candidates scored against that role, sorted by fit score. This is how the role comparison view works.

**Openings** — job postings stored in the database. Each opening has a title, a role slug, and optional fields for location, work type, and description. They act as the entry point for outbound sourcing.

**Sourcing** — `POST /openings/:id/source` starts a GitHub search job for a given opening, looking for users who match the role's expected profile. Progress streams back over SSE. Results land in `GET /openings/:id/candidates`, ranked by fit score.

The design is intentionally composable. An external ATS can push openings via `POST /openings/batch` with an `external_id` for idempotency, and the engine handles the rest. The screener-ui is a reference implementation of what you can build on top of it.

---

### The UI

The screener-ui is a React SPA. It's not the product — the API is — but it shows what the data actually looks like when it's surfaced to a recruiter.

**Dashboard** — a grid of candidate cards, each showing the latest fit score, best-fit role, and recommendation badge. Green means Interview, everything below that is Pass.

**Candidate detail** — name, GitHub link, graduation date, location, notes, and a re-analyze button. Below that, a score history chart if the candidate has been analyzed more than once, and a list of all past reports. The chart exists because a single score is a snapshot; the trend is the signal.

**Report view** — the main output. It shows the full role fit breakdown across every track and seniority level, a trajectory graph of how the candidate's work has evolved over time, the matched and missing concepts for their best-fit role, Lighthouse performance scores if the run included them, an activity panel with commit patterns and language distribution, and a list of the top repositories flagged for manual review. There's also a button to copy an AI prompt pre-loaded with the report data, and a PDF export.

**Role leaderboard** — a ranked table of all candidates against a selected role. A dropdown switches between roles. Scores render as a colored progress bar so the comparison is immediate. The 50% threshold — the Interview line — is visible without having to do any arithmetic.

**Openings** — create a job opening with a title, role, and optional location/work type. Once created, hit "Start Sourcing" to kick off a GitHub search. A live progress screen shows users found and scored in real time as the job runs. When it finishes, the ranked results appear as a candidate table on the opening's detail page.

---

### Two Audiences

The tool is framed as a recruiter tool, but candidates can use it too.

Run it against your own GitHub profile and a target role. The concept coverage output tells you exactly which signals are missing — not in vague terms like "improve your testing skills," but specifically: no testing library detected in any repo, or routing present in one project but not evidenced at scale.

That's a different kind of feedback than a rejected application. It's a gap map you can act on.

The trajectory signal matters here too. If your recent repos are less complex than your older ones, the tool catches it. If you've been building seriously for the last six months, it shows that — even if your CV doesn't reflect it yet.

---

### What This Is Not

The tool doesn't make hiring decisions. It doesn't evaluate communication, how someone handles ambiguity, or what they're like in a code review. It doesn't know about private contributions, conference talks, or a decade of systems work done entirely in closed repos.

It's an early-stage signal model. Its job is to improve the quality of information before a human review, not replace it.

The interview dynamic shifts: instead of spending the first half establishing whether someone can code, you already have a picture of what they've built and how they've grown. The interview becomes about depth, communication, and the things that genuinely don't show up in a repository.

---

### The Agentic Shift

As agentic coding tools become standard, two things happen simultaneously.

The volume of code on GitHub increases sharply. More people build more things, faster. The presence of a React app or a REST API in someone's repos stops being a differentiating signal — it becomes the floor. Tools that read keyword presence or dependency lists will get noisier as the pool grows.

At the same time, the gap between developers who use AI as a force multiplier and those who outsource their thinking to it becomes visible in the work. Someone who directs an agent to build a scoring engine with documented failure modes, architectural separation, and a full test suite made real decisions along the way. Someone who prompted their way through a boilerplate tutorial didn't. That difference shows up in folder structure, commit patterns, and project complexity — exactly the signals this tool reads.

The ATS dropout problem also scales. As self-taught and non-traditional developers proliferate — people who learned by building with AI tools rather than through formal education — more capable candidates will have less paper trail. The credential gap widens while the ability gap narrows.

A tool that reads structural quality and trajectory rather than credentials is better positioned in this environment, not worse. The noise increases, but so does the value of a signal that cuts through it.

---

### The Bigger Question

ATS systems don't screen for engineering ability. They screen for CV keywords that have historically correlated with people who had the opportunity to accumulate the right credentials.

If that correlation is weaker than assumed — especially at the junior level, where work history is short or nonexistent — then the cost of using keyword filters isn't just inefficiency. It's systematically missing a large chunk of the candidate pool before a human ever looks.

Staged.dev is an experiment in whether structured artifact signals are a better proxy. It gets some candidates wrong. But it gets them wrong for legible reasons, and the reasons are checkable — the role definitions are readable, the scoring weights are visible, and the GitHub repos it's reading are public. That's a different kind of wrong than a black-box filter no one can inspect.

---

The scoring engine, role definitions, and full API are open source.
If you're building something where candidate signal matters, or you want to run it against your own GitHub profile, it's all there.

[View on GitHub](https://github.com/Wajkie/screw-u-ats)
[PoC Frontend](https://stageddev.netlify.app/)

*I welcome feedback, role file contributions, and brutal honesty.*

