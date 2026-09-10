# Blog Implementation Guide — Sanity CMS + Next.js App Router

> A complete, self-contained spec for rebuilding the FreeSERP blog in **any** Next.js project.
> Feed this file to an AI agent (or follow it by hand) and you will end up with the exact same
> blog: a Sanity-backed, ISR-revalidated, SEO-complete blog with a listing page, client-side
> search, a Portable Text article page, related posts, JSON-LD, sitemap entries and a homepage
> teaser section.
>
> Reference implementation: `ui-freeserp-v2` (Next.js 16 App Router, React 19, TypeScript, Tailwind v4).

---

## Table of contents

1. [Architecture at a glance](#1-architecture-at-a-glance)
2. [Prerequisites](#2-prerequisites)
3. [Part A — Sanity backend (content)](#part-a--sanity-backend-content)
4. [Part B — Next.js frontend (code)](#part-b--nextjs-frontend-code)
5. [Part C — SEO plumbing](#part-c--seo-plumbing)
6. [Part D — Homepage "latest posts" section](#part-d--homepage-latest-posts-section)
7. [Part E — Adapting this to your project](#part-e--adapting-this-to-your-project)
8. [Part F — Verification checklist](#part-f--verification-checklist)
9. [Part G — Troubleshooting](#part-g--troubleshooting)
10. [Appendix — design decisions & why](#appendix--design-decisions--why)

---

## 1. Architecture at a glance

```
┌───────────────────────────┐        ┌──────────────────────────────────────────┐
│  Sanity Studio (hosted)   │  GROQ  │  Next.js App Router (marketing site)     │
│  yourproject.sanity.studio│◀──────▶│                                          │
│                           │ https  │  lib/sanity.ts      ← client + queries   │
│  Schemas:                 │        │  app/blog/page.tsx  ← listing (ISR 60s)  │
│   • post                  │        │  app/blog/[slug]    ← article (ISR 30s)  │
│   • author                │        │  components/blog/*  ← UI                 │
│   • category              │        │  app/sitemap.ts     ← post URLs (ISR)    │
│   • blockContent          │        │                                          │
└───────────────────────────┘        └──────────────────────────────────────────┘
             │                                          │
             │  images                                  │  next/image optimizes
             └──────────────▶ cdn.sanity.io ◀───────────┘  remote Sanity CDN URLs
```

**Key properties of this design**

| Property | How it's achieved |
|---|---|
| No rebuild needed to publish | ISR: `export const revalidate = 60` on the list, `30` on posts, `60` on the sitemap |
| Fast first paint on known posts | `generateStaticParams()` pre-renders every existing slug at build time |
| Never crashes if the CMS is down | Every fetch is wrapped in `try/catch` with an empty-array/empty-state fallback |
| No secrets in the frontend | `projectId`/`dataset` are public identifiers; the dataset is `public`, so **no API token is required** |
| Rich content | Portable Text with custom renderers for images, YouTube, Vimeo, code, quotes, lists, links |
| SEO complete | Per-post `generateMetadata`, canonical URLs, OpenGraph + Twitter cards, `BlogPosting` JSON-LD, sitemap, robots |
| Lowercase-slug hygiene | `permanentRedirect` from any mixed-case slug to its lowercase form |

---

## 2. Prerequisites

- Node.js 20+
- A Next.js **App Router** project (`app/` directory), TypeScript
- The `@/*` path alias mapped to the project root in `tsconfig.json`:

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "paths": { "@/*": ["./*"] }
  }
}
```

- A free Sanity.io account (https://www.sanity.io)

---

# Part A — Sanity backend (content)

The reference project keeps the Studio **outside** the Next.js app (it is deployed to
`*.sanity.studio`). That is the recommended setup: the marketing site stays a pure consumer and
ships no Studio JS. If you prefer an embedded Studio at `/studio`, see the note at the end of A3.

## A1. Create the Sanity project

```bash
# anywhere OUTSIDE your Next.js repo, e.g. ../my-studio
npm create sanity@latest -- --template clean --create-project "My Blog" --dataset production
cd my-studio
```

When prompted:
- **TypeScript?** → Yes
- **Project output path** → `my-studio`
- **Package manager** → npm

Note the **projectId** it prints (looks like `7lsuu424`) — you'll need it in Part B.

Make the dataset public so the frontend can read without a token:

```bash
npx sanity dataset visibility set production public
```

## A2. Schemas

Create these four files under `schemaTypes/`.

### `schemaTypes/blockContent.ts`

This is the rich-text type used by `post.body` and `author.bio`. Every block/mark/embed listed
here has a matching renderer in `components/blog/PortableBody.tsx` — **keep the two in sync**.

```ts
import { defineType, defineArrayMember } from 'sanity'

export default defineType({
  title: 'Block Content',
  name: 'blockContent',
  type: 'array',
  of: [
    defineArrayMember({
      title: 'Block',
      type: 'block',
      // H1 is intentionally absent: the page <h1> is the post title.
      styles: [
        { title: 'Normal', value: 'normal' },
        { title: 'H2', value: 'h2' },
        { title: 'H3', value: 'h3' },
        { title: 'H4', value: 'h4' },
        { title: 'Quote', value: 'blockquote' },
      ],
      lists: [
        { title: 'Bullet', value: 'bullet' },
        { title: 'Numbered', value: 'number' },
      ],
      marks: {
        decorators: [
          { title: 'Strong', value: 'strong' },
          { title: 'Emphasis', value: 'em' },
          { title: 'Code', value: 'code' },
        ],
        annotations: [
          {
            title: 'URL',
            name: 'link',
            type: 'object',
            fields: [
              {
                title: 'URL',
                name: 'href',
                type: 'url',
                validation: (R: any) => R.uri({ scheme: ['http', 'https', 'mailto', 'tel'] }),
              },
            ],
          },
        ],
      },
    }),

    // Inline image with alt + caption
    defineArrayMember({
      type: 'image',
      options: { hotspot: true },
      fields: [
        {
          name: 'alt',
          type: 'string',
          title: 'Alt text',
          description: 'Important for SEO and accessibility.',
        },
        { name: 'caption', type: 'string', title: 'Caption' },
      ],
    }),

    // YouTube embed
    defineArrayMember({
      name: 'youtube',
      type: 'object',
      title: 'YouTube embed',
      fields: [
        { name: 'url', type: 'url', title: 'YouTube URL' },
        { name: 'caption', type: 'string', title: 'Caption' },
      ],
      preview: { select: { title: 'url' }, prepare: ({ title }: any) => ({ title: `▶ ${title}` }) },
    }),

    // Vimeo embed
    defineArrayMember({
      name: 'vimeo',
      type: 'object',
      title: 'Vimeo embed',
      fields: [
        { name: 'url', type: 'url', title: 'Vimeo URL' },
        { name: 'caption', type: 'string', title: 'Caption' },
      ],
      preview: { select: { title: 'url' }, prepare: ({ title }: any) => ({ title: `▶ ${title}` }) },
    }),

    // Code block
    defineArrayMember({
      name: 'code',
      type: 'object',
      title: 'Code block',
      fields: [
        { name: 'code', type: 'text', title: 'Code' },
        { name: 'language', type: 'string', title: 'Language' },
      ],
    }),
  ],
})
```

### `schemaTypes/post.ts`

```ts
import { defineType, defineField } from 'sanity'

export default defineType({
  name: 'post',
  title: 'Post',
  type: 'document',
  fields: [
    defineField({ name: 'title', title: 'Title', type: 'string', validation: (R) => R.required() }),

    defineField({
      name: 'slug',
      title: 'Slug',
      type: 'slug',
      // Slugs MUST be lowercase — the article route permanently redirects
      // mixed-case URLs to lowercase, so a mixed-case slug would 404 forever.
      options: {
        source: 'title',
        maxLength: 96,
        slugify: (input) =>
          input.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-]/g, '').slice(0, 96),
      },
      validation: (R) => R.required(),
    }),

    defineField({
      name: 'excerpt',
      title: 'Excerpt',
      type: 'text',
      rows: 3,
      description:
        'Shown on cards, as the article intro paragraph, and as the meta description fallback.',
      validation: (R) => R.max(300),
    }),

    defineField({
      name: 'mainImage',
      title: 'Main image',
      type: 'image',
      options: { hotspot: true },
      fields: [{ name: 'alt', type: 'string', title: 'Alt text' }],
    }),

    defineField({
      name: 'publishedAt',
      title: 'Published at',
      type: 'datetime',
      initialValue: () => new Date().toISOString(),
      validation: (R) => R.required(),
    }),

    defineField({ name: 'author', title: 'Author', type: 'reference', to: [{ type: 'author' }] }),

    defineField({
      name: 'categories',
      title: 'Categories',
      type: 'array',
      of: [{ type: 'reference', to: [{ type: 'category' }] }],
    }),

    defineField({ name: 'body', title: 'Body', type: 'blockContent' }),

    // ─── SEO overrides ───────────────────────────────────────────────
    defineField({
      name: 'metaTitle',
      title: 'Meta title (SEO)',
      type: 'string',
      description: 'Overrides the post title in <title> and OG tags. Aim for ≤ 60 chars.',
    }),
    defineField({
      name: 'metaDescription',
      title: 'Meta description (SEO)',
      type: 'text',
      rows: 2,
      description: 'Overrides the excerpt in the meta description. Aim for ≤ 155 chars.',
    }),
  ],

  orderings: [
    {
      title: 'Newest first',
      name: 'publishedAtDesc',
      by: [{ field: 'publishedAt', direction: 'desc' }],
    },
  ],

  preview: {
    select: { title: 'title', author: 'author.name', media: 'mainImage', date: 'publishedAt' },
    prepare: ({ title, author, media, date }) => ({
      title,
      subtitle: [author, date && new Date(date).toLocaleDateString()].filter(Boolean).join(' · '),
      media,
    }),
  },
})
```

### `schemaTypes/author.ts`

```ts
import { defineType, defineField } from 'sanity'

export default defineType({
  name: 'author',
  title: 'Author',
  type: 'document',
  fields: [
    defineField({ name: 'name', title: 'Name', type: 'string', validation: (R) => R.required() }),
    defineField({ name: 'slug', title: 'Slug', type: 'slug', options: { source: 'name', maxLength: 96 } }),
    defineField({ name: 'image', title: 'Photo', type: 'image', options: { hotspot: true } }),
    defineField({ name: 'bio', title: 'Bio', type: 'blockContent' }),
    defineField({
      name: 'url',
      title: 'Profile URL',
      type: 'url',
      description: 'LinkedIn/X/personal site — emitted as schema.org sameAs.',
    }),
  ],
  preview: { select: { title: 'name', media: 'image' } },
})
```

> ⚠️ The article page derives the author URL as
> `name.toLowerCase().replace(/\s+/g, "-")` — **not** from `author.slug`. Keep the author's
> name slug-safe, or switch the code to read `author.slug.current` (see B4/C4 notes).

### `schemaTypes/category.ts`

```ts
import { defineType, defineField } from 'sanity'

export default defineType({
  name: 'category',
  title: 'Category',
  type: 'document',
  fields: [
    defineField({ name: 'title', title: 'Title', type: 'string', validation: (R) => R.required() }),
    defineField({
      name: 'slug',
      title: 'Slug',
      type: 'slug',
      options: { source: 'title', maxLength: 96 },
      validation: (R) => R.required(),
    }),
    defineField({ name: 'description', title: 'Description', type: 'text' }),
  ],
})
```

### `schemaTypes/index.ts`

```ts
import blockContent from './blockContent'
import post from './post'
import author from './author'
import category from './category'

export const schemaTypes = [post, author, category, blockContent]
```

## A3. CORS + deploy the Studio

Allow your frontend origins to read the API (no credentials needed for a public dataset):

```bash
npx sanity cors add http://localhost:3000
npx sanity cors add https://yourdomain.com
```

Deploy the Studio to a hosted URL:

```bash
npx sanity deploy      # choose a hostname → https://<hostname>.sanity.studio
```

<details>
<summary>Alternative: embed the Studio inside Next.js at <code>/studio</code></summary>

Install `sanity` + `next-sanity` in the Next app and add
`app/studio/[[...tool]]/page.tsx` re-exporting `NextStudio`. The reference project deliberately
does **not** do this — it keeps ~2 MB of Studio JS out of the marketing bundle and lets editors
work even while the site is being redeployed.
</details>

## A4. Publish a first post

In the Studio: create one **Author**, one or two **Categories**, then a **Post** with a title,
slug, excerpt, main image, published date, author, categories and body. Hit **Publish** — the
frontend only reads published documents (`perspective: "published"`).

---

# Part B — Next.js frontend (code)

## B1. Install dependencies

```bash
npm install @sanity/client @sanity/image-url @portabletext/react
```

Reference versions (Next 16 / React 19):

```jsonc
"@portabletext/react": "^4.0.3",
"@sanity/client": "^7.22.0",
"@sanity/image-url": "^1.2.0",
"next": "16.2.6",
"react": "19.2.4",
"react-dom": "19.2.4"
```

## B2. Environment variables

`.env` (or `.env.local`) — these are **public identifiers, not secrets**:

```bash
NEXT_PUBLIC_SANITY_PROJECT_ID=your_project_id
NEXT_PUBLIC_SANITY_DATASET=production
```

## B3. `next.config.ts` — allow the Sanity image CDN

`next/image` refuses remote hosts that aren't allowlisted. Without this, every post image 400s.

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "cdn.sanity.io", pathname: "/**" },
      // add any other external image hosts your site uses
    ],
  },
};

export default nextConfig;
```

## B4. `lib/sanity.ts` — client, image builder, types, queries

This is the single data-access module. Everything blog-related fetches through it.

```ts
/* eslint-disable @typescript-eslint/no-explicit-any */
import { createClient } from "@sanity/client";
import imageUrlBuilder from "@sanity/image-url";

/**
 * Sanity client. projectId/dataset are public identifiers (not secrets);
 * they default here and can be overridden via .env.local.
 */
export const client = createClient({
  projectId: process.env.NEXT_PUBLIC_SANITY_PROJECT_ID || "your_project_id",
  dataset: process.env.NEXT_PUBLIC_SANITY_DATASET || "production",
  apiVersion: "2024-01-01",
  useCdn: false,            // we rely on Next's ISR cache, not Sanity's CDN
  perspective: "published", // never leak drafts
});

const builder = imageUrlBuilder(client);

/** Build an image URL from a Sanity image reference. */
export function urlFor(source: any) {
  return builder.image(source);
}

/** A post as shown in listings/cards. */
export type BlogPost = {
  _id: string;
  title: string;
  slug: { current: string };
  excerpt?: string;
  mainImage?: any;
  publishedAt: string;
  author?: string;        // flattened to a name string for cards
  categories?: string[];  // flattened to title strings for cards
};

/** A fully-resolved post for the detail page. */
export type BlogPostFull = {
  _id: string;
  title: string;
  slug: { current: string };
  excerpt?: string;
  mainImage?: any;
  publishedAt: string;
  _updatedAt?: string;
  body?: any;
  metaTitle?: string;
  metaDescription?: string;
  author?: { name: string; image?: any; bio?: any; url?: string };
  categories?: { title: string; slug: { current: string } }[];
};

/** All posts, newest first. */
export async function getAllPosts(): Promise<BlogPost[]> {
  return client.fetch(
    `*[_type == "post"] | order(publishedAt desc) {
      _id, title, slug, excerpt, mainImage, publishedAt,
      "author": author->name,
      "categories": categories[]->title
    }`,
    {},
    { next: { revalidate: 60 } },
  );
}

/** A homepage teaser post — same as a listing post, plus an estimated read time. */
export type LatestPost = BlogPost & { readMinutes?: number };

/** The newest posts, for the homepage blog section. */
export async function getLatestPosts(limit = 3): Promise<LatestPost[]> {
  return client.fetch(
    `*[_type == "post" && defined(slug.current)] | order(publishedAt desc)[0...$limit] {
      _id, title, slug, excerpt, mainImage, publishedAt,
      "author": author->name,
      "categories": categories[]->title,
      "readMinutes": round(length(pt::text(body)) / 1100)
    }`,
    { limit },
    { next: { revalidate: 60 } },
  );
}

/** A single post by slug, or null if it doesn't exist. */
export async function getPost(slug: string): Promise<BlogPostFull | null> {
  return client.fetch(
    `*[_type == "post" && slug.current == $slug][0] {
      _id, _updatedAt, title, slug, excerpt, mainImage, publishedAt, body, metaTitle, metaDescription,
      "author": author->{name, image, bio, url},
      "categories": categories[]->{title, slug}
    }`,
    { slug },
    { next: { revalidate: 30 } },
  );
}

/** Recent posts excluding the current one — for the "more from the blog" rail. */
export async function getRelatedPosts(currentSlug: string, limit = 3): Promise<BlogPost[]> {
  return client.fetch(
    `*[_type == "post" && slug.current != $currentSlug] | order(publishedAt desc)[0...$limit] {
      _id, title, slug, excerpt, mainImage, publishedAt,
      "author": author->name,
      "categories": categories[]->title
    }`,
    { currentSlug, limit },
    { next: { revalidate: 60 } },
  );
}
```

**GROQ notes**

- `author->name` dereferences the author document and projects a single field; the `"author":`
  prefix renames it so the card can render `post.author` as a plain string.
- `categories[]->title` dereferences every item in the reference array.
- `round(length(pt::text(body)) / 1100)` computes read-minutes **server-side**:
  `pt::text()` flattens Portable Text to a plain string, and ~1100 characters ≈ 1 minute at
  ~200 wpm. This avoids shipping the whole body to the homepage just to count words.
- `{ next: { revalidate: N } }` is Next.js fetch-level ISR — `@sanity/client` forwards it to
  `fetch`, so each query gets its own cache lifetime.
- `useCdn: false` + `perspective: "published"` = always fresh within the revalidate window, never drafts.

> **Related-posts upgrade (optional):** the reference implementation shows *recent* posts, not
> topically related ones. To make it category-aware:
> ```groq
> *[_type == "post" && slug.current != $currentSlug &&
>   count(categories[@._ref in $catIds]) > 0]
>   | order(publishedAt desc)[0...$limit] { ... }
> ```

## B5. Shared design tokens

The blog components reference a `COLORS` object. If your project doesn't have one, create
`components/site/constants.ts`:

```ts
export const COLORS = {
  blue: "#0454ff",
  black: "#000",
  white: "#fff",
  gray: "#6d6d6d",
  subtle: "#8a8a93",
  border: "#eaeaea",
  bg: "#fff",
  softGray: "#f5f6f8",
  red: "#ff5b5b",
  redBg: "#fff2f2",
  blueBg: "#ebf1ff",
  purple: "#7e5bff",
  green: "#0ea66f",
} as const;
```

Swap the hex values for your brand — nothing else needs to change.

## B6. `components/blog/PostCard.tsx`

The single card used by the listing grid **and** the related-posts rail. Server component.

```tsx
import Link from "next/link";
import Image from "next/image";
import { COLORS } from "@/components/site/constants";
import { urlFor, type BlogPost } from "@/lib/sanity";

function formatDate(iso?: string) {
  if (!iso) return "Recently";
  const d = new Date(iso);
  if (d.getFullYear() < 2000) return "Recently";
  return d.toLocaleDateString("en-US", { day: "numeric", month: "short", year: "numeric" });
}

export function PostCard({ post }: { post: BlogPost }) {
  return (
    <Link
      href={`/blog/${post.slug.current}`}
      className="fs-card fs-blog-card"
      style={{
        display: "flex",
        flexDirection: "column",
        height: "100%",
        border: `1px solid ${COLORS.border}`,
        borderRadius: 14,
        overflow: "hidden",
        background: "#fff",
        textDecoration: "none",
        color: "inherit",
      }}
    >
      {post.mainImage?.asset && (
        <div
          style={{
            position: "relative",
            width: "100%",
            aspectRatio: "16 / 9",
            background: COLORS.softGray,
            overflow: "hidden",
          }}
        >
          <Image
            src={urlFor(post.mainImage).width(800).height(450).url()}
            alt={post.title}
            fill
            sizes="(max-width: 1000px) 100vw, 380px"
            style={{ objectFit: "cover" }}
          />
        </div>
      )}

      <div style={{ display: "flex", flexDirection: "column", gap: 10, padding: 22, flex: 1 }}>
        {post.categories && post.categories.length > 0 && (
          <div style={{ display: "flex", flexWrap: "wrap", gap: 6 }}>
            {post.categories.slice(0, 2).map((cat) => (
              <span
                key={cat}
                style={{
                  padding: "4px 10px",
                  borderRadius: 100,
                  background: COLORS.blueBg,
                  color: COLORS.blue,
                  fontSize: 10,
                  fontWeight: 600,
                  letterSpacing: "0.1em",
                  textTransform: "uppercase",
                  fontFamily: "var(--font-geist-mono)",
                }}
              >
                {cat}
              </span>
            ))}
          </div>
        )}

        <h3
          className="fs-clamp-2"
          style={{
            fontSize: 19,
            fontWeight: 600,
            letterSpacing: "-0.02em",
            lineHeight: 1.3,
            margin: 0,
            color: "#0f1018",
          }}
        >
          {post.title}
        </h3>

        {post.excerpt && (
          <p
            className="fs-clamp-3"
            style={{ fontSize: 14.5, lineHeight: 1.6, color: COLORS.gray, margin: 0 }}
          >
            {post.excerpt}
          </p>
        )}

        <div
          style={{
            marginTop: "auto",
            paddingTop: 10,
            display: "flex",
            justifyContent: "space-between",
            gap: 10,
            fontSize: 12,
            fontFamily: "var(--font-geist-mono)",
            color: COLORS.subtle,
          }}
        >
          <span>{post.author || "YourBrand"}</span>
          <span>{formatDate(post.publishedAt)}</span>
        </div>
      </div>
    </Link>
  );
}
```

Details that matter:
- `post.mainImage?.asset` guard — a post without an image renders a text-only card, not a broken box.
- `urlFor(...).width(800).height(450)` asks the Sanity CDN for an exactly-sized, hotspot-cropped
  image; `next/image` then serves AVIF/WebP on top.
- `marginTop: "auto"` on the meta row pins author/date to the bottom so cards of different text
  lengths line up.
- `formatDate` returns `"Recently"` for missing/garbage dates (year < 2000) instead of `Invalid Date`.

## B7. `components/blog/PostsGrid.tsx` — search + empty states

The **only** client component in the blog. It holds the search box state and filters in memory —
no extra network round-trips, which is the right call for a blog under a few hundred posts.

```tsx
"use client";

import { useMemo, useState } from "react";
import { COLORS } from "@/components/site/constants";
import type { BlogPost } from "@/lib/sanity";
import { PostCard } from "./PostCard";

function EmptyState({
  title,
  sub,
  action,
}: {
  title: string;
  sub: string;
  action?: React.ReactNode;
}) {
  return (
    <div
      style={{
        textAlign: "center",
        padding: "80px 24px",
        border: `1px dashed ${COLORS.border}`,
        borderRadius: 16,
      }}
    >
      <div style={{ fontSize: 20, fontWeight: 600, letterSpacing: "-0.02em", color: "#0f1018" }}>
        {title}
      </div>
      <p style={{ fontSize: 15, color: COLORS.gray, margin: "8px 0 0" }}>{sub}</p>
      {action && <div style={{ marginTop: 16 }}>{action}</div>}
    </div>
  );
}

export function PostsGrid({ posts }: { posts: BlogPost[] }) {
  const [query, setQuery] = useState("");

  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return posts;
    return posts.filter(
      (p) =>
        p.title.toLowerCase().includes(q) ||
        p.excerpt?.toLowerCase().includes(q) ||
        p.author?.toLowerCase().includes(q) ||
        p.categories?.some((c) => c.toLowerCase().includes(q)),
    );
  }, [posts, query]);

  return (
    <>
      <div style={{ maxWidth: 460, margin: "0 auto 48px" }}>
        <input
          type="search"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Search articles, categories, authors…"
          aria-label="Search blog posts"
          style={{
            width: "100%",
            padding: "13px 16px",
            fontSize: 15,
            borderRadius: 10,
            border: `1px solid ${COLORS.border}`,
            outline: "none",
            background: "#fff",
            color: "#0f1018",
          }}
        />
      </div>

      {posts.length === 0 ? (
        <EmptyState
          title="No posts yet"
          sub="Publish a post in Sanity Studio and it will show up here."
        />
      ) : filtered.length === 0 ? (
        <EmptyState
          title="No results"
          sub={`Nothing matches “${query}”.`}
          action={
            <button
              type="button"
              onClick={() => setQuery("")}
              className="fs-btn"
              style={{
                border: `1px solid ${COLORS.border}`,
                background: "#fff",
                color: COLORS.blue,
                fontWeight: 600,
                fontSize: 14,
                padding: "9px 18px",
                borderRadius: 100,
                cursor: "pointer",
              }}
            >
              Clear search
            </button>
          }
        />
      ) : (
        <div className="fs-grid-3">
          {filtered.map((post) => (
            <PostCard key={post._id} post={post} />
          ))}
        </div>
      )}
    </>
  );
}
```

Two distinct empty states — "no posts at all" vs "no search matches" — is the detail that makes
this feel finished. The second one offers a **Clear search** escape hatch.

## B8. `components/blog/PortableBody.tsx` — the Portable Text renderer

Maps every Sanity block/mark/embed type to styled JSX. **This must mirror `blockContent.ts`.**

```tsx
/* eslint-disable @typescript-eslint/no-explicit-any */
import Image from "next/image";
import { PortableText, type PortableTextComponents } from "@portabletext/react";
import { COLORS } from "@/components/site/constants";
import { urlFor } from "@/lib/sanity";

function youTubeId(url: string) {
  const m = url.match(/^.*(youtu.be\/|v\/|u\/\w\/|embed\/|watch\?v=|&v=)([^#&?]*).*/);
  return m && m[2].length === 11 ? m[2] : null;
}
function vimeoId(url: string) {
  const m = url.match(/vimeo\.com\/(\d+)/);
  return m ? m[1] : null;
}

const captionStyle: React.CSSProperties = {
  fontFamily: "var(--font-geist-mono)",
  fontSize: 12,
  color: COLORS.subtle,
  textAlign: "center",
  margin: "10px 0 0",
};
const embedFrame: React.CSSProperties = {
  position: "absolute",
  inset: 0,
  width: "100%",
  height: "100%",
  border: `1px solid ${COLORS.border}`,
  borderRadius: 12,
};

const components: PortableTextComponents = {
  types: {
    image: ({ value }: any) => {
      if (!value?.asset?._ref) return null;
      return (
        <figure style={{ margin: "32px 0" }}>
          <Image
            src={urlFor(value).width(1600).url()}
            alt={value.alt || ""}
            width={1600}
            height={900}
            sizes="(max-width: 820px) 100vw, 740px"
            style={{ width: "100%", height: "auto", borderRadius: 12, display: "block" }}
          />
          {(value.caption || value.alt) && (
            <figcaption style={captionStyle}>{value.caption || value.alt}</figcaption>
          )}
        </figure>
      );
    },
    youtube: ({ value }: any) => {
      const id = youTubeId(value?.url || "");
      if (!id) return null;
      return (
        <figure style={{ margin: "32px 0" }}>
          <div style={{ position: "relative", width: "100%", paddingBottom: "56.25%" }}>
            <iframe
              style={embedFrame}
              src={`https://www.youtube.com/embed/${id}`}
              title="YouTube video player"
              allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
              allowFullScreen
            />
          </div>
          {value.caption && <figcaption style={captionStyle}>{value.caption}</figcaption>}
        </figure>
      );
    },
    vimeo: ({ value }: any) => {
      const id = vimeoId(value?.url || "");
      if (!id) return null;
      return (
        <figure style={{ margin: "32px 0" }}>
          <div style={{ position: "relative", width: "100%", paddingBottom: "56.25%" }}>
            <iframe
              style={embedFrame}
              src={`https://player.vimeo.com/video/${id}`}
              title="Vimeo video player"
              allow="autoplay; fullscreen; picture-in-picture"
              allowFullScreen
            />
          </div>
          {value.caption && <figcaption style={captionStyle}>{value.caption}</figcaption>}
        </figure>
      );
    },
    code: ({ value }: any) => (
      <pre
        style={{
          background: COLORS.softGray,
          border: `1px solid ${COLORS.border}`,
          borderRadius: 12,
          padding: 18,
          overflowX: "auto",
          margin: "24px 0",
        }}
      >
        <code style={{ fontFamily: "var(--font-geist-mono)", fontSize: 13, color: "#0f1018" }}>
          {value?.code}
        </code>
      </pre>
    ),
  },

  block: {
    h2: ({ children }: any) => (
      <h2
        style={{
          fontSize: "clamp(24px, 3vw, 32px)",
          fontWeight: 600,
          letterSpacing: "-0.03em",
          lineHeight: 1.2,
          margin: "44px 0 16px",
          color: "#0f1018",
        }}
      >
        {children}
      </h2>
    ),
    h3: ({ children }: any) => (
      <h3
        style={{
          fontSize: "clamp(20px, 2.4vw, 24px)",
          fontWeight: 600,
          letterSpacing: "-0.02em",
          lineHeight: 1.25,
          margin: "32px 0 12px",
          color: "#0f1018",
        }}
      >
        {children}
      </h3>
    ),
    h4: ({ children }: any) => (
      <h4 style={{ fontSize: 19, fontWeight: 600, margin: "26px 0 10px", color: "#0f1018" }}>
        {children}
      </h4>
    ),
    blockquote: ({ children }: any) => (
      <blockquote
        style={{
          borderLeft: `3px solid ${COLORS.blue}`,
          background: COLORS.blueBg,
          margin: "24px 0",
          padding: "14px 20px",
          borderRadius: "0 10px 10px 0",
          color: "#0f1018",
          fontSize: 17,
          lineHeight: 1.6,
        }}
      >
        {children}
      </blockquote>
    ),
    normal: ({ children }: any) => (
      <p style={{ fontSize: 17, lineHeight: 1.7, color: COLORS.gray, margin: "0 0 18px" }}>
        {children}
      </p>
    ),
  },

  list: {
    bullet: ({ children }: any) => (
      <ul style={{ margin: "18px 0", paddingLeft: 22, display: "flex", flexDirection: "column", gap: 8 }}>
        {children}
      </ul>
    ),
    number: ({ children }: any) => (
      <ol style={{ margin: "18px 0", paddingLeft: 22, display: "flex", flexDirection: "column", gap: 8 }}>
        {children}
      </ol>
    ),
  },

  listItem: {
    bullet: ({ children }: any) => (
      <li style={{ fontSize: 17, lineHeight: 1.6, color: COLORS.gray }}>{children}</li>
    ),
    number: ({ children }: any) => (
      <li style={{ fontSize: 17, lineHeight: 1.6, color: COLORS.gray }}>{children}</li>
    ),
  },

  marks: {
    strong: ({ children }: any) => (
      <strong style={{ fontWeight: 600, color: "#0f1018" }}>{children}</strong>
    ),
    em: ({ children }: any) => <em>{children}</em>,
    code: ({ children }: any) => (
      <code
        style={{
          fontFamily: "var(--font-geist-mono)",
          fontSize: "0.88em",
          background: COLORS.softGray,
          border: `1px solid ${COLORS.border}`,
          borderRadius: 5,
          padding: "1px 6px",
          color: COLORS.blue,
        }}
      >
        {children}
      </code>
    ),
    link: ({ value, children }: any) => {
      const external = (value?.href || "").startsWith("http");
      return (
        <a
          href={value?.href}
          target={external ? "_blank" : undefined}
          rel={external ? "noopener noreferrer" : undefined}
          style={{ color: COLORS.blue, textDecoration: "underline", textUnderlineOffset: 2 }}
        >
          {children}
        </a>
      );
    },
  },
};

export function PortableBody({ value }: { value: any }) {
  if (!value) return null;
  return <PortableText value={value} components={components} />;
}
```

Notable:
- **No `h1`** — the page's `<h1>` is the post title, so body headings start at `h2`. Correct
  document outline for both a11y and SEO.
- External links get `target="_blank"` + `rel="noopener noreferrer"`; internal links don't.
- Video embeds use the `padding-bottom: 56.25%` aspect-ratio box, so no CLS.
- Every renderer returns `null` on malformed input (missing asset ref, unparseable video URL)
  rather than throwing — a bad paste in the CMS can't take the page down.
- `PortableBody` is reused for the **author bio** on the article page.

## B9. `components/blog/RelatedPosts.tsx`

```tsx
import { COLORS } from "@/components/site/constants";
import type { BlogPost } from "@/lib/sanity";
import { PostCard } from "./PostCard";

export function RelatedPosts({ posts }: { posts: BlogPost[] }) {
  if (!posts || posts.length === 0) return null;

  return (
    <section style={{ marginTop: 72, paddingTop: 48, borderTop: `1px solid ${COLORS.border}` }}>
      <div
        style={{
          fontSize: 12,
          fontFamily: "var(--font-geist-mono)",
          letterSpacing: "0.16em",
          textTransform: "uppercase",
          color: COLORS.blue,
        }}
      >
        Keep reading
      </div>
      <h2
        style={{
          fontSize: "clamp(24px, 3vw, 34px)",
          fontWeight: 600,
          letterSpacing: "-0.03em",
          margin: "8px 0 28px",
          color: "#0f1018",
        }}
      >
        More from the blog
      </h2>

      <div className="fs-grid-3">
        {posts.map((post) => (
          <PostCard key={post._id} post={post} />
        ))}
      </div>
    </section>
  );
}
```

## B10. CSS

The components style themselves with **inline styles**, which cannot express media queries or
`:hover`. So a small CSS file carries the responsive overrides (with `!important`, because inline
styles win otherwise) and the hover/clamp utilities.

### `components/blog/blog.css`

```css
/* ─── Mobile responsiveness ──────────────────────────────────────────
   Inline styles can't hold media queries; we use !important here to
   override them at the mobile breakpoints.
   ------------------------------------------------------------------ */

/* ── Blog index page: hero + main padding ── */
@media (max-width: 640px) {
  .fs-blog-hero {
    padding-top: 100px !important;
    padding-bottom: 56px !important;
  }
  .fs-blog-hero-inner {
    padding-left: 20px !important;
    padding-right: 20px !important;
  }
  .fs-blog-main {
    padding: 40px 20px 72px !important;
  }
}

/* ── Blog post page: hero + main padding ── */
@media (max-width: 640px) {
  .fs-blog-post-hero {
    padding-top: 100px !important;
    padding-bottom: 48px !important;
  }
  .fs-blog-post-hero-inner {
    padding-left: 20px !important;
    padding-right: 20px !important;
  }
  .fs-blog-post-main {
    padding-left: 20px !important;
    padding-right: 20px !important;
    padding-bottom: 72px !important;
  }
}

/* ── Blog post body: code blocks must scroll, not overflow ── */
@media (max-width: 640px) {
  .fs-blog-post-main pre {
    overflow-x: auto !important;
    max-width: 100% !important;
  }
  /* Ensure embedded iframes (YouTube/Vimeo) stay in viewport */
  .fs-blog-post-main iframe {
    max-width: 100% !important;
  }
}

/* ─── blog utilities ─────────────────────────────────────────────── */
.fs-clamp-2 {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
.fs-clamp-3 {
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
/* gentle image zoom on post-card hover (pairs with .fs-card lift) */
.fs-blog-card img {
  transition: transform 0.45s cubic-bezier(0.2, 0.7, 0.2, 1);
}
.fs-blog-card:hover img {
  transform: scale(1.045);
}
```

### Shared classes the blog depends on

If your project doesn't already define these, add them (they live in the site-wide stylesheet in
the reference project):

```css
/* card hover lift */
.fs-card {
  transition: transform 0.25s ease, box-shadow 0.25s ease;
}
.fs-card:hover {
  transform: translateY(-6px);
  box-shadow: 0 18px 40px rgba(0, 0, 0, 0.08);
}

/* button hover */
.fs-btn {
  transition: transform 0.2s ease, filter 0.2s ease;
}
.fs-btn:hover {
  transform: translateY(-2px);
  filter: brightness(1.08);
}

/* 3-up grid → 2-up on tablet → 1-up on mobile */
.fs-grid-3 {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 24px;
}
@media (max-width: 1000px) {
  .fs-grid-3 { grid-template-columns: repeat(2, 1fr); }
}
@media (max-width: 640px) {
  .fs-grid-3 { grid-template-columns: 1fr; }
}
```

### Wire it into `app/globals.css`

```css
@import "tailwindcss";
@import "../components/blog/blog.css";
/* ...your other partials */
```

Also make sure a mono font variable exists (the blog uses `var(--font-geist-mono)` for meta text
and chips). In `app/layout.tsx`:

```tsx
import { Geist_Mono } from "next/font/google";
const geistMono = Geist_Mono({ variable: "--font-geist-mono", subsets: ["latin"] });
// <html className={geistMono.variable}>
```

## B11. `app/blog/page.tsx` — the listing page

```tsx
import type { Metadata } from "next";
import type { CSSProperties } from "react";
import { Nav } from "@/components/site/Nav";
import { Footer } from "@/components/site/Footer";
import { PostsGrid } from "@/components/blog/PostsGrid";
import { getAllPosts, type BlogPost } from "@/lib/sanity";

export const metadata: Metadata = {
  title: "Blog — Insights & Guides | YourBrand",
  description: "Expert tips, strategies, and insights from the YourBrand team.",
  alternates: { canonical: "https://yourdomain.com/blog" },
  openGraph: {
    type: "website",
    title: "Blog — Insights & Guides | YourBrand",
    description: "Expert tips, strategies, and insights from the YourBrand team.",
    url: "https://yourdomain.com/blog",
    siteName: "YourBrand",
  },
};

// New posts appear within a minute without a rebuild.
export const revalidate = 60;

const heroSection: CSSProperties = {
  position: "relative",
  paddingTop: 160,
  paddingBottom: 90,
  textAlign: "center",
  backgroundImage:
    "linear-gradient(rgba(8,10,22,.74), rgba(8,10,22,.84)), url(/blog-hero.jpg)",
  backgroundSize: "cover",
  backgroundPosition: "center",
};

export default async function BlogPage() {
  let posts: BlogPost[] = [];
  try {
    posts = await getAllPosts();
  } catch {
    // CMS unreachable — render the empty state instead of crashing.
  }

  return (
    <>
      <Nav currentNav="Blog" />

      <header className="fs-blog-hero" style={heroSection}>
        <div
          className="fs-blog-hero-inner"
          style={{ maxWidth: 1200, margin: "0 auto", padding: "0 40px" }}
        >
          <div
            style={{
              fontFamily: "var(--font-geist-mono)",
              fontSize: 12,
              letterSpacing: "0.2em",
              textTransform: "uppercase",
              color: "#9db4ff",
            }}
          >
            Insights &amp; Guides
          </div>
          <h1
            style={{
              color: "#fff",
              fontSize: "clamp(40px, 6vw, 72px)",
              fontWeight: 600,
              letterSpacing: "-0.04em",
              lineHeight: 1.05,
              margin: "16px 0 0",
            }}
          >
            The YourBrand Blog
          </h1>
          <p
            style={{
              color: "rgba(255,255,255,.72)",
              fontSize: 17,
              lineHeight: 1.6,
              maxWidth: 560,
              margin: "16px auto 0",
            }}
          >
            Expert tips, strategies, and insights from our team.
          </p>
        </div>
      </header>

      <main
        className="fs-blog-main"
        style={{ maxWidth: 1200, margin: "0 auto", padding: "72px 40px 110px" }}
      >
        <PostsGrid posts={posts} />
      </main>

      <Footer />
    </>
  );
}
```

> `Nav` and `Footer` are your project's own components. If you don't have them, drop those two
> lines — everything else works standalone.

## B12. `app/blog/[slug]/page.tsx` — the article page

The biggest file. It does: static params, per-post metadata, lowercase-slug redirect, JSON-LD,
hero, overlapping cover image, intro + key-takeaway, Portable Text body, author box, related rail.

```tsx
import type { Metadata } from "next";
import type { CSSProperties } from "react";
import Image from "next/image";
import Link from "next/link";
import { notFound, permanentRedirect } from "next/navigation";
import { Nav } from "@/components/site/Nav";
import { Footer } from "@/components/site/Footer";
import { PortableBody } from "@/components/blog/PortableBody";
import { RelatedPosts } from "@/components/blog/RelatedPosts";
import { COLORS } from "@/components/site/constants";
import { getAllPosts, getPost, getRelatedPosts, urlFor } from "@/lib/sanity";

const SITE = "https://yourdomain.com";

export async function generateStaticParams() {
  try {
    const posts = await getAllPosts();
    return posts.map((p) => ({ slug: p.slug.current }));
  } catch {
    return [];
  }
}

export async function generateMetadata({
  params,
}: {
  params: Promise<{ slug: string }>;
}): Promise<Metadata> {
  const { slug } = await params;
  let post = null;
  try {
    post = await getPost(slug);
  } catch {
    // fall through to the not-found metadata
  }

  if (!post) {
    return { title: "Post Not Found | YourBrand Blog" };
  }

  const title = post.metaTitle || post.title;
  const description =
    post.metaDescription || post.excerpt || `Read ${post.title} on the YourBrand blog.`;
  const ogImage = post.mainImage?.asset
    ? urlFor(post.mainImage).width(1200).height(630).url()
    : undefined;

  return {
    title,
    description,
    alternates: { canonical: `${SITE}/blog/${slug}` },
    authors: post.author ? [{ name: post.author.name }] : undefined,
    openGraph: {
      type: "article",
      title,
      description,
      url: `${SITE}/blog/${slug}`,
      siteName: "YourBrand",
      publishedTime: post.publishedAt,
      images: ogImage ? [{ url: ogImage, width: 1200, height: 630, alt: post.title }] : undefined,
    },
    twitter: {
      card: "summary_large_image",
      title,
      description,
      images: ogImage ? [ogImage] : undefined,
    },
  };
}

function readingTime(body: unknown) {
  if (!body) return 1;
  const words = JSON.stringify(body).split(/\s+/).length;
  return Math.max(1, Math.ceil(words / 200));
}

function formatDate(iso?: string) {
  if (!iso) return "Recently published";
  const d = new Date(iso);
  if (d.getFullYear() < 2000) return "Recently published";
  return d.toLocaleDateString("en-US", { day: "numeric", month: "short", year: "numeric" });
}

const heroSection: CSSProperties = {
  position: "relative",
  paddingTop: 150,
  paddingBottom: 64,
  backgroundImage:
    "linear-gradient(rgba(8,10,22,.76), rgba(8,10,22,.86)), url(/blog-hero.jpg)",
  backgroundSize: "cover",
  backgroundPosition: "center",
};
const darkChip: CSSProperties = {
  padding: "4px 11px",
  borderRadius: 100,
  background: "rgba(255,255,255,.1)",
  border: "1px solid rgba(255,255,255,.18)",
  color: "#cdd9ff",
  fontSize: 10,
  fontWeight: 600,
  letterSpacing: "0.1em",
  textTransform: "uppercase",
  fontFamily: "var(--font-geist-mono)",
};

export default async function BlogPostPage({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  if (slug !== slug.toLowerCase()) permanentRedirect(`/blog/${slug.toLowerCase()}`);
  const post = await getPost(slug);
  if (!post) notFound();

  const related = await getRelatedPosts(slug, 3).catch(() => []);

  const authorSlug = post.author?.name
    ? post.author.name.toLowerCase().replace(/\s+/g, "-")
    : null;

  const jsonLd = {
    "@context": "https://schema.org",
    "@type": "BlogPosting",
    "@id": `${SITE}/blog/${slug}#article`,
    mainEntityOfPage: {
      "@type": "WebPage",
      "@id": `${SITE}/blog/${slug}`,
    },
    headline: post.title,
    description: post.excerpt || post.metaDescription,
    image: post.mainImage?.asset
      ? {
          "@type": "ImageObject",
          url: urlFor(post.mainImage).width(1200).height(630).url(),
        }
      : undefined,
    datePublished: post.publishedAt,
    dateModified: post._updatedAt || post.publishedAt,
    author: authorSlug
      ? {
          "@type": "Person",
          "@id": `${SITE}/author/${authorSlug}#person`,
          name: post.author!.name,
          url: `${SITE}/author/${authorSlug}`,
          ...(post.author!.url ? { sameAs: [post.author!.url] } : {}),
        }
      : undefined,
    publisher: { "@id": `${SITE}/#organization` },
    speakable: {
      "@type": "SpeakableSpecification",
      cssSelector: [".post-intro", ".key-takeaway"],
    },
  };

  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
      />
      <Nav currentNav="Blog" />

      <header className="fs-blog-post-hero" style={heroSection}>
        <div
          className="fs-blog-post-hero-inner"
          style={{ maxWidth: 900, margin: "0 auto", padding: "0 40px" }}
        >
          <Link
            href="/blog"
            style={{
              fontFamily: "var(--font-geist-mono)",
              fontSize: 12,
              letterSpacing: "0.1em",
              textTransform: "uppercase",
              color: "rgba(255,255,255,.6)",
              textDecoration: "none",
            }}
          >
            ← All articles
          </Link>

          {post.categories && post.categories.length > 0 && (
            <div style={{ display: "flex", flexWrap: "wrap", gap: 8, margin: "22px 0 0" }}>
              {post.categories.map((cat) => (
                <span key={cat.slug?.current || cat.title} style={darkChip}>
                  {cat.title}
                </span>
              ))}
            </div>
          )}

          <h1
            style={{
              color: "#fff",
              fontSize: "clamp(32px, 5vw, 56px)",
              fontWeight: 600,
              letterSpacing: "-0.04em",
              lineHeight: 1.1,
              margin: "20px 0 0",
            }}
          >
            {post.title}
          </h1>

          <div
            style={{
              display: "flex",
              flexWrap: "wrap",
              alignItems: "center",
              gap: 10,
              margin: "18px 0 0",
              fontFamily: "var(--font-geist-mono)",
              fontSize: 13,
              color: "rgba(255,255,255,.62)",
            }}
          >
            {authorSlug ? (
              <Link
                href={`/author/${authorSlug}`}
                style={{ color: "inherit", textDecoration: "underline", textUnderlineOffset: 3 }}
              >
                {post.author!.name}
              </Link>
            ) : (
              <span>YourBrand</span>
            )}
            <span style={{ color: "rgba(255,255,255,.3)" }}>·</span>
            <span>{formatDate(post.publishedAt)}</span>
            <span style={{ color: "rgba(255,255,255,.3)" }}>·</span>
            <span>{readingTime(post.body)} min read</span>
          </div>
        </div>
      </header>

      <main
        className="fs-blog-post-main"
        style={{ maxWidth: 900, margin: "0 auto", padding: "0 40px 110px" }}
      >
        {post.mainImage?.asset && (
          <div
            style={{
              position: "relative",
              width: "100%",
              aspectRatio: "16 / 9",
              marginTop: -48, // pulls the cover up over the dark hero
              borderRadius: 16,
              overflow: "hidden",
              border: `1px solid ${COLORS.border}`,
              boxShadow: "0 30px 60px rgba(8,32,96,.20)",
              background: COLORS.softGray,
            }}
          >
            <Image
              src={urlFor(post.mainImage).width(1600).height(900).url()}
              alt={post.title}
              fill
              priority
              sizes="(max-width: 900px) 100vw, 820px"
              style={{ objectFit: "cover" }}
            />
          </div>
        )}

        <article style={{ marginTop: 40 }}>
          {post.excerpt && (
            <p
              className="post-intro"
              style={{
                fontSize: 20,
                lineHeight: 1.6,
                fontWeight: 500,
                color: "#0f1018",
                margin: "0 0 28px",
              }}
            >
              {post.excerpt}
            </p>
          )}
          {(post.metaDescription || post.excerpt) && (
            <p
              className="key-takeaway"
              style={{
                fontSize: 15,
                lineHeight: 1.6,
                color: "#0f1018",
                borderLeft: "3px solid #2563eb",
                paddingLeft: 16,
                margin: "0 0 32px",
                fontStyle: "italic",
              }}
            >
              {post.metaDescription || post.excerpt}
            </p>
          )}
          <PortableBody value={post.body} />
        </article>

        {post.author?.bio && (
          <div
            style={{
              marginTop: 48,
              paddingTop: 32,
              borderTop: `1px solid ${COLORS.border}`,
              display: "flex",
              gap: 16,
              alignItems: "flex-start",
            }}
          >
            {post.author.image?.asset && (
              <div
                style={{
                  position: "relative",
                  width: 56,
                  height: 56,
                  borderRadius: "50%",
                  overflow: "hidden",
                  flexShrink: 0,
                }}
              >
                <Image
                  src={urlFor(post.author.image).width(112).height(112).url()}
                  alt={post.author.name}
                  fill
                  sizes="56px"
                  style={{ objectFit: "cover" }}
                />
              </div>
            )}
            <div>
              <div
                style={{
                  fontSize: 12,
                  fontFamily: "var(--font-geist-mono)",
                  textTransform: "uppercase",
                  letterSpacing: "0.14em",
                  color: COLORS.subtle,
                }}
              >
                About the author
              </div>
              <div style={{ fontSize: 18, fontWeight: 600, color: "#0f1018", margin: "4px 0 6px" }}>
                {post.author.name}
              </div>
              <div style={{ fontSize: 14, color: COLORS.gray }}>
                <PortableBody value={post.author.bio} />
              </div>
            </div>
          </div>
        )}

        <RelatedPosts posts={related} />
      </main>

      <Footer />
    </>
  );
}
```

**Why each piece exists**

| Piece | Reason |
|---|---|
| `generateStaticParams()` | Pre-renders every existing post at build; new posts still work via ISR fallback |
| `permanentRedirect` on non-lowercase slug | Kills duplicate-content URLs (`/blog/My-Post` → `/blog/my-post`) with a 308 |
| `notFound()` | Unknown slug renders `app/not-found.tsx` with a proper 404 status |
| `.catch(() => [])` on related | A related-posts failure must never break the article |
| `marginTop: -48` on the cover | Overlaps the dark hero — the signature visual of this layout |
| `.post-intro` / `.key-takeaway` classes | Referenced by the `speakable` JSON-LD for voice assistants |
| `priority` on the cover `<Image>` | It's the LCP element |
| `_updatedAt` → `dateModified` | Google uses it for freshness |
| `publisher: { "@id": ... }` | Points at the Organization node defined once in the root layout (see C3) |

> **A note on `readingTime`:** it stringifies the Portable Text JSON and counts whitespace-separated
> tokens, so it counts markup keys too and over-estimates. The homepage uses the more accurate
> GROQ `pt::text()` version. If you want accuracy here, project
> `"readMinutes": round(length(pt::text(body)) / 1100)` in `getPost` and use that instead.

---

# Part C — SEO plumbing

## C1. `app/sitemap.ts`

```ts
import type { MetadataRoute } from "next";
import { getAllPosts } from "@/lib/sanity";

// Canonical origin. Hardcoded on purpose — a sitemap must always point at the
// production domain. Matches `metadataBase` in app/layout.tsx.
const SITE = "https://yourdomain.com";

// Re-render the sitemap alongside the blog so new posts appear within a minute
// without a rebuild (mirrors `revalidate` in app/blog/page.tsx).
export const revalidate = 60;

/** Parse an ISO date, falling back to "now" for missing/garbage values. */
function safeDate(iso?: string): Date {
  if (!iso) return new Date();
  const d = new Date(iso);
  return Number.isNaN(d.getTime()) ? new Date() : d;
}

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const now = new Date();

  const staticRoutes: MetadataRoute.Sitemap = [
    { url: SITE, lastModified: now, changeFrequency: "weekly", priority: 1 },
    { url: `${SITE}/blog`, lastModified: now, changeFrequency: "daily", priority: 0.7 },
    { url: `${SITE}/privacy`, lastModified: now, changeFrequency: "yearly", priority: 0.3 },
    { url: `${SITE}/terms`, lastModified: now, changeFrequency: "yearly", priority: 0.3 },
  ];

  let posts: Awaited<ReturnType<typeof getAllPosts>> = [];
  try {
    posts = await getAllPosts();
  } catch {
    // CMS unreachable — ship the static routes rather than failing the sitemap.
  }

  const postRoutes: MetadataRoute.Sitemap = posts
    .filter((p) => p.slug?.current)
    .map((p) => ({
      url: `${SITE}/blog/${p.slug.current}`,
      lastModified: safeDate(p.publishedAt),
      changeFrequency: "monthly",
      priority: 0.6,
    }));

  return [...staticRoutes, ...postRoutes];
}
```

## C2. `app/robots.ts`

```ts
import type { MetadataRoute } from "next";

const SITE = "https://yourdomain.com";

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      { userAgent: "*", allow: "/" },
      // AI answer-engine crawlers — allow for citation
      { userAgent: "GPTBot", allow: "/" },
      { userAgent: "OAI-SearchBot", allow: "/" },
      { userAgent: "ChatGPT-User", allow: "/" },
      { userAgent: "PerplexityBot", allow: "/" },
      { userAgent: "Google-Extended", allow: "/" },
      { userAgent: "ClaudeBot", allow: "/" },
      { userAgent: "Applebot-Extended", allow: "/" },
    ],
    sitemap: `${SITE}/sitemap.xml`,
  };
}
```

## C3. Organization / WebSite JSON-LD in the root layout

Each article's JSON-LD references `publisher: { "@id": "https://yourdomain.com/#organization" }`.
That node must be defined **once**, site-wide, in `app/layout.tsx`:

```tsx
export const metadata: Metadata = {
  metadataBase: new URL("https://yourdomain.com"),
  title: "YourBrand",
  description: "...",
};

