# Project definition

**Repo:** `robertsonai/bluntinkco-site`
**What it is:** The public marketing/storefront site for Blunt Ink Co., a small coloring-book imprint. One static page — hero, a product grid of book covers, and a "Coming Next" teaser section with a fanned stack of the same covers.
**Live URL:** https://bluntink.co/
**Owner/operator context:** Maintained ad hoc by Adam (not a dedicated web team). No CI, no tests, no staging environment — `main` is production.

## Tech stack

- Plain static HTML. No framework, no bundler, no build step.
- Book cover artwork is embedded as base64 `data:image/jpeg;base64,...` URIs directly inside the HTML — there are no separate `/images/*.jpg` files served at runtime.
- Served as a Cloudflare Worker in static-assets mode via Wrangler.

## Repo layout

| Path | Purpose |
|---|---|
| `public/index.html` | **The file that is actually deployed.** `wrangler.jsonc`'s `assets.directory` points here. |
| `index.html` (repo root) | A duplicate/legacy copy of the same page, kept manually in sync. **Not served.** Exists from an earlier workflow; nobody has cleaned it up. Any edit to the page content or images must be applied to both files or they will drift. |
| `wrangler.jsonc` | Workers config. `assets.directory: "./public"`. |
| `package.json` | Just exposes `npm run deploy` → `wrangler deploy`. |
| `source-covers/` | Reference-quality copies of the original book cover JPEGs (added 2026-11-16), used as source material for AI-based image edits via Higgsfield. Not referenced by the site itself. |

## Deployment pattern

Deployment is **push-to-deploy**: Cloudflare Workers Builds has a Git integration configured on this repo, so any push to `main` automatically triggers a build and deploy of `public/` to the live Worker at bluntink.co. There is no PR-based staging step and no manual approval gate — a commit to `main` is live within roughly 20-30 seconds.

Manual deploy (rarely needed, e.g. if the Git integration is ever broken) is `npm run deploy`, which runs `wrangler deploy` and requires Cloudflare credentials that are not present in this environment.

**Write access constraint (as of 2026-11):** there is no valid `GH_TOKEN`/push credential available in the Claude Code sandbox for this repo. The only working write path found was GitHub's own web upload UI (`https://github.com/<repo>/upload/<branch>/<path>`) driven via browser automation — drag a local file onto the file-input, fill in the commit message field, click "Commit changes." This works for both adding new files and overwriting existing ones (GitHub matches by filename/path). If proper push credentials are ever configured, prefer a normal `git push` instead.

## Known gotcha: duplicated cover images inside the HTML

Each book cover's base64 image data is **not a single occurrence** in the HTML. As of 2026-11-16, each cover's bytes appear multiple times in the same file:

- Once in the main product grid, with a descriptive `alt` (e.g. `alt="Corporate Swear Words cover"`).
- Again, with **empty `alt`**, inside the "Coming Next" decorative fanned-stack section — and in some cases more than once there.

Concretely, at the time this was discovered: Corporate Swear Words appeared **4 times** total in the file, Cozy Sarcasm **3 times**, Parenting Swear Words **2 times**, Christmas Swear Words (never modified) presumably similarly duplicated.

**Consequence:** if you fix/replace a cover image by searching for its `alt` attribute and swapping only that one `<img>`'s `src`, the decorative duplicates elsewhere in the page will silently keep showing the old image. This exact mistake happened once in this repo's history (2026-11-16): three covers had a stray mouse-cursor icon baked into the JPEG pixels from a bad screenshot; an alt-scoped fix corrected the main grid but left the cursor visible in the "Coming Next" stack, which the site owner caught visually after the first "fix" went live.

**Correct approach when swapping an embedded cover image:**

1. Decode/identify the *exact* old base64 string for that cover (e.g. from the currently-live `public/index.html`).
2. Do a global string replace of that exact old base64 string with the new one, across the whole file — `html.count(old_b64)` before replacing to see how many occurrences actually exist, then `html.replace(old_b64, new_b64)`. Do **not** scope the replace to a single `<img alt="...">` match.
3. Apply the same replacement to **both** `public/index.html` and the root `index.html` (see Repo layout above).
4. Verify zero occurrences of the old base64 remain (`html.count(old_b64) == 0`) before committing.
5. After deploy, verify on the live site by hashing every `<img>` on the page (not just the ones with a matching `alt`) — fetch/decode each `<img>` element's bytes in the browser and SHA-256 them, confirming none still match the old image's hash.

## Image-editing workflow used for cover fixes

When a cover image itself needs correcting (not just a layout/HTML change), the working pattern established 2026-11-16:

1. Extract the current embedded JPEG bytes straight from `public/index.html` (regex on the `alt`-tagged `<img>`, base64-decode).
2. Push the raw flawed JPEG(s) to a `source-covers/` folder in this repo via the GitHub web-upload UI, to get a public `raw.githubusercontent.com` URL (Claude Code's own sandbox cannot reach Higgsfield's CDN/upload hosts directly due to egress restrictions, but can reach `raw.githubusercontent.com`).
3. `media_import_url` the raw GitHub URL into Higgsfield, then `generate_image` with model `gpt_image_2_6` using it as an `image_references` input — prompt for removing the specific defect while explicitly preserving composition/aspect ratio (pass `aspect_ratio` matching the source; the model defaults to a 1:1 square crop otherwise, which will distort a portrait cover).
4. Pull the result back via Higgsfield's `sandbox_exec` tool with `image_paths` — this is the one reliable bridge for getting bytes out of Higgsfield's network bubble and into Claude Code's own local filesystem (the CloudFront result URLs are blocked from Claude Code's own `curl`).
5. Resize/crop the AI output back to the exact original pixel dimensions (e.g. 600×776) using a "cover" crop (scale to cover, center-crop) rather than a naive stretch, to avoid distortion.
6. Do the global base64 replace described above, in both HTML files, and verify no old occurrences remain before pushing.
