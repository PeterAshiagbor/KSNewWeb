# Deep scan — `ortira/kickstart-manifesto`

**Scope:** full-repo scan beyond the duplication audit — security, runtime correctness, performance,
accessibility, and SEO. Four independent review passes plus a real build of the project.
**Companion to:** `kickstart-manifesto-duplication-audit.md` (structural duplication, 34 findings).
**Findings here:** 33, ranked by severity within each section.

Line references are to the archive as supplied and will drift as the repo moves.

---

## Toolchain ground truth (actually run, not inferred)

| Check | Result |
|---|---|
| `vite build` | ✅ Succeeds — **one 727 KB JS chunk (226 KB gz)**; Vite itself warns about missing code-splitting |
| `tsc -b` | ✅ Clean (but `strict: false`, so this proves little) |
| `vitest run` | ✅ 1/1 — the only test is `expect(true).toBe(true)` |
| `eslint .` | ❌ **3 errors**, 7 warnings — two errors in unused shadcn files, one `require()` in `tailwind.config.ts:90` |
| `playwright test` | ❌ Cannot run — config imports `lovable-agent-playwright-config`, which is in neither `dependencies` nor `devDependencies` |

Initial payload measured from the build: **~8.9 MB** (5.6 MB hero video + 2.8 MB PNG + 552 KB logos
+ 727 KB JS + fonts) — before the 2.5-second loading gate even lifts.

---

## 1. Security

The core access-control design is **sound**: RLS is enabled on both tables, no policy grants `anon`
any read, the allowlist's only SELECT policy is self-row-only (no email enumeration), the dead
`has_perks_access()` function has EXECUTE revoked from all client roles, and the `renderHow()`
linkifier cannot emit non-`http(s)` URLs (no XSS — verified). Residual risk is abuse- and
config-dependent, not a data breach:

### S1 · MEDIUM — Open signup: anyone can trigger magic-link emails and create auth users
`src/pages/Perks.tsx:56-61` — `signInWithOtp` is called without `shouldCreateUser: false`
(Supabase's default is `true`, confirmed absent by grep). Anyone with the public site — or the
committed anon key — can fire magic-link emails at arbitrary addresses (spam/email-quota burn,
spoofed-sender vector) and create unbounded rows in `auth.users`. The allowlist still blocks data
access, so this is abuse/DoS, not a leak.
**Fix:** `options: { shouldCreateUser: false }` + enable Supabase Auth rate limiting/CAPTCHA.

### S2 · MEDIUM (config-dependent) — RLS trusts the JWT `email` claim unconditionally
Migrations `…094611` and `…094803` gate everything on
`lower(auth.jwt() ->> 'email') = lower(allowlist.email)`. Safe for magic-link (clicking proves inbox
ownership) — but if the Supabase project ever enables email/password signup with confirmations off,
an attacker could register an allowlisted address with their own password and pass the check without
owning the inbox.
**Fix:** confirm "Confirm email" is enforced project-wide; disable unused auth providers.

### S3 · LOW — `.env` committed; `.gitignore` doesn't cover it
Anon key + project ref + URL are publishable by design (they ship in the bundle), so exposure alone
is not a breach — but it feeds S1 directly, and the next value added to that file gets committed too.
**Fix:** gitignore `.env`, commit `.env.example`.

### S4 · LOW — Real PII committed: `.lovable/plan.md`
Contains a real user's personal email, their allowlist status, and sign-in history. No build function.
**Fix:** delete + gitignore `.lovable/`; the address remains in git history if the repo is public.

### S5 · INFO — External origins, no CSP
Google Fonts (no SRI) and the OG image served from a third-party `gpt-engineer-file-uploads` bucket.
No external scripts, so supply-chain exposure is minimal. `dangerouslySetInnerHTML` exists only in
the unused shadcn chart component — not exploitable.

---

## 2. Runtime correctness

### C1 · HIGH — Perks auth race: "Verifying access…" can hang forever after magic-link sign-in
`src/pages/Perks.tsx:310-368` — the verify effect depends on the `session` **object**, and its
cleanup sets `cancelled = true`. On magic-link redirect, `onAuthStateChange` and `getSession()` each
deliver a *different object identity* for the same user. Run #1 sets `checkedForUserId` and starts
the RPC; the second `setSession` re-fires the effect, whose cleanup cancels run #1 mid-flight; run
#2 early-returns because the ref already matches. `accessState` sticks on `"checking"` — the user
who just clicked their sign-in link sits on a spinner with no retry control.
**Fix:** depend on `session?.user?.id`, and reset the ref in cleanup when the check didn't complete.

### C2 · HIGH — The Apply funnel dead-ends: "Join the waitlist" is `href="#"`
`src/pages/Apply.tsx:193-200` — the page's primary CTA scrolls to top and joins nothing (the code
comment admits the form embed is missing). **Prospective applicants cannot apply.** Likewise all ten
backer LinkedIn links are `"#"` with `target="_blank"` (`src/components/Backers.tsx:7-16`) — each
opens a new tab of the same site.