// inside <head>:
<script
  type="application/ld+json"
  dangerouslySetInnerHTML={{
    __html: JSON.stringify({
      "@context": "https://schema.org",
      "@graph": [
        {
          "@type": "Organization",
          "@id": "https://yourdomain.com/#organization",
          name: "YourBrand",
          url: "https://yourdomain.com",
          logo: {
            "@type": "ImageObject",
            url: "https://yourdomain.com/logo.png",
            width: 512,
            height: 512,
          },
          email: "support@yourdomain.com",
          description: "...",
          sameAs: ["https://linkedin.com/company/...", "https://instagram.com/..."],
        },
        {
          "@type": "WebSite",
          "@id": "https://yourdomain.com/#website",
          url: "https://yourdomain.com",
          name: "YourBrand",
          publisher: { "@id": "https://yourdomain.com/#organization" },
        },
      ],
    }),
  }}
/>
```

`metadataBase` is required for relative OG image URLs to resolve.

## C4. Author pages

The article page links every byline to `/author/<name-slugified>` and emits a `Person` node with
`@id` `<SITE>/author/<slug>#person`. **Those pages must exist**, or you ship links to 404s and a
schema `@id` that resolves to nothing.

The reference project uses a **static** page per author (`app/author/prasad-pol/page.tsx`).
Simplest faithful port:

