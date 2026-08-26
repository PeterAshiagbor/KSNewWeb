# Duplication audit — `ortira/kickstart-manifesto`

**Scope:** all 145 files of the repository (main branch, as supplied in archive form).
**Stack:** Vite · React 18 · TypeScript · Tailwind · shadcn/ui · Supabase · framer-motion.
**Volume:** 6,539 LOC across `src/`.
**Findings:** 34.

> **No two files in the repository are byte-identical** — every file hash is unique. All duplication
> found is *structural*: repeated markup, repeated configuration, repeated database logic, and the
> same facts about Kickstart typed out in several places that no longer agree with each other.

Line references are to the archive as supplied and will drift as the repo moves.

---

## Headline numbers

| Measure | Value |
|---|---|
| Share of `src/` that nothing reaches | **~64%** (4,163 of 6,539 LOC) |
| shadcn/ui components with no importer | **45 of 49** (3,764 LOC) |
| Runtime dependencies nothing imports (after that deletion) | **37 of 51** |
| Package lockfiles in the repo root | **3** |
| Copies of the same framer-motion reveal block | **9** |
| Inline `fontFamily` styles duplicating global CSS | **27** |

---

## 1. Dead weight shipped with the repo

The single largest item. The project was scaffolded from a full shadcn/ui install and the unused
92% of it was never removed, which also drags in three quarters of the dependency list.

### 1.1 — 45 of the 49 shadcn/ui components have no importer anywhere · 3,764 LOC

Only `tooltip`, `toast`, `toaster` and `sonner` are referenced outside `src/components/ui/`, and all
four are referenced only by `App.tsx` — which mounts them without ever using them (see 1.2).
Everything else — accordion, calendar, carousel, chart, command, dialog, form, sidebar, table,
select and 35 more — is unreachable.

**Fix:** delete every file in `src/components/ui/` that nothing imports. Re-add individual
components with the shadcn CLI if and when a page needs one.

### 1.2 — Two complete toast systems are mounted, and no code raises a toast

`App.tsx:39–40` renders both `<Toaster />` (the Radix stack) and `<Sonner />` — two competing
implementations of the same feature. Grepping all of `src/` outside the toast files themselves
returns **zero** calls to `toast()` or `useToast()`.

- `src/hooks/use-toast.ts` (186 LOC)
- `src/components/ui/toast.tsx` (111)
- `src/components/ui/toaster.tsx` (24)
- `src/components/ui/sonner.tsx` (27)

**Fix:** remove both. If toasts are wanted later, pick one — Sonner is the lighter — and drop the other.

### 1.3 — A three-line file exists only to re-export another file

`src/components/ui/use-toast.ts` imports `useToast` and `toast` from `@/hooks/use-toast` and
immediately re-exports them, creating a second import path for the same two symbols. Nothing
imports it.

**Fix:** delete.

### 1.4 — `NavLink.tsx` reimplements react-router's own NavLink, and nothing uses it · 28 LOC

A compatibility shim restoring the React Router v5 `activeClassName` / `pendingClassName` API on top
of the v6 render-prop API — duplicating a library the project already depends on. No importers.

**Fix:** delete. Use `NavLink` from `react-router-dom` directly if a nav link is ever needed.

### 1.5 — `useIsMobile` is consumed only by the unused sidebar component · 19 LOC

`src/hooks/use-mobile.tsx` has exactly one importer, `ui/sidebar.tsx`, which itself has none. Every
responsive decision in the real site uses Tailwind breakpoints.

**Fix:** deletes automatically with the sidebar component.

### 1.6 — Three quarters of the runtime dependencies serve only unused components · 37 of 51

Once the unimported shadcn files go, 37 of 51 `dependencies` entries have no importer: 25 Radix
packages plus `recharts`, `embla-carousel-react`, `cmdk`, `vaul`, `react-day-picker`,
`react-hook-form`, `react-resizable-panels`, `input-otp` and more.

Four are already dead **today**, before deleting anything: `@hookform/resolvers`, `date-fns`,
`zod`, and `tailwindcss-animate` — the last loaded as a Tailwind plugin whose animation utilities
appear in no class string.

