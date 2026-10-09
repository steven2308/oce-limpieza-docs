# Template Rename Implementation Plan

> **For agentic workers:** Execute task-by-task. Steps use checkbox syntax.

**Goal:** Replace template identity (`limpieza` / `Limpieza`) with `cleaning` / `Castivas Cleaning`, apply Fresh Teal, and initialize backend `.envs` from examples.

**Architecture:** Mechanical rename of Python package + Docker/env branding in backend; display/config branding + theme in frontend. No feature changes.

**Tech Stack:** Django/DRF backend, Vue 3 + Vuetify frontend, Docker Compose env files.

## Global Constraints

- Slug: `cleaning` (packages, images, DB, package.json)
- Display: `Castivas Cleaning`
- Theme: `#0D9488` / `#14B8A6` / `#DC2626`
- Do not rename parent folder or `docs/plan/` Spanish content
- Do not commit unless asked

---

### Task 1: Backend package + string rename

**Files:** `back/limpieza/` → `back/cleaning/`; all imports; compose yml; settings; templates; docs; AGENTS/README; `.editorconfig`; `.envs.example`

- [x] Rename package directory
- [x] Replace `limpieza` → `cleaning` and `Limpieza` → `Castivas Cleaning` under `back/`
- [x] Update `.envs.example` postgres/django URLs and credentials

### Task 2: Initialize backend `.envs`

- [x] Copy `.envs.example/` → `.envs/` (local + production)
- [x] Confirm values use `cleaning`

### Task 3: Frontend branding + theme

- [x] `app.ts`, `index.html`, `package.json` / lock, compose, Makefile, README
- [x] `useAuthLocale.ts` storage key
- [x] `vuetify.ts` Fresh Teal
- [x] Note icon TODO in README

### Task 4: Verify

- [x] Grep for leftover template names under back/front
- [x] Spot-check critical files