```tsx
import type { Metadata } from "next";
import { Nav } from "@/components/site/Nav";
import { Footer } from "@/components/site/Footer";
import { COLORS } from "@/components/site/constants";

const SITE = "https://yourdomain.com";

export const metadata: Metadata = {
  title: "Jane Doe — Content Lead | YourBrand",
  description: "Jane Doe is ... Author at YourBrand.",
  alternates: { canonical: `${SITE}/author/jane-doe` },
};

const personJsonLd = {
  "@context": "https://schema.org",
  "@type": "Person",
  "@id": `${SITE}/author/jane-doe#person`,
  name: "Jane Doe",
  url: `${SITE}/author/jane-doe`,
  jobTitle: "Content Lead",
  description: "...",
  knowsAbout: ["Topic A", "Topic B"],
  worksFor: { "@id": `${SITE}/#organization` },
  sameAs: ["https://linkedin.com/in/janedoe"],
};

export default function AuthorPage() {
  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(personJsonLd) }}
      />
      <Nav />
      <main style={{ maxWidth: 720, margin: "0 auto", padding: "120px 40px 100px" }}>
        <p
          style={{
            fontFamily: "var(--font-geist-mono)",
            fontSize: 11,
            letterSpacing: "0.14em",
            textTransform: "uppercase",
            color: COLORS.subtle,
            marginBottom: 16,
          }}
        >
          Author
        </p>
        <h1
          style={{
            fontSize: "clamp(32px, 5vw, 52px)",
            fontWeight: 600,
            letterSpacing: "-0.04em",
            color: "#0f1018",
            margin: "0 0 16px",
          }}
        >
          Jane Doe
        </h1>
        <p
          style={{
            fontSize: 16,
            fontFamily: "var(--font-geist-mono)",
            color: COLORS.subtle,
            margin: "0 0 24px",
          }}
        >
          Content Lead
        </p>
        <p style={{ fontSize: 17, lineHeight: 1.7, color: COLORS.gray, maxWidth: 600 }}>
          Bio paragraph…
        </p>
      </main>
      <Footer />
    </>
  );
}
```

**Better (dynamic) alternative** if you'll have many authors — add `app/author/[slug]/page.tsx`
driven by a `getAuthor(slug)` query, and switch the byline link to `post.author.slug.current`:

```ts
export async function getAuthor(slug: string) {
  return client.fetch(
    `*[_type == "author" && slug.current == $slug][0]{
      name, slug, image, bio, url,
      "posts": *[_type == "post" && author._ref == ^._id] | order(publishedAt desc){
        _id, title, slug, excerpt, mainImage, publishedAt,
        "author": author->name, "categories": categories[]->title
      }
    }`,
    { slug },
    { next: { revalidate: 300 } },
  );
}
```

---

# Part D — Homepage "latest posts" section

A distinct, larger-format card set for the marketing homepage. Server component; hides itself
entirely if there are no posts.

`components/home/Blog.tsx`:

```tsx
import Image from "next/image";
import Link from "next/link";
import { Reveal } from "@/components/site/Reveal";
import { SectionHead } from "@/components/site/SectionHead";
import { COLORS } from "@/components/site/constants";
import { getLatestPosts, urlFor, type LatestPost } from "@/lib/sanity";

