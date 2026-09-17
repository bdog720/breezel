# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Breezel: a React 19 + TypeScript + Vite app (run with Bun) — a drag-and-drop editor for generating App Store / Play Store screenshots with realistic device frames. In the Docker image a small dependency-free Bun server (`server/`) serves the app and a storage API; projects save to the container's `/data` volume. Without that API (`bun run dev`, static hosting) projects save to `localStorage`.

> The repo-root `AGENTS.md` is stale boilerplate from an unrelated JWT/auth template (TanStack Query/Form, an API client, protected routes). None of that exists here — ignore it.

## Commands

```bash
bun install                                         # install deps
bun run dev                                          # dev server → http://localhost:5173
bun run dev:server                                   # storage API on :3000 (Vite proxies /api to it)
bun run build                                        # vite build THEN tsc — build fails on any type error
bun run test                                         # full Vitest suite, once
bunx vitest run src/lib/device-overflow.test.ts      # single file
bunx vitest run -t "punch-hole"                       # by test name
bunx tsc --noEmit                                     # type-check only
```

Tests are Vitest + jsdom + Testing Library, colocated as `*.test.ts(x)`.

## Architecture

**State model:** `Project → Screenshot[] → DeviceInstance[]` (types in `src/types/index.ts`), owned by `src/context/EditorContext.tsx` — the single source of truth. Persistence goes through a `ProjectStorage` (`src/lib/storage/`) chosen at startup — see **Storage** below. A `DeviceInstance` references a `DeviceSpec` by `deviceId` and carries its own image, color, transform, and 3D angles, so one screenshot holds multiple independently-styled devices.

**Two rendering pipelines that must stay pixel-identical** — this is the load-bearing fact of the codebase:

| Pipeline | Where | Purpose |
|----------|-------|---------|
| Live preview (DOM/CSS) | `src/components/DeviceFrame/` (`DeviceFrame.tsx` flat, `DeviceFrame3D.tsx`, `CameraElements.tsx`, `DeviceButtons.tsx`) | On-canvas editing |
| Export (Canvas 2D → PNG/ZIP) | `src/lib/export-utils.ts` | Downloaded screenshots |

`export-utils.ts` re-implements the frame, screen, camera cutout, buttons, and 3D perspective projection with raw canvas calls. **Any change to how a device looks must be made in both pipelines or the export diverges from the preview.** `EXPORT_EDGE_DEPTH` (export-utils) must equal `EDGE_DEPTH` (DeviceFrame3D). The same dual-implementation rule applies to rich text: `RichTextEditor/` (DOM) ↔ `src/lib/rich-text-canvas.ts` (canvas).

**Device data is declarative.** `src/constants.ts` holds `devices: DeviceSpec[]`, `gradientPresets`, and `exportSizes`. The picker (`LeftSidebar/DeviceSection.tsx`) just maps the array, so adding a device is mostly data.

**Camera/button style is inferred, not stored:**
- `src/lib/device-platform.ts` → `isAndroidDevice(id)` (`samsung-` / `pixel-` prefixes) and `isAndroidTablet(id)`. Android ⇒ punch-hole camera + right-side buttons; Apple ⇒ Dynamic Island/notch + iPhone buttons.
- `hasIsland` ⇒ Dynamic Island; else `notchWidth > 0` ⇒ notch; else no top cutout.

**Cross-screen overflow:** `src/lib/device-overflow.ts` computes a device dragged past a screenshot edge continuing into the neighbor; both preview and export render via `getRenderableDevicesForScreenshot`, so overflow survives export. `src/lib/device-instances.ts` normalizes legacy/older persisted shapes — keep it backward-compatible; real users have projects in container storage or `localStorage` (see **Storage** below).

## Storage

- `src/routes/index.tsx` → `bootstrapEditor` picks storage via `GET /api/health` (`resolveStorage`): `ServerStorage` when the container reports writable storage, otherwise `BrowserStorage`. First run against an empty container migrates browser projects (`migrate.ts`).
- `EditorProvider` receives `storage` + `initialState`; `useProjectPersistence` autosaves (1 s debounce), exposes save status / Save now / conflict actions. Don't write to `localStorage` or `fetch` projects directly — go through `ProjectStorage`.
- Server storage saves images as `/api/images/<sha256>.<ext>` URLs (uploaded from data URLs on save); `Export Project` inlines them again. Project shapes are otherwise identical in both modes and still pass through `normalizeProject` on load.
- `server/`: `validation.ts`, `history.ts` (pure rules), `store.ts` (`FileStore`, atomic writes, revisions, history, image GC), `api.ts` (`handleRequest`, tested with Vitest), `main.ts` (`Bun.serve` only). Server code uses Web APIs + `node:` built-ins, no npm deps; it's type-checked by `tsc -p server`.

