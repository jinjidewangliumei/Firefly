# AGENTS.md — Firefly

Astro 7 + Svelte 5 static blog theme (upstream: CuteLeaf/Firefly); this checkout is a personal fork. The repo root is this directory — run all project commands here. Content, config and comments are Chinese-first; UI i18n: en / zh_TW / ja / ko / ru (non-zh locales are AI-translated).

Upstream theme docs (Chinese, author-maintained): https://docs-firefly.cuteleaf.cn/zh/ — consult these for feature/config questions before guessing.

## Commands

| Command | Notes |
| --- | --- |
| `pnpm dev` | dev server on `localhost:4321` |
| `pnpm check` | `astro check`; baseline is 0 errors/warnings; syncs content and regenerates `.astro/` types |
| `pnpm type-check` | `tsc --noEmit --isolatedDeclarations` over `src/` + `scripts/`; exported symbols need explicit types |
| `pnpm lint` / `pnpm format` | Biome over `./src ./scripts` — writes fixes in place |
| `pnpm build` | full pipeline → `dist/` (see below) |
| `pnpm preview` | serve the production build |
| `pnpm new-post <file>` / `pnpm new-d "<text>"` | scaffold a post / dynamic entry (`new-dynamic` is identical) |
| `pnpm lqips` / `pnpm github-cards` | regenerate committed JSON in `src/constants/` |

- pnpm is enforced (`preinstall` runs `only-allow pnpm`); Node >= 22.23, pnpm 11.22.
- No test framework. Verification = `pnpm check` + `pnpm type-check` (+ `pnpm build` for content/asset changes); use `pnpm dev`/`preview` and screenshots for UI changes.
- CI (`.github/workflows/`) runs `pnpm astro check`, `pnpm astro build`, and `biome ci ./src` — not the full `pnpm build` chain.

## This machine / checkout gotchas

- PowerShell blocks `.ps1` shims (ExecutionPolicy): use `pnpm.cmd <script>` / `npx.cmd` — plain `pnpm` fails with a SecurityError.
- Platform binaries are missing from `node_modules` (`@biomejs/cli-win32-x64`, `@pagefind/windows-x64`, `@cloudflare/workerd-windows-64`); registry fetches fail here (ECONNRESET). So **`pnpm lint`/`format` fail** (`Cannot find module '@biomejs/cli-win32-x64/biome.exe'`) and **local `pnpm build` fails at the Pagefind step** (`node_modules\.bin\pagefind.CMD --version` errors); this is local-only — Cloudflare Pages builds in its own environment. `pnpm check`, `pnpm type-check`, `pnpm dev` work. Repair once the registry is reachable: delete `node_modules/.modules.yaml`, `.package-map.json`, `.pnpm-workspace-state-v1.json`, then `pnpm install` (otherwise pnpm says "Already up to date" and skips them).

## Build pipeline (`pnpm build`)

`generate-github-card-data.ts` → `generate-lqips.ts` → `generate-vndb-covers.ts` → `astro build` → `prune-pio-assets.ts` → `subset-fonts.ts` → `minify-inline-scripts.ts` → `run-pagefind.ts`

- GitHub cards: fetches repo data from the GitHub API (optional `GITHUB_TOKEN`); falls back to the committed cache → `src/constants/github-card-data.json`.
- Generated but committed (and Biome-ignored): `src/constants/lqips.json`, `github-card-data.json`, `icons-data.json` (no generator for icons here). `.astro/` and `public/vndb-covers/` are gitignored.
- `generate-vndb-covers.ts` only downloads when the VNDB page is enabled and `siteConfig.vndb` has a real `userId` + `downloadCovers`; it skips files that already exist.
- `prune-pio-assets.ts` deletes unused 看板娘 assets from `dist/` based on pio config — revisit it if you change pio settings.

## Architecture

- Astro pages/layouts + Svelte 5 islands (`client:*`). Swup handles SPA transitions over fixed containers from `astro.config.mjs` (`#swup-container`, `#banner-overlay-container`, `#banner-dim-container`, `#left-sidebar-dynamic`, `#right-sidebar-dynamic`, `#floating-toc-wrapper`) — UI outside those containers won't update on navigation.
- Config-driven: features are toggled in `src/config/*`, exported via the barrel `src/config/index.ts`; matching types in `src/types`. Markdown is extended by custom remark/rehype plugins in `src/plugins/`, wired in `astro.config.mjs`; code blocks use Expressive Code.
- Content collections (`src/content.config.ts`): `posts`, `dynamic`, `projects`, `spec` under `src/content/<name>/`.
- i18n: keys in `src/i18n/i18nKey.ts`, locales in `src/i18n/languages/`, lookup via `src/i18n/translation.ts`.
- Path aliases: `@/*` → `src/*`, plus `@components`, `@assets`, `@constants`, `@utils`, `@i18n`, `@layouts`.
- Scroll paths are perf-sensitive on mobile — do not regress: `fullscreen-wallpaper-utils.ts` quantizes `--fullscreen-blur` to 2px steps and caches `--overlay-blur` (no per-frame `getComputedStyle` or full-viewport blur writes); `grid-layout-utils.ts` `updateSidebarStickySpacing()` is the per-scroll path and must not read layout (state cached by `refreshSidebarStickyState()` at init/navigation only).

## Conventions, fork specifics, deployment

- Biome: tabs, double quotes, `organizeImports` on; lint rules relaxed for `.astro`/`.svelte`.
- Conventional Commits (`feat:`, `fix:`, `chore:` …), branch `master`; the PR template asks for validation commands + screenshots; discuss major features in an issue first (`CONTRIBUTING.md`).
- Personal fork settings: don't reset `src/config/siteConfig.ts` (site URL `https://4861000.xyz`, titles, page toggles) unless asked.
- Deploy: hosted on **Cloudflare Pages** — the dashboard builds this git repo (build command `pnpm build`, output dir `dist`; env vars live in the Pages dashboard, not the repo). `vercel.json` and `.github/workflows/deploy.yml` are unused upstream leftovers; `wrangler.toml`/`wrangler.jsonc` are not part of the deploy path — `CF_WORKERS` switches to the Workers adapter (`astro.config.mjs`) and is not used here.
- Never commit secrets; keys stay in platform env (`GITHUB_TOKEN` is build-time only).
- `CLAUDE.md` has longer architecture notes; keep it consistent when editing either file.