// Rotating tag colours so three consecutive cards don't look identical.
const TAG_PALETTE = [
  { bg: COLORS.redBg, color: COLORS.red },
  { bg: COLORS.blueBg, color: COLORS.blue },
  { bg: "#f1ecff", color: COLORS.purple },
];

function formatDate(iso?: string) {
  if (!iso) return "Recently";
  const d = new Date(iso);
  if (d.getFullYear() < 2000) return "Recently";
  return d.toLocaleDateString("en-US", { day: "numeric", month: "short", year: "numeric" });
}

function metaLabel(p: LatestPost) {
  const mins = p.readMinutes ?? 0;
  return mins > 0 ? `${mins} min read` : formatDate(p.publishedAt);
}

export async function Blog() {
  let posts: LatestPost[] = [];
  try {
    posts = await getLatestPosts(3);
  } catch {
    // CMS unreachable — fall through to the empty check below.
  }

  // No posts yet (or CMS down) — hide the section rather than show placeholders.
  if (posts.length === 0) return null;

  return (
    <section
      className="fs-section"
      style={{ maxWidth: 1280, margin: "0 auto", padding: "80px 40px" }}
    >
      <SectionHead
        tag="BLOG"
        title="Playbooks, deep-dives, and tactics"
        sub="Actionable guides written by people who actually do the work."
      />
      <div className="fs-grid-3" style={{ marginTop: 56 }}>
        {posts.map((p, i) => {
          const tag = p.categories?.[0];
          const palette = TAG_PALETTE[i % TAG_PALETTE.length];
          const hasImage = Boolean(p.mainImage?.asset);
          return (
            <Reveal key={p._id} delay={(i % 3) * 0.08}>
              <Link
                href={`/blog/${p.slug.current}`}
                className="fs-card"
                style={{
                  display: "block",
                  background: COLORS.softGray,
                  borderRadius: 12,
                  overflow: "hidden",
                  textDecoration: "none",
                  color: "inherit",
                  height: "100%",
                }}
              >
                {hasImage ? (
                  <Image
                    src={urlFor(p.mainImage).width(800).height(593).url()}
                    alt={p.title}
                    width={800}
                    height={593}
                    sizes="(max-width: 1000px) 50vw, 400px"
                    style={{
                      width: "calc(100% - 16px)",
                      height: "auto",
                      margin: 8,
                      borderRadius: 12,
                      aspectRatio: "1.35",
                      objectFit: "cover",
                      display: "block",
                    }}
                  />
                ) : (
                  <div
                    style={{
                      width: "calc(100% - 16px)",
                      margin: 8,
                      borderRadius: 12,
                      aspectRatio: "1.35",
                      background: `linear-gradient(135deg, ${COLORS.blueBg}, #dbe6ff)`,
                    }}
                  />
                )}
                <div style={{ padding: "0 30px 30px" }}>
                  <div
                    style={{ display: "flex", alignItems: "center", gap: 12, marginBottom: 16 }}
                  >
                    {tag && (
                      <span
                        style={{
                          background: palette.bg,
                          color: palette.color,
                          padding: "6px 16px",
                          borderRadius: 100,
                          fontSize: 14,
                          fontWeight: 500,
                        }}
                      >
                        {tag}
                      </span>
                    )}
                    <span style={{ width: 6, height: 6, borderRadius: 3, background: "#3d3d3d" }} />
                    <span style={{ fontSize: 14, color: COLORS.black }}>{metaLabel(p)}</span>
                  </div>
                  <h5
                    style={{
                      fontSize: 22,
                      fontWeight: 600,
                      letterSpacing: "-0.04em",
                      margin: "0 0 12px",
                    }}
                  >
                    {p.title}
                  </h5>
                  {p.excerpt && (
                    <p style={{ color: COLORS.gray, fontSize: 16, lineHeight: 1.5, margin: 0 }}>
                      {p.excerpt}
                    </p>
                  )}
                </div>
              </Link>
            </Reveal>
          );
        })}
      </div>
    </section>
  );
}
```

Supporting components, if you don't already have them:

<details>
<summary><code>components/site/Reveal.tsx</code> — scroll-in animation wrapper</summary>

```tsx
"use client";

