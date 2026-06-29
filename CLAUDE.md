# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

`tots-container-lib-angular` is a **meta-repository**: it has no source code, no `package.json`, and no build tooling of its own. It exists only to aggregate the `@tots/*` Angular libraries as **git submodules** so they can be cloned and worked on together. Each submodule is an independent, self-contained Angular library workspace published to npm under the `@tots/*` scope.

All real work happens *inside* a submodule. There is no root-level build, install, or test.

```bash
git submodule update --init --recursive   # populate submodules after clone
git submodule status                       # see each submodule's pinned commit/branch
```

## Submodules and their npm packages

Each directory is its own Angular CLI workspace (`angular.json`, `package.json`, `package-lock.json`, `node_modules`). A single submodule often contains **multiple** publishable library projects under `projects/tots/<name>/`.

| Submodule dir | Primary project | Other projects in the workspace |
|---|---|---|
| `tots-core` | `@tots/core` | — |
| `tots-loading` | `@tots/loading` | — |
| `tots-cloud-storage` | `@tots/cloud-storage` | — |
| `tots-not-found` | `@tots/not-found` | — |
| `tots-auth` | `@tots/auth` | `@tots/auth-layout` |
| `tots-table` | `@tots/table` | `@tots/date-column`, `@tots/editable-columns` |
| `tots-form` | `@tots/form` | `form-api`, `form-sidebar-page`, `date-field-form`, `users-selector-menu`, `day-selector-menu`, `range-date-selector-menu`, `html-field-form`, `quill-mention-field-form`, `monaco-editor-field-form`, `form-wizard` |
| `tots-layout` | `@tots/layout` | `@tots/confirm-modal`, `@tots/crud-page` |
| `tots-notification` | `@tots/notification` | `@tots/notification-core` |
| `tots-account` | `@tots/account-core` | — |
| `tots-task` | `@tots/task` | — |
| `tots-templates` | `@tots/templates-core` | `@tots/templates-editor` |

## Dependency graph (build foundation → consumers)

`@tots/core` is the foundation; everything else builds on top. When changing a low-level package, expect downstream packages to be affected.

- **No `@tots` deps:** `core`, `loading`, `cloud-storage`, `not-found`
- `form` → `core`, `loading`
- `table` → `core`
- `auth` → `core`
- `account` → `auth`, `core`
- `task` → `core`, `table`, `date-column`
- `templates` → `core`, `ngx-monaco-editor-v2`
- `notification` → `loading`
- `layout` → `core`, `cloud-storage`, `form`, `loading`, `table`, `filter-box`, `filter-menu`, `users-selector-menu` (top of the stack — most dependent)

**Cross-submodule deps** (e.g. `form` → `loading`) resolve from `node_modules` (the published package). **Within a single submodule**, sibling projects are consumed from build output: `tsconfig.json` maps `@tots/<name>` → `dist/tots/<name>`, so a dependent project must be built first.

## ⚠️ Angular versions are NOT uniform across submodules

Submodules are pinned to different branches and different Angular major versions. Always check the submodule's own `package.json` before assuming an API/syntax level.

- **Angular 20.3** (branch `v20`): `core`, `auth`, `cloud-storage`, `form`, `layout`, `loading`, `not-found`, `table`
- **Angular 15.1** (branch `main`/other): `account`, `task`, `templates`, `notification`

## Common commands (run inside a submodule, never at root)

```bash
cd tots-core            # pick the submodule you're working in
npm install             # each submodule installs independently

npm run build           # ng build — builds the workspace default project
ng build --project=@tots/form --configuration production   # build one specific project
npm run watch           # ng build --watch --configuration development
npm test                # ng test — Karma + Jasmine (launches Chrome)
```

### Running a single test
Karma runs all `*.spec.ts` by default. To focus, use Jasmine's `fdescribe` / `fit` in the spec, or scope by project:
```bash
ng test --project=@tots/core
```

### Publishing
The `package.json` of each submodule defines numbered `buildN` scripts (`build2`, `build3`, …) — one per publishable project. Each builds in production mode then `npm publish --access public` from its `dist/tots/<name>/` output. Example (`tots-form`):
```bash
npm run build2   # @tots/form
npm run build3   # @tots/form-api
# ... build4..build12 for the remaining form projects
```

## Library conventions

- **Project layout:** every library lives at `projects/tots/<name>/` with a `src/public-api.ts` barrel (the package entry point — anything not re-exported here is not part of the public API) and `src/lib/` organized by kind: `entities/`, `services/`, `helpers/`, `operators/`, plus the NgModule.
- **Component selector prefix:** `lib`.
- **TypeScript is strict:** `strict`, `noImplicitOverride`, `noPropertyAccessFromIndexSignature`, `strictTemplates`, `strictInjectionParameters` are all on. New code must satisfy these.

### `@tots/core` is the data-access backbone
- `TotsBaseHttpService<T>` — generic CRUD base class (`create`, `fetchById`, `update`, `list`, `removeById`, `updateInBulk`, `removeInBulk`). Subclass it and set `basePathUrl`; URLs are built from injected config + `basePathUrl`.
- Config is injected via the `TOTS_CORE_PROVIDER` token, typed as `TotsCoreConfig` (provides `baseUrl`, `lang`). Consuming apps subclass `TotsCoreConfig`.
- `TotsQuery` builds paginated/filtered list requests (`page`, `per_page`, `wheres`, `orders`, `withs`, …) and serializes to a query string; `TotsListResponse<T>` is the paginated response envelope. Use these for any list endpoint rather than hand-rolling params.