**Fix:** prune `package.json` after the component deletion, then regenerate the lockfile. Install
size and `npm audit` surface both drop sharply.

### 1.7 — Tailwind declares eight `sidebar-*` colours backed by CSS variables that don't exist

`tailwind.config.ts:50–59` maps `sidebar`, `sidebar-foreground`, `sidebar-primary` and five more
onto `--sidebar-background` etc. None of those custom properties is ever defined in
`src/index.css`. Any class using them resolves to an invalid colour.

**Fix:** remove the `sidebar` block, and the `accordion-down` / `accordion-up` keyframes and
animations, which only the unused accordion uses.

### 1.8 — `src/App.css` is a file containing the comment `/* Reset */` · 1 LOC

Nothing imports it; `main.tsx` imports only `index.css`.

**Fix:** delete.

### 1.9 — The custom cursor ships as both PNG and SVG; only the PNG is wired up

`index.css:42` sets `cursor: url('/kickstart_cursor.png')`. `public/kickstart_cursor.svg` is the
same mark in a second format, referenced nowhere. `public/placeholder.svg` — Lovable scaffolding —
is likewise unused.

**Fix:** delete the unused SVGs, or switch the cursor to the SVG and delete the PNG. Not both.

### 1.10 — Three package lockfiles, three possible dependency trees

`bun.lock` (165 KB), `bun.lockb` (247 KB — the binary form of the same thing) and
`package-lock.json` (238 KB) all sit in the repo root. Whichever one CI or a teammate's machine
happens to honour decides which versions get installed, and nothing keeps the three in step.

**Fix:** pick one package manager. Delete the other two lockfiles and gitignore them. If Bun: keep
`bun.lock`, drop the legacy `bun.lockb`.

---

## 2. Repeated markup across components

Every landing section was written by copying the previous one. The shared parts were never factored
out, so a change to the site's rhythm — section padding, scroll animation, heading scale — now has
to be made in seven to nine places by hand.

### 2.1 — The same four-line scroll-reveal block is pasted into nine components · ×9

```jsx
<motion.div
  initial={{ opacity: 0, y: 30 }}
  whileInView={{ opacity: 1, y: 0 }}
  viewport={{ once: true, margin: "-100px" }}
  transition={{ duration: 0.6 }}
>
```

`ProofBanner:12` · `Philosophy:6` · `WhoItsFor:13` · `WhatYouGet:30` · `AlumniGrid:147` ·
`Backers:59` · `Partners:62` · `ApplyCTA:7` · `EmailCTA:7`

**Fix:** export one `revealOnScroll` variant object and spread it — `<motion.div {...revealOnScroll}>`.

### 2.2 — Seven sections repeat the identical section-shell class string · ×7

`py-20 md:py-32 px-6 md:px-16 lg:px-24 border-t border-border` appears verbatim in ProofBanner,
Philosophy, WhoItsFor, WhatYouGet, AlumniGrid, ApplyCTA and EmailCTA. The narrower fragment
`px-6 md:px-16 lg:px-24` — the site's page gutter — occurs **29 times**.

**Fix:** one `<Section>` wrapper carrying the shell plus the reveal from 2.1. Collapses both
findings and makes the gutter editable in one place.

### 2.3 — All three sub-pages carry a hand-copied top bar · ×3

The logo-plus-"← Back" header is nine identical lines in `Apply.tsx:66`, `Fellows.tsx:206` and
`Perks.tsx:404`. Only Perks differs, and only because it spells the font style as a `monoFont`
object rather than inline — same output.

**Fix:** extract `<PageHeader />` next to the existing `<Footer />`, which the same three pages
already share correctly.

### 2.4 — The four-cell stat strip is written twice, and the copies have drifted · ×2

