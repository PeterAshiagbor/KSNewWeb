# Kickstart SEO pack: ready-to-apply changes + Google Search Console setup

**Target repo:** `ortira/kickstart-manifesto` (Vite + React SPA). Apply these in that repo. File
paths below are relative to its root.

**Goal:** when someone searches for Kickstart (the founder community, the fellows, the programme),
Google finds the right pages, shows the right title and description, and recognises Kickstart as
one organisation.

**Assumed domain:** `https://kickstartglobal.com`. If the site is deployed anywhere else, change
`SITE_URL` in step 2 and every URL in steps 1, 5 and 6 to match.

> **A realistic expectation for brand searches.** "Kickstart" on its own is a crowded term
> (Kickstarter, the UK government's Kickstart Scheme, a dozen gyms and apps). GSC doesn't make you
> rank. It gets pages indexed and shows what people actually search. The changes below make sure
> Kickstart **wins the searches that are specifically about it**: "Kickstart Global", "Kickstart
> London", "Kickstart founders", "Kickstart fellowship", "Kickstart fellows", "Kickstart 0% equity".
> Winning bare "kickstart" then comes down to links and mentions from elsewhere (press, university
> societies, partners, LinkedIn, Substack), not on-site changes.

---

## Why the site currently underperforms in search

From the deep scan (`docs/kickstart-manifesto-deep-scan.md`):

- **`/fellows` and `/perks` are linked from nowhere.** Google can only find them through a sitemap
  or a link, and the site has neither.
- **Every route has the homepage's title and description.** `/apply` and `/fellows` appear in
  search results as duplicates of the homepage.
- **No canonical URL, no structured data, no sitemap.** Google has to guess what the organisation
  is called and which URL is the real one.
- **The homepage has no `<h1>`.** The brand is an image and the tagline is a paragraph.
- **About 8.9 MB loads behind a fixed 2.5-second loading screen.** That gives a "Poor" Core Web
  Vitals score, which is a ranking signal.

---

## 1. `index.html`: canonical, structured data, self-hosted share image

Copy `src/assets/kickstart_full_logo.png` to `public/kickstart-logo.png`. Download the current
`og:image` from the Lovable bucket URL and save it as `public/og.png`.

Then replace the `<meta property="og:image">` and `<meta name="twitter:image">` lines and add the
blocks below inside `<head>`:

```html
<link rel="canonical" href="https://kickstartglobal.com/" />
<meta property="og:url" content="https://kickstartglobal.com/" />
<meta property="og:site_name" content="Kickstart" />
<meta property="og:image" content="https://kickstartglobal.com/og.png" />
<meta name="twitter:image" content="https://kickstartglobal.com/og.png" />

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "Organization",
      "@id": "https://kickstartglobal.com/#organization",
      "name": "Kickstart",
      "alternateName": "Kickstart Global",
      "url": "https://kickstartglobal.com/",
      "logo": "https://kickstartglobal.com/kickstart-logo.png",
      "description": "A London community and venture programme for student founders building in frontier technology. 0% equity.",
      "email": "london@kickstartglobal.com",
      "address": { "@type": "PostalAddress", "addressLocality": "London", "addressCountry": "GB" },
      "sameAs": ["https://kickstartglobal.substack.com"]
    },
    {
      "@type": "WebSite",
      "@id": "https://kickstartglobal.com/#website",
      "name": "Kickstart",
      "alternateName": "Kickstart Global",
      "url": "https://kickstartglobal.com/",
      "publisher": { "@id": "https://kickstartglobal.com/#organization" }
    }
  ]
}
</script>
```

**Extend `sameAs`** with every official Kickstart profile: the Kickstart LinkedIn company page,
Instagram, X, YouTube. This is the strongest on-site signal that ties those profiles and the site
together as one entity. Only list profiles Kickstart actually owns.

The `WebSite` block with `name` and `alternateName` tells Google what site name to show above the
result ("Kickstart" rather than the bare domain).

---

## 2. New file `src/lib/usePageMeta.ts`: unique title, description and canonical per route

No new dependency is needed. Google renders JavaScript and reads the values this sets.

```ts
import { useEffect } from "react";

export const SITE_URL = "https://kickstartglobal.com";

type PageMeta = { title: string; description: string; path: string; noindex?: boolean };

function setMeta(attr: "name" | "property", key: string, content: string) {
  let el = document.head.querySelector<HTMLMetaElement>(`meta[${attr}="${key}"]`);
  if (!el) {
    el = document.createElement("meta");
    el.setAttribute(attr, key);
    document.head.appendChild(el);
  }
  el.setAttribute("content", content);
}

export function usePageMeta({ title, description, path, noindex }: PageMeta) {
  useEffect(() => {
    const url = SITE_URL + path;
    document.title = title;
    setMeta("name", "description", description);
    setMeta("property", "og:title", title);
    setMeta("property", "og:description", description);
    setMeta("property", "og:url", url);
    setMeta("name", "twitter:title", title);
    setMeta("name", "twitter:description", description);
    setMeta("name", "robots", noindex ? "noindex, follow" : "index, follow");

    let canonical = document.head.querySelector<HTMLLinkElement>('link[rel="canonical"]');
    if (!canonical) {
      canonical = document.createElement("link");
      canonical.rel = "canonical";
      document.head.appendChild(canonical);
    }
    canonical.href = url;
  }, [title, description, path, noindex]);
}
```

Call it at the top of each page component. The copy below uses only claims the site already makes.

```ts
// src/pages/Index.tsx
usePageMeta({
  path: "/",
  title: "Kickstart | 0% equity community for student founders in London",
  description:
    "Kickstart is a London community and venture programme for student founders building in frontier tech. 0% equity. Alumni have built $500M+ in companies, including YC- and Techstars-backed startups.",
});

// src/pages/Apply.tsx
usePageMeta({
  path: "/apply",
  title: "Apply to Kickstart | 0% equity founder programme",
  description:
    "Kickstart accepts ~0.1% of applicants: student and recent-graduate founders across the UK already building in AI, biotech, neurotech, climate, defence or deep tech. 0% equity.",
});

// src/pages/Fellows.tsx
usePageMeta({
  path: "/fellows",
  title: "Kickstart Fellows | Cohort B5, 2026",
  description:
    "Meet the 14 Kickstart fellows in cohort B5, building in quantum, AI, robotics, space and neurotech from Imperial, Cambridge, UCL, King's and more.",
});

// src/pages/Perks.tsx (members-only, keep it out of search)
usePageMeta({ path: "/perks", title: "Founder perks | Kickstart", description: "Partner perks for Kickstart founders.", noindex: true });

// src/pages/NotFound.tsx
usePageMeta({ path: "/404", title: "Page not found | Kickstart", description: "This page doesn't exist.", noindex: true });
```

Every claim in this copy already appears on the site: "$500M+ in companies built" comes from
ProofBanner, and the YC and Techstars badges come from the alumni grid (Origami, AudioGen).

---

## 3. `src/components/Hero.tsx`: give the homepage an `<h1>`

Change the `motion.div` that wraps the logo image into a `motion.h1`. Nothing changes visually.

```tsx
<motion.h1
  initial={{ opacity: 0, y: 30 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: 0.8, ease: "easeOut" }}
>
  <img
    src={kickstartLogo}
    alt="Kickstart"
    width={850}
    height={94}
    fetchPriority="high"
    className="w-[80vw] md:w-[50vw] lg:w-[40vw] h-auto"
  />
</motion.h1>
```

`width` and `height` reserve the space so the hero text doesn't jump when the logo loads.

---

## 4. `src/components/Footer.tsx`: real links to every public page

This fixes the orphan pages for crawlers and for mobile visitors, who currently have no navigation
at all. It's a small, visible addition in the existing footer style.

```tsx
import { Link } from "react-router-dom";
import kickstartLogo from "@/assets/kickstart_full_logo.png";

const links = [
  { to: "/", label: "Home" },
  { to: "/apply", label: "Apply" },
  { to: "/fellows", label: "Fellows" },
];

const Footer = () => (
  <footer className="py-12 px-6 md:px-16 lg:px-24 border-t border-border">
    <div className="flex flex-col md:flex-row md:items-center md:justify-between gap-6">
      <img src={kickstartLogo} alt="Kickstart" width={850} height={94} className="h-8 w-auto" />
      <nav aria-label="Footer" className="flex gap-6">
        {links.map((l) => (
          <Link key={l.to} to={l.to} className="text-sm text-muted-foreground hover:text-foreground transition-colors">
            {l.label}
          </Link>
        ))}
      </nav>
      <div className="flex flex-col md:items-end gap-1">
        <a href="mailto:london@kickstartglobal.com" className="text-sm text-muted-foreground hover:text-foreground transition-colors">
          london@kickstartglobal.com
        </a>
        <p className="text-xs text-muted-foreground">London, UK</p>
      </div>
    </div>
  </footer>
);

export default Footer;
```

"London, UK" also moves from `text-muted-foreground/50`, which fails contrast at 2.27:1, to
`text-muted-foreground`, which passes at 5.89:1.

`/perks` is deliberately left out of the footer because it's invite-only.

---

## 5. New file `public/sitemap.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url><loc>https://kickstartglobal.com/</loc></url>
  <url><loc>https://kickstartglobal.com/apply</loc></url>
  <url><loc>https://kickstartglobal.com/fellows</loc></url>
</urlset>
```

## 6. `public/robots.txt`: point crawlers at the sitemap

Add this as the last line:

```
Sitemap: https://kickstartglobal.com/sitemap.xml
```

Don't add `Disallow: /perks`. Google has to be able to crawl the page to see its `noindex` tag.

---

## 7. Two bigger levers, each worth its own PR

- **Prerender the three public routes** with `vite-ssg` or a prerender plugin. The server currently
  sends an empty `<div id="root">`. Google renders JavaScript, but most other crawlers (Bing's
  lighter passes, LinkedIn, Slack, WhatsApp and AI search engines) see nothing. The site is
  entirely static content, so prerendering is straightforward and gives the largest SEO gain after
  steps 1 to 6.
- **Core Web Vitals.** Cap the loading screen (`App.tsx`, currently a fixed 2.5 s) or show it only
  on a visitor's first visit. Re-encode `public/hero-bg.mp4` (5.6 MB) and convert
  `src/assets/backers-bg.png` (2.8 MB) to WebP. Together those take the first-visit page weight
  from about 8.9 MB to about 1.3 MB. The deep scan has the details.

---

## Google Search Console: setup and the weekly routine

### One-time setup (about 20 minutes, needs access to the domain's DNS)

1. **search.google.com/search-console → Add property → Domain** → enter `kickstartglobal.com`.
   Add the TXT record Google gives you at the domain's DNS provider and click **Verify**. A Domain
   property covers `www`, the bare domain and every subdomain in one place.
2. **Sitemaps** (left menu) → submit `sitemap.xml`. It should show **Success** and "3 discovered
   pages" within a day.
3. **URL Inspection** (top search bar) → paste each of `/`, `/apply` and `/fellows` → **Test live
   URL**, check that the rendered screenshot shows real content and not a blank page or the
   loading screen, then **Request indexing**.
4. **Settings → Users and permissions** → add whoever maintains the site as a Full user, so the
   property doesn't depend on one person's login.
5. Optional: link GSC to Google Analytics, if Kickstart uses GA, for landing-page conversion data.

### What to check, and when

| When | Report | What to look for |
|---|---|---|
| Week 1 | **Pages** | `/`, `/apply` and `/fellows` listed as *Indexed*. `/perks` listed as *Excluded by 'noindex' tag* (that's intended). |
| Week 1 | **Enhancements → Organization** (if shown) | Structured data detected with no errors. You can also paste the homepage into search.google.com/test/rich-results. |
| Weekly | **Performance → Search results** | Add a **Query** filter, *Custom (regex)* `kickstart`, to see every brand search. Sort by Impressions. A query with many impressions and few clicks means the title or description isn't matching what that searcher wanted, so rewrite the copy for that page in step 2. |
| Weekly | **Performance**, filter by **Page** | Which queries bring people to `/apply` and `/fellows`. Those are the phrases to reuse in each page's copy. |
| Monthly | **Core Web Vitals** | Mobile URLs moving from *Poor* to *Good* after the step 7 asset fixes. |
| Monthly | **Links** | Which sites link to Kickstart. New university-society, press and partner links are what move bare "kickstart" rankings. |

### Off-site work that matters for brand searches

- Ask every partner, sponsor, university society and alumni company that mentions Kickstart to
  link to `https://kickstartglobal.com/`, using the name "Kickstart" or "Kickstart Global".
- Use one name everywhere. Pick "Kickstart" or "Kickstart Global" and keep LinkedIn, Substack, the
  site title and press mentions consistent, so Google treats them as the same organisation.
- Link back to the site from the Substack "About" page, the LinkedIn company page and every
  fellow's profile that mentions Kickstart.
