# Task: SEO & metadata fixes — TULSI + Academy OS landing sites

Two separate repos. **Apply this file to each one**, using the per-site values
in the tables below.

| Site | Repo | Live URL |
|---|---|---|
| TULSI (company) | `tulsi-solutions` | `https://tulsi-solutions.vercel.app` |
| Academy OS (product) | `academy-os-landing` | `https://academy-os-landing.vercel.app` |

Both are Next.js App Router on Vercel. Both currently score **B** with an
**F on structured data**.

Deferred: custom domain, SPF and DMARC records. Those need a purchased domain
and DNS access — see the last section, do not attempt them now.

---

## Task 1 — Metadata (highest impact)

Both sites are missing Open Graph tags, Twitter Card tags, and a canonical URL.
Every link shared on WhatsApp or LinkedIn currently renders as a bare URL with
no title, description or preview image. This is the single most valuable fix
here, because WhatsApp and LinkedIn are the only distribution channels in use.

Edit the root `app/layout.tsx` `metadata` export in each repo.

### TULSI — `tulsi-solutions`

Current title is 62 chars (truncates in search) and meta description is 174
chars (truncates). Shorten both.

```ts
import type { Metadata } from "next";

const siteUrl = process.env.NEXT_PUBLIC_SITE_URL ?? "https://tulsi-solutions.vercel.app";

export const metadata: Metadata = {
  metadataBase: new URL(siteUrl),
  title: "TULSI — Software built around how businesses work",
  description:
    "We study how an industry actually works, then build the software it should have had. Our first: academies.",
  alternates: { canonical: "/" },
  openGraph: {
    type: "website",
    siteName: "TULSI",
    url: siteUrl,
    title: "TULSI — Software built around how businesses work",
    description:
      "We study how an industry actually works, then build the software it should have had. Our first: academies.",
    images: [{ url: "/og.png", width: 1200, height: 630, alt: "TULSI" }],
  },
  twitter: {
    card: "summary_large_image",
    title: "TULSI — Software built around how businesses work",
    description:
      "We study how an industry actually works, then build the software it should have had. Our first: academies.",
    images: ["/og.png"],
  },
};
```

Target: title under 60 characters, description under 160.

### Academy OS — `academy-os-landing`

Title (53 chars) and description (129 chars) already pass. Keep the wording,
add the missing OG/Twitter/canonical blocks.

```ts
import type { Metadata } from "next";

const siteUrl = process.env.NEXT_PUBLIC_SITE_URL ?? "https://academy-os-landing.vercel.app";

export const metadata: Metadata = {
  metadataBase: new URL(siteUrl),
  title: "Academy OS — Run your academy from one calm dashboard",
  description:
    "Enquiries, students, batches, fees and attendance — everything a dance, music, art or coaching academy needs, in one place.",
  alternates: { canonical: "/" },
  openGraph: {
    type: "website",
    siteName: "Academy OS",
    url: siteUrl,
    title: "Academy OS — Run your academy from one calm dashboard",
    description:
      "Enquiries, students, batches, fees and attendance — everything a dance, music, art or coaching academy needs, in one place.",
    images: [{ url: "/og.png", width: 1200, height: 630, alt: "Academy OS" }],
  },
  twitter: {
    card: "summary_large_image",
    title: "Academy OS — Run your academy from one calm dashboard",
    description:
      "Enquiries, students, batches, fees and attendance — everything a dance, music, art or coaching academy needs, in one place.",
    images: ["/og.png"],
  },
};
```

**Using `metadataBase` with an env var matters** — when the custom domain
arrives, set `NEXT_PUBLIC_SITE_URL` in Vercel and every absolute URL
(canonical, OG, sitemap) updates without a code change.

### OG image

Create `public/og.png` in each repo, **1200×630px**. Keep it simple: logo,
product name, one line of positioning, brand background. No screenshots of
features that don't exist.

---

## Task 2 — `lang` attribute (Academy OS only)

The audit flags a missing `lang` attribute on `academy-os-landing`. Confirm the
root layout has:

```tsx
<html lang="en">
```

`tulsi-solutions` already declares this correctly — no change there.

---

## Task 3 — Organization schema (fixes the F grade)

Both sites score **F on structured data** because no identity schema exists.
Add JSON-LD to the root layout of each site, inside `<body>`:

### TULSI

```tsx
<script
  type="application/ld+json"
  dangerouslySetInnerHTML={{
    __html: JSON.stringify({
      "@context": "https://schema.org",
      "@type": "Organization",
      name: "TULSI",
      url: siteUrl,
      description:
        "Workflow-first software for industries that run on paper, spreadsheets and WhatsApp.",
      foundingDate: "2026",
      address: {
        "@type": "PostalAddress",
        addressLocality: "Mumbai",
        addressRegion: "Maharashtra",
        addressCountry: "IN",
      },
      founder: {
        "@type": "Person",
        name: "Tulsimani Kumar",
        jobTitle: "Founder",
      },
      sameAs: ["REAL_LINKEDIN_URL"],
    }),
  }}
/>
```

### Academy OS

Use `SoftwareApplication`, not `Organization` — it describes a product:

