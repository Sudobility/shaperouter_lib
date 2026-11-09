# ShapeRouter Lib

> **Git policy — never auto-commit or auto-push.** Leave work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when
> the user explicitly asks in that turn.** Approval for an earlier change does not carry
> forward, and finishing a task is not permission to commit it. Stay on the current
> branch — no feature branches for code changes.

Business-logic library for ShapeRouter: Zustand caching stores, manager hooks, endpoint
templates, and JSON-Schema utilities. npm name `@sudobility/shaperouter_lib`.

## Stack

- TypeScript, strict, `target: ES2020`, `module: ESNext` (ESM), `moduleResolution: bundler`.
- Zustand v5 for state. React 18+ / `@tanstack/react-query` 5+ are peer deps.
- Build = `tsc -p tsconfig.build.json` then `node scripts/fix-esm-imports.mjs`.
- Tests = Vitest. Package manager = **Bun only** (never npm/yarn/pnpm for installs).
- No bundled runtime dependencies — `zustand` and all `@sudobility/*` packages are peer deps.

## Commands

```bash
bun run build        # tsc + fix-esm-imports.mjs -> dist/
bun run build:watch  # tsc --watch
bun run clean        # rm -rf dist
bun run test:unit    # Vitest, single run (use this for CI/verification)
bun run test:watch   # Vitest watch mode
bun run typecheck    # tsc --noEmit
bun run lint         # eslint src --ext .ts,.tsx
bun run lint:fix
bun run format       # prettier --write
bun run verify       # typecheck + lint + test:unit + build  (run before pushing)
```

There is **no `test` or `test:run` script** — the script names are `test:unit` / `test:watch`.

## Layout

```
src/
  index.ts                     # re-exports ./business + ./utils
  business/
    stores/                    # Zustand caches (keys, projects, endpoints,
                               #   settings, analytics, providerModels)
    hooks/                     # manager hooks + useEndpointTester,
                               #   useBudgetTracker, useProject/EndpointTemplates
    templates/endpoint-templates.ts   # 9 ProjectTemplates + ALL_TEMPLATES + applyTemplate
  utils/
    schema-validation.ts       # validate*/generateSample*/keyToSampleLabel
    test-metrics.ts            # extractTestMetrics
```

Each subdir has an `index.ts` barrel; export new symbols from the local barrel AND they
flow up through `business/index.ts` / `utils/index.ts` to the package root.

## Architecture

- **Stores**: Zustand singletons holding client-side caches. Scope keys are per-store:
  `entitySlug` (keys, projects, analytics), `userId` (settings), composite
  `"entitySlug:projectId"` (endpoints, via a `makeCacheKey` helper), `LlmProvider`
  (providerModels). Every cache entry carries `cachedAt` (ms) for staleness.
- **Manager hooks** wrap `@sudobility/shaperouter_client` hooks with store caching. Pattern:
  config in (`baseUrl`, `networkClient`, scope key — `null` disables fetch, `token`,
  optional `testMode`/`autoFetch`), a `useEffect` syncs fresh server data into the store,
  and reads **prefer fresh server data, falling back to cache**. An empty server array
  means "loading", not "empty" — `isCached` is true only when server is empty and cache
  has data. Return value is `useMemo`-wrapped.
- **Dependency chain**: `shaperouter_app → shaperouter_lib → shaperouter_client → shaperouter_types`.

## Conventions

- Prettier: single quotes, semicolons, `printWidth` 80, `arrowParens: avoid`.
- Keep stores entity/user-scoped; never cache under the wrong scope key.
- Schema validation is **intentionally lightweight**: no `$ref`, no `allOf`/`oneOf`/`anyOf`,
  no `pattern`/`format`, no conditionals. Don't "fix" these as bugs.

## Gotchas

- **Package is public** — `package.json` sets `publishConfig.access: "public"` and is
  **published to GitHub Packages** (`.npmrc`: `@sudobility:registry=https://npm.pkg.github.com`),
  NOT the public npm registry. It is not a private package.
- **Publishing is CI-driven** — `.github/workflows/ci-cd.yml` calls the shared
  `johnqh/workflows` reusable workflow with `publish-target: github` on push to
  `main`/`develop`. Don't hand-run `npm publish` unless asked; version bumps go through CI.
- **ESM import fixup is required** — `tsc` (bundler resolution) emits extensionless relative
  imports that Node's ESM loader rejects; `scripts/fix-esm-imports.mjs` rewrites them to
  `./x.js` / `./dir/index.js` after every build. Never skip it; `bun run build` includes it.
- **localStorage persistence**: `useProviderModelsStore` (key `shaperouter-provider-models`,
  1-hour TTL via `isStale` + `DEFAULT_CACHE_MAX_AGE_MS`) and `useBudgetStore`
  (key `shaperouter-budgets`). All other stores are in-memory and lost on refresh.
  The persist adapter guards `globalThis.localStorage` so it no-ops in non-browser envs.
- **Stores are global singletons** — call `clearAll()` on logout / entity switch.
- **No budget period auto-reset** — `BudgetPeriod` is `'daily'|'weekly'|'monthly'`; the
  consumer handles period boundaries.
- **Peer dep on `shaperouter_client`** — when bumping it, re-verify the manager hooks here.

## Downstream

`shaperouter_app` consumes the stores, manager hooks, and templates. After changes:
`bun run verify`, then let CI publish, then `bun update @sudobility/shaperouter_lib` in the
app. For local iteration use `bun link` (link `shaperouter_client` first if changing it too).