### C3 · HIGH — No scroll reset on route change
React Router v6 doesn't reset scroll and no `ScrollToTop` exists. The main entry to `/apply` is the
envelope at the **bottom** of the home page — users land on Apply scrolled deep into the page.
`scroll-behavior: smooth` on `html` means the eventual fix must scroll with `behavior: "auto"`.

### C4 · HIGH — Missing env vars white-screen the whole site, not just Perks
`src/integrations/supabase/client.ts:5-11` — `createClient` runs at module scope and Perks is
statically imported by `App.tsx`, so it executes on every page load. Deploy anywhere that doesn't
inject `VITE_SUPABASE_URL` and the homepage throws during module evaluation. Works today only
because `.env` is committed.
**Fix:** `React.lazy` the Perks route (also the biggest bundle win) or validate env at startup.

### C5 · MEDIUM — Flip-card content unreachable on touch
`src/components/AlumniGrid.tsx:64-108` — company descriptions are revealed only by `:hover`; no
click, focus, or keyboard path. On phones and tablets the back face is effectively invisible. (The
`!important` vs inline-transform CSS interplay is correct — this is purely an input-modality bug.)

### C6 · MEDIUM — NotFound links home with `<a href="/">`
Full document reload → replays the 2.5 s loading gate. Should be `<Link to="/">`.

### C7 · LOW — LoadingScreen digits reshuffle mid-fade
`Math.random()` in render re-rolls all ~40 digits when the fade-out state flips. Memoize.

### C8 · LOW — SideNav goes stale past "Apply"; marquee loop has a ~5 px seam
All nine observed section ids exist (verified) — but `EmailCTA` has no id, so "Apply" stays
highlighted at the bottom of the page. The Partners marquee period (3 copies) undershoots by ⅓ gap
each cycle.

Explicitly checked and **not** bugs: the Apply/Fellows stat-strip border ternaries (all four cells
trace correctly at both breakpoints), the `{...fadeUp}` + explicit `transition` spread ordering, the
flip-card CSS specificity, and the vitest include pattern.

---

## 3. Performance

### P1 · CRITICAL — 5.6 MB hero video at ~8.9 Mbps, autoplaying at full viewport
`public/hero-bg.mp4`: 1920×1072, 5 s loop, no `poster`, default `preload="auto"` — it downloads
immediately on every visit and competes with JS and fonts. On iOS Low Power Mode autoplay is refused
and, with no poster, the hero shows nothing but the overlay.
**Fix:** re-encode ~720p CRF 28-32 (expect 400-800 KB, ~90 % smaller), add poster + `preload="metadata"`.

### P2 · CRITICAL — 2.8 MB PNG as a section background
`src/assets/backers-bg.png` (1071×1920 photographic PNG) sits under a 40 % overlay where fine detail
is invisible — wrong codec by ~10×, and it loads eagerly despite being several viewports below the
fold. **Fix:** WebP/AVIF q≈60 → 150-300 KB, lazy-applied.

### P3 · HIGH — The 2.5 s loading gate is a self-inflicted Core Web Vitals "Poor"
A pure `setTimeout` uncorrelated with any real loading: it delays fast connections without helping
slow ones, pushes real-world LCP to 4-6 s+, and hides the hero's entrance animations, which finish
behind the overlay and are never seen. **Fix:** gate on actual asset readiness capped ≈800 ms, or
first-visit-only.

### P4 · HIGH — No code splitting; Supabase ships to every visitor
All five routes statically imported; zero `React.lazy` in the codebase. Every homepage visitor
downloads and parses the Supabase client plus three pages they'll likely never visit — the build
confirms one 727 KB chunk. `@tanstack/react-query` is instantiated in `App.tsx` yet **zero**
`useQuery`/`useMutation` calls exist — ~13 KB gz of dead provider. **Fix:** lazy routes + delete
react-query.

### P5 · MEDIUM — 552 KB of logos rendered at ≤48 px
Partner logos up to 800×800 rendered at 28-40 px; alumni up to 1104 px wide rendered at 48 px.
Resize to 2× display size + WebP → ~50-60 KB. The marquee also renders 8 logos as 72 DOM nodes in
three permanently-animating rows (2× duplication suffices; pause off-screen).

### P6 · MEDIUM — No `width`/`height`, `loading="lazy"`, or `srcset` on any image
Zero instances across `src/` (verified). Hero logo reflows the hero text when it decodes; all
below-fold images download eagerly.

### P7 · LOW — Fonts and stray serif
Six render-blocking Google-Fonts files; Mono isn't used above the fold. `ApplyCTA` uses
`font-serif` with no serif configured → silent Times fallback on the envelope letter, almost
certainly unintended.

---

## 4. Accessibility

### A1 · CRITICAL — Hover-only flip cards exclude keyboard and touch users (WCAG 2.1.1)
Same root cause as C5: plain `<div>`, no `tabindex`, no toggle. Most of a marketing site's audience
is on mobile — they never see the company descriptions.
**Fix:** make the card a `<button aria-expanded>`, flip on focus/click as well as hover.