```tsx
<script
  type="application/ld+json"
  dangerouslySetInnerHTML={{
    __html: JSON.stringify({
      "@context": "https://schema.org",
      "@type": "SoftwareApplication",
      name: "Academy OS",
      applicationCategory: "BusinessApplication",
      operatingSystem: "Web",
      url: siteUrl,
      description:
        "Academy management software for dance, music, art and coaching academies.",
      publisher: { "@type": "Organization", name: "TULSI" },
    }),
  }}
/>
```

**Do NOT add `offers` / pricing to the schema yet.** Billing doesn't exist;
publishing structured prices you can't transact on is worse than omitting them.

**Do NOT add `LocalBusiness` schema or a Google Business Profile link** even
though the audit marks them High Priority. Those checks assume a physical
storefront. This is SaaS. Skipping them is correct.

---

## Task 4 — `robots.txt` and `sitemap.xml`

Neither site has either file. Low value for a single-page site, but cheap.

Create `app/robots.ts` in each repo:

```ts
import type { MetadataRoute } from "next";

const siteUrl = process.env.NEXT_PUBLIC_SITE_URL ?? "https://REPLACE_PER_SITE";

export default function robots(): MetadataRoute.Robots {
  return {
    rules: { userAgent: "*", allow: "/" },
    sitemap: `${siteUrl}/sitemap.xml`,
  };
}
```

Create `app/sitemap.ts`:

```ts
import type { MetadataRoute } from "next";

const siteUrl = process.env.NEXT_PUBLIC_SITE_URL ?? "https://REPLACE_PER_SITE";

export default function sitemap(): MetadataRoute.Sitemap {
  return [
    { url: siteUrl, lastModified: new Date(), priority: 1 },
  ];
}
```

Add `/privacy` to the Academy OS sitemap — that page exists.

---

## Task 5 — Tap target sizes (mobile usability)

**25 tap targets too small** on Academy OS, **19** on TULSI. Academy owners will
open these on a phone, so this is a real usability problem, not a score.

Minimum touch target is **44×44px**. Audit and fix:

- Nav links (both sites)
- Footer links — usually the worst offenders, small text with tight spacing
- The language switcher ("English (Global)")
- Anchor links within sections
- Social icons in the footer

Add padding rather than increasing font size, so the visual design is unchanged:

```css
min-height: 44px;
min-width: 44px;
display: inline-flex;
align-items: center;
```

Verify in Chrome DevTools device mode at 390px width.

---

## Task 6 — Phone number

Both audits report "Address found but phone number is missing."

Add a real WhatsApp number to the contact section and footer of both sites,
as a proper `wa.me` link with the number in it:

```
https://wa.me/91XXXXXXXXXX
```

**Also fix the placeholder links flagged earlier:**

| Site | Broken | Should be |
|---|---|---|
| Both | `https://linkedin.com` | real LinkedIn profile URL |
| Both | `https://wa.me` (no number) | `https://wa.me/91XXXXXXXXXX` |
| Academy OS | 7× `http://localhost:3000` | `https://academy-os-beta.vercel.app` |

The `localhost:3000` links are on every CTA on the Academy OS page — nav, hero,
trial button, all three pricing buttons, and the final CTA. **Fix these first.**
No visitor can currently reach the product.

---

## Task 7 — Analytics

Neither site has analytics. Without it there's no way to know whether anything
here worked. Use Vercel Analytics — it's one dependency and one component:

```bash
npm i @vercel/analytics
```

```tsx
import { Analytics } from "@vercel/analytics/react";
// inside <body>
<Analytics />
```

---

## Deferred — do after the domain is purchased

Do not attempt these now. They require DNS control, which `vercel.app` does not
give you.

1. **Point both sites at the real domain** — e.g. `wearetulsi.com` and
   `academyos.wearetulsi.com` (or a separate product domain).
2. **Set `NEXT_PUBLIC_SITE_URL`** in Vercel env vars for each project. If
   Task 1 was done as specified, this is the only change needed — canonical,
   OG and sitemap URLs all update automatically.
3. **SPF record** — required before emailing academies, or outreach lands in
   spam.
4. **DMARC record** — start at `p=none` and monitor before enforcing.
5. **Re-run the FreeSERP audit** to confirm the structured data grade moved
   from F.

---

## Ignore these audit findings

The audit flags them; they are wrong or not worth acting on:

- **HTTP/2 not enabled** — Vercel's edge serves HTTP/2. Not controllable, and
  likely a detection error in the audit tool.
- **Unminified JS/CSS** — Next.js minifies production builds. False positive.
- **Missing charset** — both reports simultaneously state charset is UTF-8 and
  flag it as missing. Contradictory; already correct.
- **LocalBusiness schema / Google Business Profile / Local SEO** — storefront
  checks, irrelevant to SaaS.
- **Google AMP** — deprecated, do not implement.
- **Facebook Pixel** — no ad spend, no reason.

---

## Verification

After deploying each site:

1. Paste the URL into `https://www.opengraph.xyz` — confirm title, description
   and image render
2. Send the link to yourself on WhatsApp — confirm the preview card appears
3. Google Rich Results Test — confirm the schema parses
4. Chrome DevTools at 390px — tap through every link and button
5. Click every CTA on the Academy OS page — confirm none go to localhost
6. Re-run the FreeSERP audit on both

Expected: On-Page SEO C→A, Structured Data F→A, Technical C→B
(SPF/DMARC keep it off A until the domain move).