import { useEffect, useRef, useState } from "react";

export function Reveal({
  children,
  delay = 0,
  style,
}: {
  children: React.ReactNode;
  delay?: number;
  style?: React.CSSProperties;
}) {
  const ref = useRef<HTMLDivElement>(null);
  const [shown, setShown] = useState(false);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    const obs = new IntersectionObserver(
      (e) => e[0].isIntersecting && setShown(true),
      { threshold: 0.15 }
    );
    obs.observe(el);
    return () => obs.disconnect();
  }, []);

  return (
    <div
      ref={ref}
      style={{
        opacity: shown ? 1 : 0,
        transform: shown ? "translateY(0)" : "translateY(40px)",
        transition: `opacity .7s ${delay}s ease, transform .7s ${delay}s ease`,
        ...style,
      }}
    >
      {children}
    </div>
  );
}
```
</details>

<details>
<summary><code>components/site/SectionHead.tsx</code> — centered section heading</summary>

```tsx
import { Reveal } from "./Reveal";
import { COLORS } from "./constants";

export function SectionHead({
  tag,
  title,
  sub,
  titleColor,
  subColor,
}: {
  tag?: string;
  title: string;
  sub?: string;
  titleColor?: string;
  subColor?: string;
}) {
  return (
    <Reveal style={{ maxWidth: 700, textAlign: "center", margin: "0 auto" }}>
      {tag && (
        <span
          style={{
            fontFamily: "var(--font-geist-mono)",
            fontSize: 12,
            letterSpacing: "0.16em",
            textTransform: "uppercase",
            color: COLORS.blue,
          }}
        >
          {tag}
        </span>
      )}
      <h2
        style={{
          fontSize: "clamp(28px, 4vw, 56px)",
          fontWeight: 600,
          letterSpacing: "-0.04em",
          lineHeight: 1.1,
          margin: "20px 0 12px",
          ...(titleColor ? { color: titleColor } : {}),
        }}
      >
        {title}
      </h2>
      {sub && <p style={{ color: subColor ?? COLORS.gray, fontSize: 18, lineHeight: 1.5 }}>{sub}</p>}
    </Reveal>
  );
}
```
</details>

Then render `<Blog />` inside `app/page.tsx`.

---

# Part E — Adapting this to your project

Search-and-replace list — do these in order and the blog will be fully yours:

| Find | Replace with |
|---|---|
| `https://freeserp.com` / `https://yourdomain.com` | your production origin (appears in the blog page, article page, sitemap, robots, layout, author page) |
| `FreeSERP` / `YourBrand` | your brand name |
| `your_project_id` | your Sanity projectId |
| `fs-` CSS prefix | your own prefix (or leave it) |
| `COLORS` hex values | your palette |
| `var(--font-geist-mono)` | your mono font variable, or delete the mono styling |
| hero `backgroundImage` URL | your own hero image (put it in `public/`) |
| `/author/...` static page | your author(s), or the dynamic route from C4 |

