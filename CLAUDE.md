# CLAUDE.md

Guidance for AI assistants working in the Remotion monorepo.

> For the full dev-environment setup and cloud-agent caveats, also read [AGENTS.md](AGENTS.md).

---

## Project overview

Remotion is a TypeScript/React framework for creating videos programmatically. Compositions are written as React components; they are rendered to video by headless Chromium + a Rust compositor + ffmpeg. The current version is **4.0.474** (see `packages/core/src/version.ts`).

---

## Monorepo layout

```
remotion/
├── packages/              # 124 packages — everything lives here
├── .github/               # CI workflows
├── .githooks/             # Pre-commit hook (runs bun pre-commit.ts)
├── .claude/               # Claude Code project settings
├── .agents/               # Agent-specific docs
├── turbo.json             # Turbo build pipeline
├── package.json           # Root workspace & scripts
├── tsconfig.json          # Root TS config with 72 project references
├── bunfig.toml            # Bun runtime config
├── go.work                # Go workspace (lambda-go SDK)
├── pre-commit.ts          # Formatting hook script
└── publish.ts             # Release script
```

### Key package groups

| Group | Packages | Purpose |
|---|---|---|
| **Core** | `remotion`, `@remotion/cli`, `@remotion/bundler`, `@remotion/renderer` | Video composition framework, CLI, Webpack bundler, Node.js rendering engine |
| **Player / Studio** | `@remotion/player`, `@remotion/studio`, `@remotion/studio-server`, `@remotion/studio-shared` | Embeddable React player; interactive Studio UI + backend |
| **Cloud rendering** | `@remotion/lambda`, `@remotion/lambda-client`, `@remotion/serverless`, `@remotion/cloudrun`, `@remotion/vercel` | AWS Lambda, GCP Cloud Run, Vercel Sandbox backends |
| **Browser / WebCodecs** | `@remotion/web-renderer`, `@remotion/webcodecs`, `@remotion/media-parser`, `@remotion/media` | In-browser rendering and media handling |
| **Visual components** | `@remotion/shapes`, `@remotion/paths`, `@remotion/transitions`, `@remotion/effects`, `@remotion/three`, `@remotion/skia` | SVG, Three.js, Skia integrations, transition effects |
| **Audio / Captions** | `@remotion/sfx`, `@remotion/captions`, `@remotion/elevenlabs`, `@remotion/openai-whisper`, `@remotion/whisper-web` | Sound effects, captions, speech-to-text |
| **Fonts / Styling** | `@remotion/fonts`, `@remotion/google-fonts`, `@remotion/tailwind`, `@remotion/tailwind-v4`, `@remotion/enable-scss` | Typography and CSS tooling |
| **Tooling** | `@remotion/eslint-plugin`, `@remotion/eslint-config`, `@remotion/zod-types`, `@remotion/licensing` | Linting, validation, licensing |
| **Compositor** | `@remotion/compositor` + OS variants | Rust binary for video composition (darwin, linux, win32) |
| **Templates** | `template-*` (30+ templates) | Starter projects for `create-video` |
| **Examples / Tests** | `@remotion/example`, `@remotion/it-tests`, `@remotion/react18-tests` | Dev testbed and integration test suites |
| **Docs** | `packages/docs` | Docusaurus documentation site |

---

## Tech stack

| Concern | Tool |
|---|---|
| Language | TypeScript (strict, ES2018 target), TSX, some Go (Lambda SDK), Rust (compositor) |
| Package manager | **Bun 1.3.3** — use `bun`/`bunx`, never `npm`/`npx` |
| Monorepo orchestration | **Turbo 2.9.14** — caches task outputs in `dist/` and `build/` |
| Bundling | Webpack (via `@remotion/bundler`) for compositions; Bun for packages |
| React | React **19.2.3** |
| Testing | **Bun test** (primary), **Vitest 4.0.9**, **Playwright 1.55.1**, **@testing-library/react 16.1.0** |
| Linting | **ESLint 9.19.0** |
| Formatting | **Prettier 3.8.1** + **oxfmt 0.35.0** (Rust formatter) |
| CI | GitHub Actions (`.github/workflows/`) |

---

## Setup commands

```bash
bun install                        # Install all workspace deps
bunx turbo run make                # Build every package (alias: bun run build)
bunx turbo run lint test           # Lint + test everything
bun run clean                      # Remove dist/ artifacts

# Build / watch a single package
bunx turbo run make --filter='@remotion/player'
bunx turbo watch make --filter='@remotion/player'
```

Run `bun run build` before tests or Studio — many packages depend on each other's compiled output.

---

## Development services

