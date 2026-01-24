# Build Your Own Minimal Developer Portfolio (React)

**Frontend-Only, API-Driven, Step-by-Step Guide**

---

## Phase 1 — Mental Model (Do This Before Writing UI)

### Step 1.1 — Treat this as a “Reader-first” product

Your portfolio is not a dashboard.
It’s a **reading experience**.

Everything you build should optimize for:

* Scannability
* Calm reading
* Zero cognitive load

This is why:

* No cards
* No grids for blogs
* No heavy shadows
* No loud colors

Keep this mental rule active throughout development.

---

### Step 1.2 — Identify your only 3 dynamic domains

From the frontend perspective, your app only has:

1. **Static content**

   * Name
   * Intro
   * Work experience
   * Social links

2. **Projects (API-driven)**

   * List view
   * Detail view

3. **Blogs (API-driven)**

   * List view
   * Detail view

Everything else is layout.

---

## Phase 2 — App Skeleton (Without Touching Data)

### Step 2.1 — Build the layout shell first

Before thinking about APIs:

* Navbar (top)
* Main content area
* Footer (bottom)

**Why first?**
Because this shell never changes across pages.

Your screenshots already prove:

* Same navbar everywhere
* Same footer everywhere
* Only content changes in the middle

Once this is stable, everything else becomes predictable.

---

### Step 2.2 — Lock spacing and width rules

Decide once:

* Max content width
* Vertical spacing between sections
* Divider style

This ensures:

* Blogs and projects look consistent
* Dark mode doesn’t drift visually

After this step, **never change spacing again unless broken**.

---

### Step 2.3 — Define typography hierarchy (non-negotiable)

Your UI relies heavily on typography, so mentally define:

* Page title (Blog / Project name)
* Section heading (Overview, Tech Stack)
* Body text
* Muted meta text (dates, summaries)

This matters because:

* API data will flow into these slots
* You don’t want to rethink typography per page

---

## Phase 3 — Routing & Page Contracts

### Step 3.1 — Define page responsibilities (contracts)

Each page must have **exactly one responsibility**.

| Page           | Responsibility              |
| -------------- | --------------------------- |
| Home           | Summary + navigation        |
| Projects       | Show all projects           |
| Project Detail | Show **one** project deeply |
| Blog           | Show all blog posts         |
| Blog Detail    | Show **one** blog deeply    |

No page should fetch data it doesn’t display.

---

### Step 3.2 — Decide route semantics early

Even without code, mentally lock routes:

* `/projects` → list
* `/projects/:slug` → detail
* `/blog` → list
* `/blog/:slug` → detail

This helps later when:

* Linking from home → blog
* Linking from blog → back
* SEO decisions

---

## Phase 4 — API Integration Strategy (Frontend Perspective Only)

> We do NOT discuss API shape or structure.

### Step 4.1 — Treat APIs as “content providers”

On the frontend:

* APIs are **read-only**
* No mutations
* No user interaction

This means:

* No loading spinners everywhere
* No optimistic updates
* Simple fetch → render flow

---

### Step 4.2 — Decide fetch timing per page

Very important thinking step:

| Page           | When to fetch                       |
| -------------- | ----------------------------------- |
| Home           | No API (or minimal highlight fetch) |
| Projects       | Fetch on page load                  |
| Project Detail | Fetch on page load                  |
| Blog           | Fetch on page load                  |
| Blog Detail    | Fetch on page load                  |

You are **not building infinite scroll**.
Keep it simple and predictable.

---

### Step 4.3 — Define loading philosophy

Your design is calm, so:

* Avoid skeleton overload
* Prefer subtle placeholders
* Avoid jumping layouts

Consistency > fancy loaders.

---

## Phase 5 — Build Pages One by One (Order Matters)

### Step 5.1 — Home Page (Static First)

Build Home **without APIs** initially.

Sections:

1. Intro text
2. Work experience
3. Projects (static placeholders)
4. Writing (static placeholders)
5. Connect

Why static first?

* Layout correctness
* Typography consistency
* Spacing validation

Later you can swap placeholders with API-driven content.

---

### Step 5.2 — Projects List Page

Now introduce APIs.

This page should:

* Fetch projects once
* Render them in a simple vertical list
* Show title + short description
* Link to detail page

**No pagination logic** unless forced.

---

### Step 5.3 — Project Detail Page

This page must feel:

* Editorial
* Calm
* Focused

Mentally break into sections:

1. Title + short description
2. Overview
3. Tech Stack
4. Challenges
5. Links

Each section maps to API fields — nothing extra.

---

### Step 5.4 — Blog List Page

Treat this like a reading index:

* Title
* One-line summary
* Divider
* Repeat

Avoid:

* Cards
* Thumbnails
* Tags (for now)

This matches your screenshots perfectly.

---

### Step 5.5 — Blog Detail Page (Most Important Page)

This page defines your **engineering voice**.

Structure:

1. Blog title
2. Meta info (date)
3. Intro paragraph
4. Section headings
5. Bullet points
6. Footer links

Your job:

* Make long text readable
* Avoid visual fatigue
* Respect whitespace

---

## Phase 6 — Dark Mode Parity

### Step 6.1 — Dark mode must be equal, not secondary

Rule:

> Dark mode is not a theme, it’s a first-class mode.

Checklist:

* Same contrast ratios
* Same spacing
* Same hierarchy
* No color inversions that break reading

Test blogs specifically in dark mode.

---

## Phase 7 — Navigation & Flow

### Step 7.1 — Back navigation consistency

Every detail page must have:

* “Back to Blog”
* “Back to Projects”

Same position, same style, everywhere.

Users should never feel lost.

---

### Step 7.2 — Cross-linking

From:

* Home → Blog
* Home → Projects
* Blog list → Blog detail
* Project list → Project detail

No deep linking surprises.

---

## Phase 8 — Final Polish Checklist

Before calling it “done”:

* Scroll experience feels smooth
* Text line-length is comfortable
* No layout shift during loading
* Mobile readability is acceptable
* Dark mode blog reading feels good

---
