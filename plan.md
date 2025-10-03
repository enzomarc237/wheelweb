# MCP-Driven UI Inspiration Library (Tauri + TypeScript)

## Overview
- macOS-first desktop app using Tauri (Rust shell) + React/TypeScript UI.
- Local, app-managed image library with SQLite (WAL) for organization and fast queries.
- TypeScript MCP server sidecar exposes tools for AI agents to list/filter images and retrieve rich design specs JSON.
- Auto-extract: dimensions, aspect ratio, color palette; heuristic layout classification. Manual annotations for deeper specs. No OCR.

## Architecture
- UI: React + Vite + Tailwind in `apps/desktop/src`.
- Shell: Tauri (Rust) in `apps/desktop/src-tauri` for windowing, dialogs, file ops, spawning MCP, secure FS access.
- DB: SQLite file at app data dir (macOS: `~/Library/Application Support/WheelWeb/InspirationLibrary/library.db`) with WAL.
- Assets: Original images copied under `~/Library/Application Support/WheelWeb/InspirationLibrary/images/{yyyy}/{mm}/{hash}.{ext}`; thumbnails in `thumbnails/`.
- MCP server: TypeScript Node CLI in `packages/mcp-server` (stdio transport), launched/managed by Tauri; shares the same SQLite DB and reads files read-only.
- Shared types/schemas in `packages/shared`.

## Data Model (SQLite)
- `projects(id, name unique, description, created_at)`
- `categories(id, name unique)`
- `images(id, filename, file_hash unique, original_ext, file_size, width, height, aspect_ratio, project_id, added_by, imported_at, file_path, thumb_path)`
- `image_categories(image_id, category_id)`
- `tags(id, name unique)`
- `image_tags(image_id, tag_id)`
- `specs(image_id unique, json_specs TEXT, auto_palette JSON, auto_layout JSON, notes TEXT, updated_at)`
- Indices: on `images(project_id)`, `image_categories(category_id)`, `image_tags(tag_id)`, `tags(name)`, `categories(name)`.

## Metadata Pipeline
- Import:
  1) Copy file → library path; compute hash; dedupe by `file_hash`.
  2) Use `sharp` to read dimensions, create `thumb` (e.g., 512px max), compute aspect ratio.
  3) Use `node-vibrant` (or `colorthief`) to extract palette (dominant + population) → `auto_palette`.
  4) Heuristic layout guess: downsample to grid (e.g., 24x) and cluster to detect columns/hero/sidebar → store in `auto_layout`.
  5) Insert rows; attach to selected `project` and `categories`.
- Manual annotations in UI populate `specs.json_specs` (layout, components, color, typography, spacing, interaction, notes).

## Specs JSON (stored in `specs.json_specs`)
```json
{
  "$schema": "https://wheelweb.app/schemas/design-specs.schema.json",
  "layoutStructure": "hero+2-column|grid|single-column|sidebar-left|sidebar-right|custom",
  "components": ["navbar","hero","card","form","modal","sidebar","tabs","table"],
  "colorPalette": {"primary":"#2B6CB0","secondary":"#68D391","accent":"#ED8936","neutral":"#F7FAFC","background":"#FFFFFF","text":"#1A202C"},
  "typography": {"primaryFamily":"Inter","secondaryFamily":"Georgia","scale":"1.125","weights":[400,600,700]},
  "spacing": {"base":8,"scale":[8,12,16,24,32,48]},
  "grid": {"columns":12,"gutter":24,"container":1200},
  "iconography": {"style":"outline","set":"heroicons"},
  "effects": {"radius":12,"shadow":"md","blurs":false},
  "interactions": ["hover-card-elevate","focus-outline","cta-bounce"],
  "notes": "Concise rationale/usage notes."
}
```