### A2 · CRITICAL — There is no navigation to `/fellows` or `/perks` at all — on any device
Verified by grep: **nothing anywhere links to them**; they exist only as route definitions. SideNav
is `hidden md:flex` (mobile has zero nav UI) and only scrolls homepage sections anyway. The pages
are reachable solely by typing the URL — which is also SEO finding E2.
**Fix:** real header/footer navigation on all breakpoints.

### A3 · HIGH — `prefers-reduced-motion` respected nowhere (WCAG 2.3.3)
Zero hits across the codebase (verified): framer-motion everywhere, three infinite marquees, flip
cards, the matrix loading screen, autoplaying video, smooth scroll.
**Fix:** `<MotionConfig reducedMotion="user">` + one CSS media block killing marquee/matrix/smooth-scroll.

### A4 · HIGH — LoadingScreen: 4 Hz flashing overlay with no accessible semantics
`matrix-flicker` runs at 0.25 s — four opacity flashes per second, above the 3 Hz general-flash
threshold (small dark-red area mitigates but doesn't eliminate the photosensitivity concern, and on
mobile the grid is a meaningful viewport fraction). No `role="status"`/`aria-busy`; focus is not
trapped, so keyboard users can Tab into the hidden page underneath.

### A5 · MEDIUM — Contrast failures (computed)
- **White on red = 4.00:1 — fails AA normal text.** Used on the 10 px "APPLY" seal, the Substack
  button, and the waitlist button. Darken the button red to ≈#E60000 or use near-black text.
- **`text-muted-foreground/50` = 2.27:1 — fails badly** ("London, UK", Footer).
- **Placeholder at /60 = 2.78:1 — fails** (Perks email input).
- For the record: primary red on background = **4.95:1** (passes AA — though pure red collapses for
  protanopia), muted-foreground = 5.89:1 (pass). Backers glass cards over a photo are unverifiable
  by design — give them an opaque ground.

### A6 · MEDIUM — Custom cursor forced on `*`
Links and buttons never show `pointer` (except one stray `cursor-pointer` on the envelope letter),
inputs never show the I-beam, and custom cursors ignore OS cursor-size/contrast settings.
**Fix:** scope to `body`, restore `pointer`/`text` on interactive elements.

### A7 · MEDIUM — Ten nameless icon-only links; button nested inside a link
Backers' arrow links have no `aria-label` (Fellows does this correctly — copy it). The envelope
CTA nests a `<button>` inside `<Link>` — invalid HTML; make the seal a `<span aria-hidden>` and give
the Link an `aria-label`.

### A8 · MEDIUM — Index page has no `<h1>`
The brand is an `<img>`, the tagline a `<p>`; every section starts at h2. Fellows and Perks skip
h1→h3. Wrap the hero logo + tagline in an `<h1>`.

### A9 · LOW — Perks collapsible lacks `aria-controls`; decorative video not `aria-hidden`; default
focus rings are low-visibility on #0A0A0A (add a `:focus-visible` ring in `--ring`).

---

## 5. SEO

### E1 · HIGH — Client-only SPA behind a fake loading gate
Served HTML is an empty `<div id="root">`; the 2.5 s timer degrades the render snapshot for
Googlebot and real users alike; non-Google crawlers (most AI/social crawlers) see nothing. The site
is fully static content — trivially prerenderable (`vite-ssg` / prerender plugin).

### E2 · HIGH — `/fellows` and `/perks` are orphan pages
No internal links (see A2) and no sitemap → effectively unindexable.

### E3 · MEDIUM — One set of meta tags for all routes; no canonical
No head management anywhere: `/apply`, `/fellows`, `/perks` all present the homepage title and
description in search results and shares. No `rel="canonical"`, no `og:url`.

### E4 · MEDIUM — OG image lives in the site-builder's bucket
`storage.googleapis.com/gpt-engineer-file-uploads/…` — not brand-controlled, can vanish, and leaks
the build tool. Self-host `/og.png` with explicit dimensions.

### E5 · LOW — No `sitemap.xml`; robots.txt has no `Sitemap:` line. Four routes — a five-minute fix.

The marquee also announces each partner logo up to 24 times to screen readers; mark the rows
`aria-hidden` and provide one visually-hidden partner list.

---

## If only five things get done

1. **Fix the Apply funnel** (C2) — the site's entire purpose currently dead-ends at `href="#"`.
2. **Remove or cap the loading gate and compress the two big assets** (P1-P3) — takes the initial
   payload from ~8.9 MB to ~1.3 MB and LCP from 4-6 s to well under 2.5 s.
3. **Add real navigation** (A2/E2) — un-orphans two pages for users and crawlers at once.
4. **`shouldCreateUser: false` + auth rate limiting** (S1) and confirm email-confirmation is on (S2).
5. **Fix the Perks sign-in race** (C1) — the paying-off moment of the magic-link flow currently
   hangs on a timing coin-flip.

---

*Deep scan performed against an archive of `ortira/kickstart-manifesto` (main): four independent
review passes (security, correctness, performance, accessibility/SEO), key claims cross-verified by
grep and a real `vite build` / `tsc` / `vitest` / `eslint` run. 33 findings.*