**Files created, at a glance**

```
lib/sanity.ts                          # client + types + GROQ queries
components/site/constants.ts           # COLORS (if not already present)
components/blog/PostCard.tsx
components/blog/PostsGrid.tsx          # "use client"
components/blog/PortableBody.tsx
components/blog/RelatedPosts.tsx
components/blog/blog.css
components/home/Blog.tsx               # optional homepage section
app/blog/page.tsx
app/blog/[slug]/page.tsx
app/sitemap.ts
app/robots.ts
app/author/<slug>/page.tsx             # or app/author/[slug]/page.tsx
next.config.ts                         # images.remotePatterns += cdn.sanity.io
app/globals.css                        # @import blog.css
.env                                   # NEXT_PUBLIC_SANITY_* vars
```

**Tuning knobs**

| Knob | Where | Default |
|---|---|---|
| Listing freshness | `revalidate` in `app/blog/page.tsx` + `getAllPosts` | 60s |
| Article freshness | `revalidate` in `getPost` | 30s |
| Sitemap freshness | `revalidate` in `app/sitemap.ts` | 60s |
| Related posts count | `getRelatedPosts(slug, N)` | 3 |
| Homepage teasers | `getLatestPosts(N)` | 3 |
| Read-time divisor | `1100` chars/min (GROQ), `200` wpm (page) | — |
| Cards per row | `.fs-grid-3` | 3 → 2 → 1 |