`Apply.tsx:100–129` and `Fellows.tsx:249–275` are the same 27 lines. The differences are accidental:
Apply wraps it in `<motion.div {...fadeUp}>` and Fellows in a plain `<div>` (so the Fellows strip
silently doesn't animate), and Apply's border logic reads `"border-r md:border-r"` where Fellows
reads `"border-r"` — the first a class that redundantly repeats itself.

**Fix:** extract `<StatStrip items={…} />`. Decide once whether it animates.

### 2.5 — Three separate arrays named `stats`, holding overlapping numbers · ×3

`ProofBanner.tsx:3` ($500M+, 21, 0%) · `Apply.tsx:13` (~0.1%, 0%, 21, 800+) · `Fellows.tsx:127`
(14, B5, 2026, Active). Two assert the same two facts — *21 average age*, *0% equity* — in two
different capitalisations of the label.

**Fix:** one `src/content/stats.ts`; pages pick the subset they display.

### 2.6 — `fadeUp` is defined identically in two page files · ×2

Character-for-character the same object literal at `Apply.tsx:6` and `Perks.tsx:19` — and a second,
subtly different preset from 2.1 (`y:20`/`-80px` vs `y:30`/`-100px`). The site has two reveal
animations that look almost but not quite the same.

**Fix:** move both into `src/lib/motion.ts` and decide whether the difference is intentional.

### 2.7 — "Show all on mobile" is implemented three times from scratch · ×3

The same `useState` + `slice(0, n)` + toggle-button pattern in `AlumniGrid.tsx:112` (slice 3),
`Backers.tsx:43` (slice 3) and `Fellows.tsx:200` (slice 6). The three disagree on copy — two offer
"Show less", Fellows offers no way back.

**Fix:** one `useCollapsibleList(items, limit)` hook, or a `<CollapsibleGrid>` that owns the button.

### 2.8 — Three components render every card into the DOM twice and hide one copy with CSS · ×3

AlumniGrid, Backers and Fellows each build a mobile list and a desktop list side by side, switching
with `sm:hidden` / `hidden sm:grid`. On a desktop load Fellows mounts 20 `motion.div` cards to show
14; AlumniGrid mounts 30 flip cards to show 21, each with its own logo `<img>`; Backers mounts 13 to
show 10. Hidden images are still fetched.

**Fix:** render the list once and vary only the grid classes, or gate the slice on a breakpoint
value rather than duplicating the tree.

### 2.9 — The section eyebrow exists in two competing forms, eleven times · ×11

The home page uses a sans variant — `text-xs uppercase tracking-[0.2em] text-primary mb-3
font-medium` — five times (Philosophy, WhatYouGet, AlumniGrid, Backers, Partners). The sub-pages use
a mono variant with different spacing six times (Apply, Fellows, and four places in Perks). Apply
even wraps its version in a local `<Eyebrow>` component the other pages don't import.

**Fix:** promote Apply's `<Eyebrow>` to `src/components/`, give it a `variant` prop if both looks
are wanted, use it in all eleven places.

### 2.10 — Heading, lede and CTA-button class strings repeated verbatim · ×4 / ×3 / ×2 / ×13

- `text-3xl md:text-5xl lg:text-6xl font-bold text-foreground mb-4 tracking-tight` — 4× (AlumniGrid, Backers, EmailCTA, WhatYouGet)
- `text-muted-foreground text-base md:text-lg mb-12 max-w-xl` — 3×
- The uppercase primary CTA button — 2× (Apply's waitlist link, Perks' sign-in submit)
- The decorative red full stop `<span className="text-primary">.</span>` — **13×**

**Fix:** either `@layer components` classes in `index.css` (`.section-title`, `.section-lede`,
`.btn-primary`) or small `<SectionTitle>` / `<Button>` components. The full stop deserves a one-line
`<Dot />`.

### 2.11 — Perks' three states repeat the same shell three times · ×3

`SignInView`, `NotApprovedView` and `VerificationErrorView` each open with the identical `<section>`
class string, the identical `<motion.div {...fadeUp} className="max-w-xl w-full">`, an eyebrow
`<span>` and an `<h1>` at the same size. Only the words differ.
`Perks.tsx:71–82` · `143–154` · `167–178`

**Fix:** one `<GateView eyebrow title>{children}</GateView>`; the three states become three short
call sites.

---

## 3. Styling declared in more than one place

### 3.1 — The two typefaces are re-declared inline 27 times, on top of the global CSS · ×27

`index.css` already sets IBM Plex Sans on `body` and on `h1–h6`, and IBM Plex Mono on `.font-mono`.
Despite that, `style={{ fontFamily: "'IBM Plex Sans', sans-serif" }}` is written out 12 times across
10 components, and the Mono equivalent 15 times across the three pages. Perks alone recognised the
problem and made a local `monoFont` constant — which Apply and Fellows don't use.

- **Sans ×12** — Hero, ProofBanner, Philosophy ×2, WhoItsFor, WhatYouGet ×2, AlumniGrid, Backers, Partners, EmailCTA, SideNav
- **Mono ×15** — Apply ×6, Fellows ×8, Perks ×1

**Fix:** add `fontFamily.sans` and `fontFamily.mono` to the Tailwind theme, then use `font-sans` /
`font-mono` classes and delete all 27 inline styles. Changing the brand typeface becomes a one-line
edit instead of a 27-file sweep.

### 3.2 — The page background colour is expressed three different ways · ×3

`index.html` hard-codes `#0a0a0a` on `<html>` and again on `<body>`; `index.css` defines the same
colour again as the token `--background: 0 0% 4%`. They agree today by coincidence of rounding;
retheming means remembering that two of the three live in a file Tailwind never touches.

Separately, `AlumniGrid` hard-codes `hsl(0 0% 5%)` twice as its card background instead of using
`bg-card` — and `--card` is 7%, so the alumni cards are quietly a different shade from every other
card in the design system.

**Fix:** keep one inline hex in `index.html` (it must work before CSS loads), drop the duplicate,
comment it as mirroring `--background`. Replace AlumniGrid's literals with `bg-card` and decide
which shade is correct.

### 3.3 — The page title and description are each written three times in `index.html` · ×3

`<title>`, `og:title` and `twitter:title` carry the same string; `description`, `og:description` and
`twitter:description` carry another. There is also an `og:image` pointing at a
`gpt-engineer-file-uploads` bucket rather than an asset in this repo, and two stray blank lines
where tags were removed.

**Fix:** low stakes, but if the copy changes all three must change together. Worth a comment marking
them as a set, and worth moving the OG image into `public/`.

---

## 4. Duplication in the Supabase migrations

Seven migrations, all dated the same day, iterating on one question: *is this user allowed to see
perks?* Each attempt was left in place, so the answer is now encoded three times over — and the
abandoned attempts are still live objects.

### 4.1 — `is_on_perks_allowlist()` is defined three times, with different semantics each time · ×3

- `…092632` — `SECURITY DEFINER`, joins `auth.users` on a citext cast
- `…094516` — same join, `lower()` comparison
- `…094611` — `SECURITY INVOKER`, reads the allowlist directly off the JWT claim

Only the last runs. The first two are pure history, but a reader must open all three to know that.

**Fix:** leave shipped migrations alone (rewriting applied history is worse), but add a short
`supabase/README.md` stating which definition is current. Squash before the next environment is
provisioned.

### 4.2 — `has_perks_access(uuid)` is defined twice and is now dead, but still exists in the database · ×2

Created in `…075121`, redefined in `…075214`, orphaned by `…094803` when the perks RLS policy was
rewritten to inline the check instead. That final migration revokes execute on it from every role
but never drops it — so a `SECURITY DEFINER` function reading `auth.users` is still sitting in the
`public` schema with no caller.

**Fix:** add a migration with `drop function if exists public.has_perks_access(uuid);`. An unused
SECURITY DEFINER function is worth removing rather than merely un-granting.

### 4.3 — The allowlist test is written out in three separate objects · ×3

`lower(a.email::text) = lower(auth.jwt() ->> 'email')` appears in the `is_on_perks_allowlist()`
body, in the `"Users can verify their own allowlist row"` policy, and again in the
`"Approved users can view perks"` policy. Change the matching semantics — trimming whitespace, say —
and missing one copy means the RPC and the row policy disagree about who is allowed in.

**Fix:** have both policies call `public.is_on_perks_allowlist()` so the predicate lives in one place.

### 4.4 — The same three-line REVOKE incantation is repeated in four migrations · ×4

`revoke execute … from public / anon / authenticated` is re-issued in `…075151`, `…075214`,
`…094516` and `…094803` — mostly re-revoking what was already revoked, because each
`create or replace` resets the grants and nobody was certain which state they were in.

**Fix:** symptom of the churn above; resolves when the function definitions are squashed to one.

---

## 5. Configuration written twice

### 5.1 — `vite.config.ts` and `vitest.config.ts` restate the same plugin and alias setup · ×2

Both declare `plugins: [react()]` and both resolve `"@" → ./src`. The Vitest config silently misses
Vite's `dedupe` list and the `componentTagger` plugin, so tests run against a slightly different
module graph than the app does.

**Fix:** delete `vitest.config.ts` and move the `test` block into `vite.config.ts` (Vitest reads it
natively), or use `mergeConfig` to extend the real config rather than restating it.

### 5.2 — Three tsconfigs repeat the same compiler options instead of extending one base · ×3

`paths`, `skipLibCheck`, `noImplicitAny: false`, `noUnusedLocals: false` and
`noUnusedParameters: false` are written into both `tsconfig.json` and `tsconfig.app.json`; the
bundler-mode block (`moduleResolution`, `isolatedModules`, `allowImportingTsExtensions`,
`moduleDetection`, `noEmit`) is repeated in `tsconfig.app.json` and `tsconfig.node.json`. The two
also disagree on `strict` — `false` for app code, `true` for config files.

**Fix:** put shared options in `tsconfig.base.json` and have both project configs `extends` it.

---

## 6. The same facts typed in several places

The category with real business risk. Kickstart's headline claims are hard-coded as strings
scattered through JSX, and some copies have already fallen out of sync.

### 6.1 — Every headline number is hard-coded in three or more places

```
0% equity     ProofBanner:6 · Apply:15 (stat) · Apply:29 (bullet) · Apply:85 (headline)
21 avg age    ProofBanner:5 · Apply:16 (stat) · Apply:22 (bullet)
0.1% accept   Apply:14 (stat) · Apply:86 (headline) · Apply:172 (prose)
Cohort B5     Fellows:129 (stat) · Fellows:228 (eyebrow) · Perks:258 (eyebrow)
14 fellows    Fellows:128 (stat) · Fellows:244 (prose) · FELLOWS.length (button)
```

The fellows count is the clearest illustration: the button already computes `FELLOWS.length`, so
adding a fifteenth fellow updates the button and leaves the stat tile and the sentence both saying 14.

**Fix:** a `src/content/facts.ts` exporting `EQUITY_TAKEN`, `AVG_AGE`, `ACCEPTANCE_RATE`, `COHORT`,
deriving fellow and perk counts from the arrays themselves. Cohort rollover becomes one edit.

### 6.2 — The partner and mentor roster exists in four versions that don't match · ×4

`Partners.tsx` lists 8 logos. `Backers.tsx` lists 10 people whose companies partly overlap (Sequoia
Capital, 1517 Fund) and partly don't (OakNorth, Octopus Energy, J.P. Morgan, Monzo, Wise, OpenAI).
`Apply.tsx:32` names a fifth set in prose — "Sequoia, OakNorth, Octopus, J.P. Morgan, 1517 Fund,
Antler" — where **Antler appears nowhere else in the site**. And `AlumniGrid` uses "Techstars" and
"YC-backed" as company badges, a sixth place the same names are typed.

All ten `linkedin` fields in `Backers.tsx` are the placeholder `"#"`, so every card links to the top
of the page.

**Fix:** one `src/content/network.ts` with a typed list of organisations and people tagged by
relationship (partner / mentor / investor). Every surface reads from it; the Apply prose is
generated from the same list.

### 6.3 — One alumni company appears in two datasets under two spellings · ×2

`AlumniGrid.tsx:53` lists **"InX Tech"** — "Industrial X-ray technology". `Fellows.tsx:35` describes
Aryan Shah building **"InxTech"** — "autonomous underwater vehicles for offshore energy inspection".
Same company, two names, two descriptions that don't obviously describe the same business. A visitor
moving between the pages sees a contradiction.

**Fix:** decide the canonical name and description, put the company in one `companies.ts`, have the
fellow record reference it by id.

### 6.4 — Repeated phrasing inside the copy itself · ×3

`Philosophy.tsx` opens its pull-quote with "Most talented people never compound." and then opens the
very next paragraph with "Most talented people scatter their energy…" — the echo reads as an editing
artefact rather than a rhetorical device. "Building in frontier tech" appears three times across
Apply and Fellows.

`Partners.tsx` also hand-writes its eight partner names into *three shuffled arrays* (`row1`,
`row2`, `row3`) and then triples each row at render time — 8 real partners become 72 DOM nodes and
72 image references.

**Fix:** for Partners, keep one array and shuffle it with a seeded helper per row. For the copy, a
light editorial pass by whoever owns the voice.

---

## 7. Not duplication, but found on the way through

### 7.1 — `.env` is committed and `.gitignore` does not cover it

The file holds `VITE_SUPABASE_PROJECT_ID`, `VITE_SUPABASE_PUBLISHABLE_KEY` and
`VITE_SUPABASE_URL`. These are publishable values that ship in the client bundle anyway, so this is
not an emergency — but the pattern is, because the next secret added to that file will be committed
too. `.gitignore` lists `*.local`, which suggests `.env` was meant to be ignored and never was.

**Fix:** add `.env` to `.gitignore`, commit a `.env.example` with empty values, and confirm no
non-publishable key was ever added to it.

### 7.2 — `.lovable/plan.md` contains a real user's email address and their access state

A saved AI working note debugging one person's perks access. It names their address, states they are
on the allowlist, and reports on their sign-in history. It has no function in the build.

**Fix:** delete the file and add `.lovable/` to `.gitignore`. If the repo is or becomes public, the
address is in the history too.

### 7.3 — The Playwright setup imports a package that isn't installed

`playwright.config.ts` and `playwright-fixture.ts` both import from
`lovable-agent-playwright-config`, which appears in neither `dependencies` nor `devDependencies`.
There are also no `.spec.ts` files, so there is nothing for Playwright to run. Vitest has exactly one
test — `expect(true).toBe(true)`.

**Fix:** either install the package and write a smoke test per route, or delete both Playwright
files. Right now they only look like coverage.

### 7.4 — Generator defaults left in place

`README.md` reads "Welcome to your Lovable project / TODO: Document your project here".
`package.json` is still named `vite_react_shadcn_ts`. `components.json` declares
`baseColor: "slate"` while the actual palette is pure-red-on-near-black.

**Fix:** fifteen minutes. The README in particular is what a new contributor hits first.

---

## What to do first

Ordered by payoff per unit of risk. The first three are close to free and take roughly two thirds of
the codebase off the table before anyone has to make a design decision.

| Effort | Step | Effect |
|---|---|---|
| 30 min | Delete the 45 unimported shadcn files and both toast systems | −4,163 LOC (~64% of `src/`); then prune `package.json` and regenerate one lockfile |
| 10 min | Delete two of the three lockfiles, `App.css`, the unused SVGs, `NavLink.tsx` | Pure removal, nothing to review |
| 15 min | Gitignore `.env` and `.lovable/`; delete `plan.md` | Do this before the repo is shared any wider |
| 2 hrs | Extract `<Section>`, `<PageHeader>`, `<Eyebrow>`, `<StatStrip>`, one motion preset | Collapses findings 2.1, 2.2, 2.3, 2.4, 2.9 in one pass |
| 1 hr | Move typefaces into the Tailwind theme; delete all 27 inline `fontFamily` styles | Brand font becomes a one-line change |
| 2 hrs | Create `src/content/` — facts, network, companies, fellows | Fixes 6.1, 6.2 and 6.3 together |
| 1 hr | Drop dead `has_perks_access`; route both RLS policies through `is_on_perks_allowlist()` | One authorisation rule in one place |
| 30 min | Merge the Vitest config into Vite; add `tsconfig.base.json` | Tests run against the same module graph as the app |

---

*Audit performed against an archive of `ortira/kickstart-manifesto` (main). 145 files · 6,539 LOC in
`src/` · 34 findings · no byte-identical files detected.*
