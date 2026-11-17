# Blunt Ink Co.

Marketing/storefront site for Blunt Ink Co.'s coloring book line ("Corporate Swear Words," "Parenting Swear Words," "Christmas Swear Words," "Cozy Sarcasm"). Single static page: hero, product grid, and a "Coming Next" teaser section.

## Stack

- Plain static HTML/CSS/JS — no framework, no build step.
- Deployed as a Cloudflare Worker using [Wrangler](https://developers.cloudflare.com/workers/wrangler/)'s static-assets mode (see `wrangler.jsonc`).
- Book cover images are embedded directly in the HTML as base64 data URIs (no separate image files are fetched at runtime).

## Repo structure

```
index.html          # legacy/root copy — NOT the one that gets served (see Deployment)
public/
  index.html         # the file Cloudflare Workers actually serves
package.json         # just the `deploy` script
wrangler.jsonc        # Workers config — assets.directory points at ./public
source-covers/        # reference copies of the original (pre-fix) cover JPEGs
```

**Important:** `index.html` exists in both the repo root and in `public/`. Only `public/index.html` is deployed (`wrangler.jsonc` → `assets.directory: "./public"`). The root copy is stale/legacy duplication from an earlier workflow and is kept in sync by hand when edits are made, but it is not itself served. See `.claude/PROJECT.md` for the gotcha this causes when editing embedded images.

## Local development

No dev server is configured. The page is static HTML; open `public/index.html` directly in a browser to preview, or use `wrangler dev` if you want it served through the Workers runtime locally.

## Deployment

Deploys automatically via Cloudflare Workers Builds' Git integration: any push to `main` triggers a build and deploy of `public/` to the live Worker. There is no manual `wrangler deploy` step needed for normal changes — pushing to `main` is the deploy.

To deploy manually instead:

```
npm run deploy   # runs `wrangler deploy`
```

## Editing the book cover images

The four cover images are base64-encoded JPEGs inlined directly in `index.html` (root) and `public/index.html`. There is **no single `<img>` per cover** — each cover's image data appears multiple times in the file: once in the main product grid (with a descriptive `alt` attribute, e.g. `alt="Corporate Swear Words cover"`) and again, with empty `alt`, in the decorative "Coming Next" fanned-stack teaser. When replacing a cover image, do a **global byte-level find/replace of the old base64 string**, not an alt-text-scoped replace — otherwise the decorative duplicates will silently keep the old image. See `.claude/PROJECT.md` for details and a worked example.