---

# Part F — Verification checklist

Run through this after implementing:

- [ ] `npm run dev` → `/blog` lists your published posts
- [ ] Typing in the search box filters by title, excerpt, author and category
- [ ] Clearing the search restores the full list; a no-match query shows **Clear search**
- [ ] With zero posts, `/blog` shows "No posts yet" (not a crash)
- [ ] Clicking a card opens `/blog/<slug>` and renders the body
- [ ] Body renders: h2/h3/h4, paragraphs, bullet + numbered lists, blockquote, inline code,
      code block, bold, italic, internal link, external link (opens in a new tab),
      inline image with caption, YouTube embed, Vimeo embed
- [ ] Post images load (if not → `cdn.sanity.io` missing from `next.config.ts`)
- [ ] `/blog/My-Post` 308-redirects to `/blog/my-post`
- [ ] `/blog/does-not-exist` returns a real 404
- [ ] `view-source:` on an article shows `<script type="application/ld+json">` with `BlogPosting`
- [ ] Validate that JSON-LD at https://validator.schema.org and Google's Rich Results Test
- [ ] `/sitemap.xml` includes every post URL with a sane `lastmod`
- [ ] `/robots.txt` points at the sitemap
- [ ] OG preview looks right (paste the URL into an OG debugger or a Slack DM)
- [ ] Related posts appear at the bottom of an article and exclude the current post
- [ ] Author byline links to a page that exists
- [ ] Resize to 375px wide: hero padding shrinks, grid is 1-up, code blocks scroll, no
      horizontal page scroll
