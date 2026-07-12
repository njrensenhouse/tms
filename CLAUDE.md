# TMS — Task Management System

A personal task manager with a horizontal, infinitely-scrolling timeline. Single-file web app, hosted on GitHub Pages, backed by Supabase, with two-way Todoist sync.

**Live:** https://njrensenhouse.github.io/tms/
**Owner:** Nathan (njrensenhouse)

## The whole app is one file

`index.html` contains everything — HTML, CSS (in `<style>`), and vanilla JS (in the final `<script>`). No build step, no framework, no npm. You edit `index.html` and that's the whole app.

- ~1600 lines. The `<script>` block is the app; a small `<style>` block holds all CSS.
- Rendering is a full re-render: `mount()` rebuilds `app.innerHTML` from state on every change. There is no virtual DOM. After `mount()`, event-wiring helpers like `initTimeline()`, `initChips()`, `initBars()` re-attach listeners.
- State lives in module-level globals: `AREAS` (object), `PROJECTS` (array), `TASKS` (array), `UNSCHED` (array). Plus UI flags (`searchOpen`, `addOpen`, `editOpen`, `schedOpen`, `menuOpen`, `editing`, `draft`, etc.).

### Verify before you push
There's no test suite. After editing, sanity-check the JS parses by extracting the inline `<script>` and running `node --check` on it. Broken JS silently blanks the page, so this matters.

## Data / date model

- **Day index model:** day index `0` == `2026-06-22` (`EPOCH = new Date(2026,5,22)`). `dateForIndex(i)`, `indexForDate(d)`, `ymd(i)` convert. `TODAY` is the index of the current day.
- **Timeline layout** is driven by a single CSS variable `--colw` (px per day column) on `.tl-inner`. Zooming just updates that variable — no DOM rebuild — which keeps pinch-zoom smooth. `COLW` (JS) mirrors it and is persisted to `localStorage` (`tms_colw`).
- **Window:** `win = {start, weeks}` is the rendered slice of the infinite timeline. It grows as you scroll (guarded against runaway growth by a cooldown + a 140-week cap).
- **Within-day order:** each task has an integer `ord`; `renumberDay()` / `moveTaskTo()` keep it consistent; preserved across Todoist syncs.
- Completed items are filtered OUT of the timeline (only shown in search). Todoist archives completed tasks, so they vanish from data on the next pull — a local `recentDone` list (localStorage `tms_recentDone`) keeps them visible in search.

## Backend

### Supabase (project ref `agicwlugswhimonyigbu`)
- Postgres tables: `areas`, `projects`, `tasks`, `unscheduled`. Open RLS policies (personal app).
- Frontend uses supabase-js v2 from CDN with the **publishable** key (safe to expose; it's in `index.html` on purpose).
- `loadData()` reads all tables; `saveArea/saveProject/saveTask` upsert single rows.
- Column notes: areas have `emoji`, `note`, `grades` (jsonb), cover fields; tasks have `description`, `recurring`, `rec_days` (jsonb), `ord`; projects have `description`, `recurring`.

### Todoist sync — Supabase Edge Function `todoist-sync`
- Deno function (`todoist-sync/index.ts`), `verify_jwt=false`, CORS `*`.
- Reads the secret `TODOIST_API_TOKEN` (a Supabase secret — **never put this token in the repo or in index.html**) plus auto-injected `SUPABASE_URL` / `SUPABASE_SERVICE_ROLE_KEY`.
- Uses Todoist API **v1** (`https://api.todoist.com/api/v1/`), paginated `{results, next_cursor}`.
- Actions: `pull` (Todoist → Supabase, non-destructive area sync, preserves `ord`, computes recurring occurrences), `create`, `update` (incl. area change via `/tasks/{id}/move`), `complete`, `reopen`, `delete`, `create_project`, `update_project`, `delete_project`.
- Excludes the Todoist projects **Inbox** and **Groceries** from import (`EXCLUDED_NAMES`).
- The frontend calls it via `sync(action, payload)` → `POST` to `SYNC_URL`.
- To deploy changes to it from Claude Code, use the Supabase CLI: `supabase functions deploy todoist-sync` (or the Supabase MCP if connected). It is NOT part of the GitHub Pages deploy.

## Frontend conventions & gotchas

- **Optimistic writes:** UI updates immediately; `sync(...)` runs in the background. A `pending` counter pauses the 30s background sync while writes are in flight. New items use a temp id, reconciled once Todoist returns the real id.
- **Undo/redo:** full local snapshots (`snapshot()`, `pushUndo()`, `restoreState()`), with `resyncStates()` diffing back to Todoist/Supabase.
- **Areas** map 1:1 to Todoist projects. Each area has a color (bright), plus a derived tint/ink, an emoji, and an optional cover image (Unsplash/link/upload, with crop).
- **Cards (task chips)** are neutral gray (`var(--card)`); the area's vibrant color (`--ac`) is used only for the checkbox border and the pencil edit icon. When zoomed out past ~3 weeks (`.compact`), chips shrink to solid `--ac` squares.
- **iPad / Apple Pencil:** `pencil-only` mode (localStorage `tms_pencilOnly`) reserves the Pencil for dragging cards. Cards are `touch-action:none` so the Pencil drags instantly; a custom one-finger pan in `initTimeline()` handles finger-scrolling over cards, and yields to 2-finger pinch-zoom. Other iPad prefs: `tms_tapUndo`, `tms_fourFinger`.
- **Drag:** `beginCardDrag()` shows a floating preview + a placeholder that drops the card into the exact slot where released, sliding other cards aside.
- **Access gate:** a basic client-side password gate (sha256 hash in `PASS_HASH`). Not a real security boundary — the Supabase key is public by design.
- **Theme/scale:** dark (default) / light via `data-theme`; UI scale via `document.documentElement.style.zoom`. Both saved per-device.

## Deploy workflow (this is the point of using Claude Code)

Hosting is GitHub Pages, "Deploy from a branch" (main / root). To ship a change:

1. Edit `index.html` (surgical diffs — don't rewrite the whole file).
2. Sanity-check JS parses (see "Verify before you push").
3. `git add -A && git commit -m "..." && git push`
4. GitHub Pages redeploys automatically in a minute or two. Hard-refresh to bypass CDN cache.

Keep changes as small diffs and commit often. Never commit the Todoist API token or any Supabase service-role key.