## Adding a device (the common task)

1. Append a `DeviceSpec` to `devices` in `src/constants.ts`. The `id` prefix drives styling: `iphone-*` / `ipad-*` (Apple), `samsung-*` / `pixel-*` (Android), and any id containing `tab` is treated as a tablet.
2. Set `hasIsland` / `notchWidth` for the cutout. Use `width`/`height` at true pixel resolution — the frame is rendered from that aspect ratio.
3. Adding a **new Android brand** (not Samsung/Pixel) means broadening `ANDROID_PREFIXES` in `src/lib/device-platform.ts` — nothing else, thanks to the shared helper.
4. Optionally add an App Store submission size to `exportSizes`.
5. No component edits are needed for a normal iPhone/iPad/Galaxy/Pixel — data only.
6. Run `bun run gen:agent-docs` so the agent import prompt lists the new device (`prompt.test.ts` fails otherwise).

## Agent import

`src/lib/agent-import/` turns an agent-written `breezel.json` + screenshots into an ordinary `Project`: `bundle.ts` (files/folder/zip) → `schema.ts` (Zod) → `compile.ts`, orchestrated by `pipeline.ts` and surfaced by `components/AgentImport/AgentImportModal.tsx` (Project menu → Import from agent).

- `schema.ts` is the single source of truth for manifest types, validation errors and the published JSON Schema. Keep it free of `.transform()` (`z.toJSONSchema` can't represent transforms); normalize in `compile.ts`.
- `docs/agent-import/PROMPT.md`, `breezel-import.schema.json` and `example/breezel.json` are **generated**. After changing the schema, devices, fonts, brand styles, layout presets or export sizes, run `bun run gen:agent-docs`; `prompt.test.ts` fails on stale docs.
- Imported headline/subheadline HTML is untrusted — it must go through `sanitize.ts`.
- Layout presets live in `src/lib/layout-presets.ts`, shared with the editor's Position Presets panel (which applies only the device part).

## Internal docs

`internal-docs/` is gitignored. Put short-term internal Markdown and artifact files there: plans, specs, research notes, audits and handoff notes for work that spans several chat sessions.

- Write new plans, specs and research here, including files a skill would otherwise put in `docs/` (for example superpowers plans and specs). The existing `docs/superpowers/` files stay where they are.
- Start each file with its date and status, so a later session can tell whether it is still current.
- These files exist only on this machine. Anything that must ship with the repo (user docs, generated agent-import docs) still belongs in `docs/`.
- Check `internal-docs/` at the start of a session when the task continues earlier work.

## Constraints & gotchas

- ⚠️ The app was called AppShots before it became Breezel. Keep reading the legacy names: `APPSHOTS_*` env vars (`server/config.ts`), `appshots-project` backups (`project-io.ts`), `appshots.json` / `appshots-import` manifests (`bundle.ts`, `schema.ts`), and the `appshots-*` localStorage keys (`migrate.ts`, `onboarding.ts`), which stay unrenamed on purpose.

- ❌ Don't change a device's visuals in only one pipeline — preview and `export-utils.ts` must match.
- ❌ Don't break `device-instances.ts` normalization or bump persisted shapes without a migration — it silently corrupts saved user projects.
- ✅ Do run `bun run build` before claiming done; `tsc` is part of the build and gates type errors that `vitest` alone won't catch.
- ✅ Do keep camera/button logic flowing through `device-platform.ts` rather than re-adding inline `id.startsWith(...)` checks.

## Testing & workflow

- TDD is expected: write the failing test first (see `src/lib/*.test.ts`), watch it fail, then implement. Pure logic (platform detection, overflow math, device normalization, rich-text) is unit-tested; visual frame output is verified by running the app.
- Routing is file-based TanStack Router in `src/routes/`. Fonts load on demand via `src/lib/google-fonts.ts`; export awaits `document.fonts.ready`.
