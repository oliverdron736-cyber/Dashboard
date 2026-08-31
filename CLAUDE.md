# My Habit Tracker — Project Context

**Live URL:** https://oliverdron736-cyber.github.io/Habit-tracker/
**GitHub Repo:** https://github.com/oliverdron736-cyber/Habit-tracker
**Firebase Project ID:** habit-tracker-1fec3

This file exists so a fresh Claude Code session has the context of everything already built,
without needing it re-explained. The single most important fact: **`index.html` is the entire
app.** It's one self-contained HTML/CSS/JS file with no build step — everything lives in it.

## Current baseline

The `index.html` currently in this repo already includes everything below, through:
- The frosted-glass/blue visual redesign
- Exact Tabler-icon tab icons
- The bottom tab bar on mobile (top pill-row on desktop)
- Email/password authentication (migrated twice: sync-code → username/PIN concept → real email)

If a request sounds like it's asking to (re)build something in the list below, **check the
current file first** — it's very likely already done, and the ask is a refinement, not a
rebuild.

## Tech stack

- Single-file HTML/CSS/JS PWA, no build tools, no bundler
- Firebase Firestore (compat SDK v10.14.1 via CDN) for cross-device data sync
- Firebase Authentication (email/password) for login
- Firebase Storage (compat SDK v10.14.1 via CDN) for task photo attachments — only the
  `todoImages/{uid}/...` path is used; bytes live in Storage, only the download URL is stored in
  Firestore
- SortableJS v1.15.6 (CDN) for drag-and-drop reordering
- GitHub Pages for static hosting
- Deploy = upload `index.html` to repo root; Pages redeploys in ~30–90s

## Data model

Everything lives in one Firestore doc at `users/{uid}` (per authenticated user), mirrored to
localStorage. Fields: `habits`, `checkins`, `prefs`, `achievements`, `notes`, `folders`, `todos`,
`todoLists` — all included in the Settings > Backup & Restore JSON export (verified working).

- `habits`: `[{id, name, xp, active, scheduleType, days, anchorDate, repeatFrequency,
  trackMetric, metricUnit, logs:[{id,date,minutes,distance}], scheduleHistory:[{effectiveUntil,
  days}]}]`
  - `scheduleHistory` is important: when a habit's weekly `days` change, the *old* days get
    archived here with the date the change took effect. Streak/calendar logic checks this
    history so changing a habit's schedule never retroactively breaks past streaks. See
    `scheduleDaysForDate()` and `isScheduled()`.
- `checkins`: `{ "YYYY-MM-DD": { habitId: true } }`
- `notes`: `[{id, title, content (HTML, rich text), updatedAt, folderId, order}]`
- `folders`: `[{id, name}]` — notes must live in a folder, no "unfiled" concept
- `todos`: `[{id, listId, text, done, dueDate, dueTime, notes, subtasks:[{id,text,done}],
  images:[{id,url,path}], order, createdAt}]`
  - `images` holds Firebase Storage download URLs, not the image bytes themselves (see Tech
    stack). `path` is the Storage object path, kept so the file can be deleted when the image or
    the task itself is removed. Client-side compresses to a max 1600px-edge JPEG before upload.
- `todoLists`: `[{id, name}]`

## Authentication (current: real email/password)

Went through two iterations — worth knowing so nobody "fixes" it back to something older:
1. **Original:** a 48-char random sync code, used as the Firestore doc ID directly
   (`synccodes/{code}`), security enforced purely by code length (`>= 32` chars).
2. **Now:** real Firebase Auth (email/password), data at `users/{uid}`, secured by
   `request.auth.uid == userId`. Includes a working "Forgot password?" flow
   (`sendPasswordResetEmail`) that deliberately shows the same message whether or not the email
   exists (no account enumeration).