- [ ] Publish a new post in the Studio → it appears on `/blog` within ~60s **without** a redeploy
- [ ] `npm run build` succeeds and pre-renders each post (check the route list output)

---

# Part G — Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Images 400 / "hostname not configured" | `cdn.sanity.io` not allowlisted | Add it to `images.remotePatterns` in `next.config.ts` (B3) |
| `/blog` always empty, no error | Dataset is private, or wrong `projectId`/`dataset` | `npx sanity dataset visibility set production public`; check `.env` |
| Posts appear in the Studio but not on the site | Documents are drafts | Hit **Publish** — the client uses `perspective: "published"` |
| Content updates don't show | ISR window not elapsed | Wait out `revalidate`, or hard-refresh in dev; consider a webhook + `revalidatePath` for instant updates |
| `Cannot read properties of undefined (reading 'current')` | A post has no slug | The queries assume `slug` exists — add `&& defined(slug.current)` to `getAllPosts` and require the field in the schema |
| Article 404s but the post exists | Slug has uppercase characters | The lowercase redirect makes mixed-case slugs unreachable — re-slugify in the Studio (the `slugify` option in A2 prevents this) |
| Custom Portable Text block renders nothing | No renderer for that `_type` | Add a matching entry under `components.types` in `PortableBody.tsx` |
| Hydration mismatch on dates | `toLocaleDateString` differs server/client | Pin the locale (it already is: `"en-US"`); don't use the visitor's locale |
| `useState` error in a blog component | Missing `"use client"` | Only `PostsGrid` (and `Reveal`) need it; keep everything else on the server |
| Build fails during `generateStaticParams` | CMS unreachable at build time | The `try/catch` already returns `[]` — check it wasn't removed |

**Optional upgrade — instant publishing via webhook**

Instead of waiting for ISR, add `app/api/revalidate/route.ts` that verifies a Sanity webhook
signature and calls `revalidatePath("/blog")` + `revalidatePath("/blog/" + slug)` +
`revalidatePath("/sitemap.xml")`. Configure the webhook in Sanity → API → Webhooks to fire on
`post` document changes. Keep the `revalidate` values as a safety net.

---

# Appendix — design decisions & why

1. **Sanity, not MDX/local files.** Non-developers publish without a PR or a deploy. The tradeoff
   — a network dependency — is absorbed by ISR caching plus `try/catch` fallbacks everywhere.

2. **No API token.** The dataset is public and `perspective: "published"` hides drafts, so the
   frontend needs no secret. That means the queries are safe to run at build, on the server, and
   from Edge alike.

3. **`useCdn: false` + Next ISR.** Two caches stacked would make invalidation confusing. Next's
   fetch cache is the one that matters, and it's per-query tunable.

4. **Server components by default.** The only `"use client"` in the blog is `PostsGrid` (search
   state) — so post HTML ships fully rendered and the JS payload stays tiny.

5. **Client-side search over a search API.** For a blog of this size, sending all posts once and
   filtering in `useMemo` is instant and free. Above ~500 posts, switch to a GROQ `match` query
   or pagination.

6. **Flattened card projections.** `"author": author->name` keeps `BlogPost` a flat, cheap type;
   the article page uses a separate, richer `BlogPostFull`.

7. **Inline styles + a small CSS file.** Styles live next to the markup they belong to; the CSS
   file exists only for what inline styles can't do (media queries, `:hover`, line clamping) — and
   uses `!important` there because inline styles have higher specificity.

8. **Defensive everywhere.** Every fetch is caught, every optional field is guarded, every bad
   date falls back to "Recently", every malformed embed renders `null`. A CMS is user input.

9. **SEO is not an afterthought.** Canonical URLs, OG + Twitter cards, `BlogPosting` with linked
   `Person` and `Organization` `@id`s, `speakable` selectors, an ISR sitemap and an AI-crawler-
   friendly robots.txt all ship with the first post.