## MCP Server (TypeScript)
- Transport: stdio per MCP. Process is a Node CLI started by Tauri and can also run standalone.
- Tools:
  - `list_images`:
    - Params: `{ project?: string; categories?: string[]; tags?: string[]; limit?: number; offset?: number }`
    - Returns: `{ images: ImageSummary[] }`
  - `get_image_details`:
    - Params: `{ ids?: string[]; project?: string; categories?: string[]; tags?: string[] }`
    - Returns: `{ images: ImageDetails[] }`
  - `search`:
    - Params: `{ query: string; limit?: number }` (full-text across names, tags, categories, notes)
    - Returns: `{ images: ImageSummary[] }`
- Types:
```ts
interface ImageSummary { id: string; filename: string; fileUri: string; thumbUri: string; project: string | null; categories: string[]; tags: string[]; width: number; height: number; aspectRatio: number; dominantColors: string[]; importedAt: string; }
interface ImageDetails extends ImageSummary { specs: any; autoPalette: { colors: { hex: string; population?: number }[] }; autoLayout: { kind: string; columns?: number; notes?: string }; }
```
- File URIs: `file://` absolute paths into the library; thumbs likewise. Ensure agent runtime has read access.

## UI (React)
- Screens:
  - Library Grid: filter by project/category/tag; infinite scroll; multi-select.
  - Import/Uploader: drag-and-drop + file dialog; choose project/categories; progress.
  - Image Detail: preview, auto metadata, editable specs form (schema-driven via `react-jsonschema-form`).
  - Projects & Categories Manager.
  - Settings: library path override, MCP server controls (auto-start, port/stdio), DB maintenance (reindex, backup).
- Components: `ImageCard`, `FilterBar`, `SpecsEditor`, `PaletteSwatches`, `LayoutBadge`.

## Desktop Integration (Tauri)
- Use `@tauri-apps/plugin-dialog` for file picker.
- Use `@tauri-apps/plugin-shell` to spawn MCP sidecar (stdio) and supervise lifecycle.
- FS ops via `@tauri-apps/api/fs` and Rust commands for high-throughput copies.
- SQLite via `tauri-plugin-sql` (or Rust `sqlx`) in the shell; MCP accesses DB via `better-sqlite3` in WAL mode (read-mostly).

## Project Structure
- `apps/desktop/` (React + Tauri)
  - `src/` UI code
  - `src-tauri/` Rust commands (import, thumbnails, DB ops, MCP launcher)
- `packages/mcp-server/` (TypeScript Node CLI implementing MCP)
- `packages/shared/` (Zod schemas, TypeScript types)
- `schemas/design-specs.schema.json`

## Build & Run (macOS)
- Prereqs: Node 20+, pnpm, Rust toolchain, Xcode CLT.
- Dev:
  - `pnpm install`
  - `pnpm dev` (Vite) + `pnpm tauri dev` (spawns MCP automatically)
  - MCP standalone: `pnpm --filter @wheelweb/mcp-server dev`
- Prod:
  - `pnpm build && pnpm tauri build` (bundles desktop app)
  - Bundle MCP: `pkg` or `nexe` to produce a self-contained Node binary; app places alongside resources.

## Security & Privacy
- Local-only processing; no outbound network by default.
- Sandboxed FS: library folder + read-only import paths.
- DB backups to user-selected location.

## Testing
- Unit: metadata extraction (palette, dimensions, layout heuristic), Zod schemas.
- Integration: DB migrations, import flow, MCP tools.
- E2E: UI filters, details editing, MCP responses against seeded fixtures.

## Acceptance Criteria
- Import images, assign to project/category, dedupe by hash.
- Browse/filter images with thumbnails and metadata.
- Edit and persist structured design specs.
- MCP tools return correct JSON with file URIs; filters work.
- App launches MCP sidecar automatically; sidecar robust to restarts.

## Key Files to Implement
- `apps/desktop/src-tauri/src/commands/import.rs` (copy+thumbs+db)
- `apps/desktop/src-tauri/src/mcp.rs` (spawn/supervise sidecar)
- `packages/mcp-server/src/index.ts` (MCP entry)
- `packages/mcp-server/src/tools/listImages.ts`, `getImageDetails.ts`, `search.ts`
- `packages/shared/src/types.ts`, `schemas.ts` (Zod)
- `schemas/design-specs.schema.json`
- `apps/desktop/src/screens/*` & components
