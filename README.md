# @sudobility/shaperouter_lib

Business-logic library for ShapeRouter: Zustand caching stores, manager hooks
(that wrap `@sudobility/shaperouter_client` API hooks with caching), pre-built
project/endpoint templates, and lightweight JSON-Schema validation utilities.

- **Language**: TypeScript (strict, `ES2020` target, `ESNext` modules / ESM)
- **State**: Zustand v5
- **Build**: `tsc` + a post-build ESM-import fixup script
- **Tests**: Vitest
- **Package manager**: Bun

## Installation

Published to **GitHub Packages** (not the public npm registry). Consuming
projects need an `.npmrc` pointing the `@sudobility` scope at GitHub Packages:

```
@sudobility:registry=https://npm.pkg.github.com
```

Then:

```bash
bun add @sudobility/shaperouter_lib
```

### Peer dependencies

These must be installed in the consuming app:

- `react` >= 18
- `@tanstack/react-query` >= 5
- `zustand` >= 5
- `@sudobility/shaperouter_client` (API hooks wrapped by the manager hooks)
- `@sudobility/shaperouter_types`
- `@sudobility/types`

The library has **no bundled runtime dependencies** — everything above is a peer
dependency.

## Usage

```typescript
import {
  useKeysStore,
  useKeysManager,
  ALL_TEMPLATES,
  applyTemplate,
  textClassifierTemplate,
} from '@sudobility/shaperouter_lib';

// Reactive read from a cache store
const keys = useKeysStore(state => state.getKeys(entitySlug));

// Manager hook: client fetch + Zustand cache fallback + CRUD
const { keys, isLoading, isCached, createKey } = useKeysManager({
  baseUrl,
  networkClient,
  entitySlug, // null disables fetching
  token,
});

// Apply a template to build a project + its endpoints
const { project, endpoints } = applyTemplate(
  textClassifierTemplate,
  'my-classifier', // project name
  'key-123'        // llm_key_id for endpoints
);
```

## What's exported

- **Stores** (Zustand): `useKeysStore`, `useProjectsStore`, `useEndpointsStore`,
  `useSettingsStore`, `useAnalyticsStore`, `useProviderModelsStore`
  (+ `DEFAULT_CACHE_MAX_AGE_MS`).
- **Manager hooks**: `useKeysManager`, `useProjectsManager`,
  `useEndpointsManager`, `useSettingsManager`, `useAnalyticsManager`,
  `useProviderModelsManager`.
- **Other hooks**: `useEndpointTester`, `useBudgetTracker` (+ `useBudgetStore`),
  `useProjectTemplates`, `useEndpointTemplates`.
- **Templates**: `ALL_TEMPLATES`, `applyTemplate`, and 9 individual templates
  (see below).
- **Utils**: `validateValue`, `validateInput`, `generateSampleValue`,
  `generateSampleInput`, `keyToSampleLabel`, `extractTestMetrics`.

Most stores cache in memory (lost on refresh). `useProviderModelsStore` persists
to `localStorage` (key `shaperouter-provider-models`, 1-hour TTL via `isStale`);
`useBudgetStore` persists to `localStorage` (key `shaperouter-budgets`).

## Templates

9 pre-built project templates for common AI use cases:

| Template | Category |
| -------- | -------- |
| `textClassifierTemplate` | Classification |
| `sentimentAnalyzerTemplate` | Analysis |
| `dataExtractorTemplate` | Extraction |
| `contentGeneratorTemplate` | Generation |
| `localizationTemplate` | Translation |
| `imageRecognitionTemplate` | Vision |
| `imageGenerationTemplate` | Generation |
| `imageProcessingTemplate` | Vision |
| `audioTranscriptionTemplate` | Audio |

## Development

```bash
bun install           # Install deps (use Bun, not npm/yarn/pnpm)

bun run build         # Build to dist/ (tsc + scripts/fix-esm-imports.mjs)
bun run build:watch   # tsc --watch
bun run clean         # Remove dist/

bun run test:unit     # Run Vitest once
bun run test:watch    # Run Vitest in watch mode

bun run typecheck     # tsc --noEmit
bun run lint          # ESLint over src/
bun run lint:fix      # ESLint --fix
bun run format        # Prettier --write src/
bun run format:check  # Prettier --check src/

bun run verify        # typecheck + lint + test:unit + build (run before pushing)
```

### Publishing

CI (`.github/workflows/ci-cd.yml`) auto-publishes to GitHub Packages on push to
`main`/`develop` via the shared `johnqh/workflows` reusable workflow. The
`prepublishOnly` script runs `clean` + `build`.

## Architecture

```
shaperouter_app (frontend)
    └── shaperouter_lib (this package — stores, managers, templates)
        └── shaperouter_client (API hooks)
            └── shaperouter_types (types)
```

Part of the ShapeRouter multi-project workspace; see `../CLAUDE.md` for the full
dependency graph and build order, and `CLAUDE.md` here for working-in-this-repo
notes.

## License

BUSL-1.1