**Firestore security rules currently required (already set up, but here for reference):**
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /synccodes/{code} {
      allow get, create, update: if code.size() >= 32;
      allow list, delete: if false;
    }
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```
The `synccodes` block is only kept so `migrateOldSyncCodeDataIfPresent()` can pull in data from
anyone who still has an old sync code saved locally. Safe to remove once confident nobody needs
it.

**Firebase Console setup required** (not in code, can't be verified by reading the file):
Authentication > Sign-in method > Email/Password must be enabled.

**Storage security rules required for task photos** (Firebase Console > Storage > Rules — Storage
must also be enabled on the project if it isn't already):
```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /todoImages/{userId}/{allPaths=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```
Without this, uploads/deletes from the To-Do task modal's Photos section will fail with a
permission error even though the client code is correct.

## Feature map (roughly chronological)

- **Today tab:** checklist, completion ring (now in the header, top-right, enlarged), streak
  badge (flame icon, under the title), collapsible "weekly & monthly stats," collapsible
  "Manage habits" (moved here from Settings — lazy-renders only when expanded, see gotcha below)
- **Monthly tab:** calendar grid, day-detail panel, color-coded completion (green/amber/red are
  meaningful status colors — not the theme's accent color, deliberately untouched by the
  redesign)
- **Notes tab:** folders (Level 1) > notes-in-folder (Level 2) > editor (Level 3). Rich text via
  a `contenteditable` div (bold/italic/bullets/numbers), a fixed bottom formatting toolbar with
  live active-state highlighting, paste sanitization (preserves structure from apps like Apple
  Notes, strips clashing fonts/colors), and a backspace fix for exiting a list cleanly. "⋮" menu
  on both folder cards and note rows (Rename/Delete/Move — replaced old inline text links and
  cross-folder drag targets). Drag-to-reorder within a folder still works.
- **To-Do tab:** lists (Level 1) > tasks-in-list (Level 2). Encircled "+" opens a task detail
  modal (create AND edit use the same modal; an empty task is auto-discarded on close *unless* it
  has photos attached — see gotcha below). Modal has due date, due time, an "Add subtask" field,
  a notes textarea, a "Photos" section (upload/remove, see Data model + Tech stack for how
  images are stored), and "Move to list…" (reassigns `listId`, mirrors notes' "Move to folder…").
  Inline "▸" dropdown on each task shows existing subtasks for viewing/checking off *only* when
  subtasks exist — adding one never auto-expands it. Drag-to-reorder works on the active task
  list.
- **Settings tab:** Account (shows logged-in email, Log out), Backup & Restore (full JSON export
  covering every field above).
- **Visual redesign:** frosted-glass theme (blur + translucent panels), black background, blue
  accent (`#3fa3ff`), unified sans-serif typography (the CSS variables `--font-serif` and
  `--font-mono` were both repointed to the same system-sans stack — a single-point change that
  cascaded everywhere rather than editing every element). Tab icons are exact Tabler SVG paths
  (fetched from Tabler's own site, not approximated). Checkboxes are outline-circle /
  circle-check SVGs, not the old filled squares. Bottom tab bar (icon-over-label, fixed,
  blurred) on mobile only (`max-width: 820px`); desktop keeps a top pill-row.

## Known gotchas (things that already caused real bugs — worth reading before touching related code)

1. **Don't reactively re-render "Manage habits" from the Firestore sync listener.** An earlier
   version did this to keep it "live," but local saves echo back through the same `onSnapshot`
   listener as remote changes, so it was wiping whatever the user was mid-typing into the
   "new habit" field on *every* unrelated action (even a checkbox toggle elsewhere). Current
   fix: it only re-renders when the dropdown is actually opened, plus after direct
   add/edit/remove actions on habits themselves.
2. **Font/icon glyphs don't always render centered in their box** — "+", "⋮", and a hand-drawn
   flame icon all had this problem (looked visually off-center or, in the flame's case,
   completely garbled) despite the *container* being correctly sized/centered. Prefer precise
   SVG icons over text glyphs or hand-drawn paths; when a real icon set is being matched (e.g.
   Tabler), fetch the actual path data rather than approximating from memory.
3. **`scheduleHistory` exists specifically so changing a habit's weekly days doesn't corrupt
   past streaks.** If asked to "let me change which days a habit runs," check whether this
   mechanism already covers it before adding new logic.
4. **The empty-task auto-discard in `closeTodoTaskModal()` checks `t.images.length` too, not
   just `t.text`.** Without that, a task with only photos and no title would silently vanish
   (photos and all) the moment its modal closed. If new task fields get added later, check
   whether they need the same guard.

## Deployment

Upload `index.html` to the repo root on GitHub (manually, or however Claude Code's GitHub
integration is set up in this session) — GitHub Pages redeploys automatically. Bump the
`CACHE_NAME` version in `service-worker.js` if a change needs to force-bypass the PWA's offline
cache on users' devices.