| Service | Command | URL |
|---|---|---|
| Remotion Studio (main dev UI) | `cd packages/example && bun run dev` | `http://localhost:3000` |
| Player testbed | `cd packages/player-example && bun run dev` | varies |
| Docs site | `cd packages/docs && bun run start` | `http://localhost:3001` |

### Rendering from CLI (inside `packages/example`):

```bash
bunx remotion compositions                                # list compositions
bunx remotion render <comp-id> --output ../../out/video.mp4
bunx remotion still  <comp-id> --output ../../out/still.png
```

---

## Before committing

1. `bun run build` — verify all packages compile
2. `bun run stylecheck` — runs `turbo run lint formatting` (ESLint + Prettier/oxfmt)
3. Commit `bun.lock` whenever dependencies change

The pre-commit hook (`pre-commit.ts`) runs formatting automatically. Never skip it with `--no-verify`.

---

## Testing

```bash
bun run test                   # lint + all unit tests via Turbo
bun test                       # run tests in the current package directory

# Specialized suites (run from repo root)
bun run testssr                # SSR rendering tests
bun run teste2e                # Playwright end-to-end tests
bun run testwebcodecs          # WebCodecs browser tests
bun run testwebrenderer        # In-browser renderer tests (concurrency=1)
bun run testlambda             # AWS Lambda rendering tests
bun run testlayoututils        # Layout utils visual tests
bun run testeffectsvisual      # Effects visual tests
```

- Vitest is used for packages that need its advanced features; others use Bun's built-in runner.
- `@remotion/openai-whisper` tests need `OPENAI_API_KEY` — one failure without it is expected.
- `@remotion/lambda-go` lint requires Go ≥ 1.23.0; the VM ships 1.22.2, so that lint target is optional.

---

## Code conventions

### TypeScript
- Strict mode everywhere. No `any` unless truly necessary.
- Target: ES2018. Module: CommonJS for packages; ESM for browser targets.
- Project references are declared in `tsconfig.json` at the root.
- Each package has its own `tsconfig.json` that inherits from root.

### Package scripts (`make` / `test` / `lint`)
Every package follows the same script contract:

| Script | What it does |
|---|---|
| `make` | Compile TS (`tsgo -d`) + optional bundle step (`bundle.ts`) |
| `test` | `bun test src/` or vitest |
| `lint` | ESLint on `src/` |
| `formatting` | oxfmt check |
| `format` | oxfmt fix |

### Versioning
All `@remotion/*` packages share the same version. Bump only `packages/core/src/version.ts` — the publish script propagates it. The next version increments the **patch** number.

### PR titles
Format: `` `@remotion/<package>`: Short imperative description. ``  
Example: `` `@remotion/player`: Add fullscreen API. ``

### Comments
Write comments only when the _why_ is non-obvious. Never describe what the code does — well-named identifiers do that. No multi-line comment blocks.

### No backwards-compat hacks
Don't add deprecated exports, `_unused` renames, or `// removed` comments. Delete dead code completely.

---

## Turbo pipeline key rules

- `make` depends on `^make` — upstream packages must be built first.
- `test` depends on `^make` — tests always run against compiled output.
- `@remotion/example#bundle` is a special gate: many test suites (`testssr`, `testlambda`, `testtemplates`) depend on it.
- `build-docs` additionally requires `@remotion/convert#build-spa`, `@remotion/brand#bundle`, and the example testbed bundle.

---

## Dependency catalog (pinned versions in root `package.json`)

Key pinned versions to keep consistent across packages:

| Package | Version |
|---|---|
| react / react-dom | 19.2.3 |
| typescript | 5.9.3 |
| vitest | 4.0.9 |
| playwright | 1.55.1 |
| @types/node | 20.12.14 |
| zod | 4.3.6 |
| next | 16.2.6 |
| three | 0.178.0 |
| sharp | 0.34.5 |
| openai | 4.67.1 |
| eslint | 9.19.0 |
| prettier | 3.8.1 |

When adding a new dependency that matches one of these, use the catalog version.

---

## Adding things

| Task | Skill / Reference |
|---|---|
| New CLI option | Use `/add-cli-option` skill |
| New package | Use `/add-new-package` skill |
| New sound effect | Use `/add-sfx` skill |
| New expert on docs | Use `/add-expert` skill |
| New bug entry | Use `/add-bug` skill |
| New docs demo | Use `/docs-demo` skill |
| Contribution guide | `packages/docs/docs/contributing/` |

---

## Known caveats

- Studio sometimes reports "Already running on port 3000" if a previous instance is still alive. Check with `curl http://localhost:3000`.
- After `bun install`, always `bun run build` before running tests.
- `@remotion/compositor` ships platform-specific Rust binaries; the correct variant is resolved automatically by the package manager on install.
- The `go.work` file covers `@remotion/lambda-go` only. No Go tooling is needed for core development.
