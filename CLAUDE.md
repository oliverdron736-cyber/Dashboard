# My Habit Tracker — Project Context

**Live URL:** https://oliverdron736-cyber.github.io/Dashboard/
**GitHub Repo:** https://github.com/oliverdron736-cyber/Dashboard
**Firebase Project ID:** habit-tracker-1fec3

The repo was renamed from `Habit-tracker` to `Dashboard` — GitHub redirects the old name for
git operations, but don't rely on that indefinitely. If any of this session's tooling still
references the old name, that's expected to keep working via the redirect; a fresh session should
use `Dashboard` directly.

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
- Firebase Storage (compat SDK v10.14.1 via CDN) for photo attachments — `todoImages/{uid}/...`
  for task photos, `noteImages/{uid}/{noteId}/...` for photos embedded in notes; bytes live in
  Storage, only the download URL is stored in Firestore
- SortableJS v1.15.6 (CDN) for drag-and-drop reordering
- GitHub Pages for static hosting
- Deploy = upload `index.html` to repo root; Pages redeploys in ~30–90s

## Data model

Everything lives in one Firestore doc at `users/{uid}` (per authenticated user), mirrored to
localStorage. Fields: `habits`, `checkins`, `prefs`, `achievements`, `notes`, `folders`, `todos`,
`todoLists` — all included in the Settings > Backup & Restore JSON export (verified working).

- `habits`: `[{id, name, xp, active, scheduleType, days, anchorDate, repeatFrequency,
  trackMetric, metricUnit, logs:[{id,date,minutes,distance}], scheduleHistory:[{effectiveUntil,
  days}], createdAt}]`
  - `scheduleHistory` is important: when a habit's weekly `days` change, the *old* days get
    archived here with the date the change took effect. Streak/calendar logic checks this
    history so changing a habit's schedule never retroactively breaks past streaks. See
    `scheduleDaysForDate()` and `isScheduled()`.
  - `createdAt` (a `YYYY-MM-DD` string, same format/comparability as `anchorDate` and
    `scheduleHistory[].effectiveUntil`) stamps the day a habit was added, set once in the
    `addHabitBtn` click handler. It is deliberately **not** user-editable — there's no UI for it
    anywhere; it's recorded at creation and left alone. `isScheduled()` treats any date before it as "not scheduled,"
    which is what stops a brand-new habit from retroactively counting as scheduled-but-missed on
    every past day back to `TRACKING_START_DATE`, dragging down the Monthly view's completion %
    for days it didn't exist yet. Habits created before this field existed have no `createdAt` at
    all — `isScheduled()` only applies the cutoff `if(habit.createdAt && ...)`, so old habits keep
    their full existing history untouched. `TRACKING_START_DATE` is a separate, global,
    non-per-habit cutoff ("no real data before this date at all") — the two checks are
    independent and both apply.
- `checkins`: `{ "YYYY-MM-DD": { habitId: true } }`
- `notes`: `[{id, title, content (HTML, rich text), updatedAt, folderId, order}]`
  - Photos inside a note live as plain `<img src="...">` tags in `content` pointing at
    `noteImages/{uid}/{noteId}/...` Storage download URLs — unlike `todos.images`, there's no
    separate structured array, since a note's photos are just part of its free-form HTML. See the
    gotcha below for how their Storage cleanup works without one.
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

`init()` explicitly calls `firebase.auth().setPersistence(Auth.Persistence.LOCAL)` before
attaching the auth-state listener, rather than relying on the SDK's default fallback chain — this
is what keeps a session across closing/reopening the browser or app. It's wrapped in try/catch so
a rejection (rare, but possible if IndexedDB is genuinely unavailable) can't block app init. Note
this only controls what the app itself requests; the browser/OS can still clear storage
independently (Safari's ITP caps un-visited "web content" storage at ~7 days — an installed
home-screen PWA, which the manifest already supports via `display:standalone`, gets a separate,
more durable storage container than a regular Safari tab).

The login form (`#loginForm`, wrapping `loginEmail`/`loginPassword`/`loginBtn`) is a real
`<form>` with a `type="submit"` button — not just a button with a click handler — for two
reasons: it lets Enter/"Go" on the keyboard submit, and it makes Safari more willing to offer
AutoFill/Face ID/Touch ID suggestions for it at all (bare buttons are treated less reliably as
"real" login forms). On top of that, `loginEmail`/`loginPassword` have a CSS
`:-webkit-autofill` + imperceptible-animation trick (`onNoteAuthAutofill` in the `<style>` block)
that fires an `animationstart` event specifically when the browser genuinely autofills a field —
including a Face ID/Touch ID-unlocked Keychain credential — and never on manual typing. When that
fires and *both* fields already hold a value, the form submits itself automatically, so unlocking
with Face ID logs you in without an extra tap on "Log In". This is the standard cross-browser way
to detect autofill, since there's no native "autofilled" DOM event. Only the login form got this
treatment (not signup) since that's the flow this exists for.

That autofill hook is useless if the form isn't on screen when Safari sizes up the page, which is
why `init()` **shows `#authGate` immediately** when `localStorage` has no `EMAIL_KEY`, instead of
waiting for `onAuthStateChanged`. Waiting means first pulling the Firebase SDK over the network,
and by the time that resolves Safari has already decided the page has no login form worth filling
— so it never offers the saved password or the Face ID prompt, and a signed-out visit gets typed
by hand. A stored `EMAIL_KEY` is a synchronous "this container has a session" hint: absent, show
the gate now; present, stay hidden (and prefill the email) so a signed-in load never flashes the
login screen. `onAuthStateChanged` still sets the real state afterwards either way, so the early
show is a hint, not a second source of truth — don't delete it as redundant.

This matters most from the **Scriptable widget**, whose `widget.url` opens Safari, a completely
separate storage container from the installed home-screen app. A login in one is invisible to the
other, so the widget path always lands signed-out until it's logged in on the Safari side too.
There is no iOS URL scheme to open an installed home-screen web app directly — don't go looking.

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
    match /noteImages/{userId}/{allPaths=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```
Without this, uploads/deletes from the To-Do task modal's Photos section, or from the notes
formatting toolbar's photo button, will fail with a permission error even though the client code
is correct. The `noteImages` rule is required separately from `todoImages` — adding one doesn't
cover the other.

## Feature map (roughly chronological)

- **Today tab:** checklist, a "Progress" card with three always-visible rings side by side (Day /
  Week / Month — `todayRingProgress`/`weeklyRingProgress`/`monthlyScoreRingProgress`), collapsible
  "Manage habits" (moved here from Settings — lazy-renders only when expanded, see gotcha below).
  The header itself is just the title and date — no overall streak badge or ring (removed; the
  per-habit flame+streak next to each checklist row is separate and still there). There's no
  combined "how many days in a row was I 100%" streak concept anymore, only each individual
  habit's own streak (`computeHabitStreak()`).
- **Monthly tab:** calendar grid, day-detail panel, color-coded completion (green/amber/red are
  meaningful status colors — not the theme's accent color, deliberately untouched by the
  redesign). Tapping a habit's name in the day-detail panel opens that habit's Edit modal
  (`openHabitEditModal`) — note this differs from the Today checklist, where tapping a name opens
  the read-only detail/stats modal (`openHabitDetail`) instead.
- **Notes tab:** folders (Level 1) > notes-in-folder (Level 2) > editor (Level 3). Rich text via
  a `contenteditable` div. Formatting lives behind a floating "Aa" pill (bottom-left in the
  editor) that expands into a single horizontal row of tools to its right: Bold, Italic,
  Underline, Bullet list, Numbered list, a text-size number box (`notesFmtSizeInput` — Word-style:
  an actual `<input type="number">` sitting inline in the toolbar row itself, not a popup or a
  fixed set of presets, flanked by a tiny stacked up/down stepper (`notesFmtSizeUp`/
  `notesFmtSizeDown`, ±1px per click) inside the same bordered `.notes-fmt-size-wrap`; applies on
  Enter, on blur, or on a stepper click, via `execCommand('fontSize')` converted to a real px
  value since that command only supports the legacy 1-7 scale), and a photo button
  (uploads to `noteImages/`, inserts an `<img>` at the cursor — see Data model + gotcha below for
  cleanup). Since it's a real element inside the toolbar panel (not a separate modal), the
  document-level "click outside closes the toolbar" handler needs no special-casing for it — that
  was an actual bug in an earlier popup-based version of this control, fixed by moving to this
  inline design instead. The size box doesn't try to reflect the current selection's existing
  size back into itself (unreliable across mixed contenteditable content) — it always shows the
  last size applied, defaulting to 15px to match the note body's own base font-size. The panel
  itself is intentionally tight (small gaps, ~31px buttons) so the whole row fits without
  scrolling on a narrow phone; it repositions itself to sit
  just above the on-screen keyboard via the `visualViewport` API (see
  `updateNotesToolbarKeyboardOffset()`), falling back to its default position above the mobile
  tab bar when no keyboard is showing. Also: paste sanitization (preserves structure from apps
  like Apple Notes, strips clashing fonts/colors), and a backspace fix for exiting a list cleanly.
  "⋮" menu on both folder cards and note rows (Rename/Delete/Move — replaced old inline text links
  and cross-folder drag targets). Drag-to-reorder within a folder still works.
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
5. **Both places a task's images can go away — deleting a single photo (`deleteTaskImage()`) and
   deleting the whole task (`todoTaskDeleteBtn` handler) — must call Storage `.delete()` for
   every image, not just clear the array.** Otherwise the Storage bucket accumulates orphaned
   files with nothing in Firestore pointing to them (silent, unbounded growth against the 5GB
   free-tier stored limit). If another path can remove a task with images (bulk actions, etc.),
   it needs the same cleanup.
6. **Never mutate a captured object reference across an `await`/event gap when a live Firestore
   listener is attached — always re-look it up from the current array by id first.** The sync
   listener (`attachSyncListener()`) reassigns the whole `habits`/`todos`/etc. array on *every*
   write, including this session's own writes echoing back (`docRef.onSnapshot`, typically
   tens of ms after `saveHabits()`/`saveTodos()`). A handler that closes over an object from an
   earlier render (e.g. `h` in `buildHabitRow(h)`) and mutates it directly can go stale mid-edit:
   the next `saveHabits()` pushes the live `habits` array, which no longer contains that object,
   silently discarding the edit. This was a real bug — typing a habit's metric unit (or name, or
   toggling a schedule day) could revert to an earlier keystroke's value after the echo landed.
   Fixed in `buildHabitRow()` by having every handler look up `habits.find(x => x.id === habitId)`
   fresh before mutating (see the `current()` helper there), mirroring the todo task modal's
   existing `const live = todos.find(...)` pattern. Any new per-item edit UI (habits, todos,
   notes, folders, lists) must use this same fresh-lookup pattern, never mutate a closed-over
   item directly.
7. **Note photos have no dedicated "remove" button to hook Storage cleanup into** — they're plain
   `<img>` tags inside free-form `contenteditable` HTML, so a photo can disappear via any ordinary
   edit (backspace, select-and-delete), not just a discrete action like `deleteTaskImage()`. Fixed
   by diffing the note-image URLs present before vs. after every save
   (`debouncedSaveActiveNote`, using `extractNoteImageUrls()`/`activeNoteImageUrls`) and deleting
   whatever dropped out, plus a separate sweep on whole-note deletion. If notes ever gain another
   way to bulk-replace `content` (an "undo" feature, a template picker, etc.), that path needs the
   same diff-and-delete treatment or photos will silently orphan in Storage.
8. **There are two intentionally different "milestone" concepts per habit — don't conflate them.**
   `achievements` is a lifetime record ("this habit reached a 10-day streak at some point"), used
   by the habit detail modal's "Milestones" trophy wall (`buildMilestoneWallHtml`) and the
   celebration toast (`syncHabitMilestones`) — both deliberately never reset, so re-crossing a
   milestone after a broken streak still toasts, and a wall badge earned months ago stays lit
   forever. `highestAchievedMilestone(currentStreak)` is different on purpose: it's the mini badge
   next to each habit in the Today checklist, and it tracks the *current* unbroken streak only
   (computed live from `computeHabitStreak(h).current`, not the `achievements` array) — it goes
   away the moment a streak breaks and climbs back up as a new one re-crosses each milestone. A
   habit with a lifetime-best 10-day streak that's currently broken correctly shows the "10" badge
   on its wall but no mini badge at all in the checklist. If asked to change how milestones behave,
   check which of these two the request is actually about before touching either.
9. **`openHabitDetail()`'s metric-log handlers (`metricAddBtn`, its edit-log row click, and
   `.metric-log-delete`) are a second real instance of the gotcha #6 stale-reference pattern** —
   found when checking a metric-tracked habit off (which auto-opens this same modal per
   `toggleCheckin`'s `{focusMetricLog:true}` call) let the `checkins` write's sync echo land
   *while the modal was already open*, reassigning `habits` and orphaning the `h` these handlers
   had closed over. Logging a run right after auto-opening could silently fail to save the first
   time and only work the second, once the modal had re-opened against the fresh object. Fixed
   the same way as `buildHabitRow()`: every handler now re-looks-up `habits.find(x => x.id ===
   habitId)` at click time instead of using the `h` captured when the modal opened. Any other
   modal that stays open across an async gap and mutates a captured item needs the same check.
10. **A habit has no concept of "when it started" unless it has `createdAt`.** Before this field
    existed, `isScheduled()` only checked `scheduleDaysForDate()`/`scheduleHistory` — it had no way
    to know a habit didn't exist yet on some past date, so a brand-new weekly habit was
    retroactively "scheduled" (and therefore counted as missed) on every day back to the global
    `TRACKING_START_DATE` constant, dragging down Monthly view completion % for days before the
    habit was ever added. Fixed by stamping `createdAt: todayKey` on new habits in the
    `addHabitBtn` handler and adding a cutoff at the top of `isScheduled()`. Since `isScheduled()`
    is the shared function `dayStats()`, `computeHabitStreak()`, and `scanHabitMilestoneCrossings()`
    all call, the fix cascades automatically to Monthly view, streaks, and milestones — no need to
    patch those callers separately. The cutoff only applies `if(habit.createdAt && ...)`, so
    existing habits (which have no `createdAt`) are completely unaffected. If another new
    habit-level field ever needs "since when does this apply" semantics, this is the pattern to
    follow rather than inventing a second global cutoff constant.
    - **That backward-compatibility initially meant habits created *before* this shipped kept
      counting on past days** (no `createdAt` → cutoff never applies). Fixed by
      `backfillHabitCreatedAt()`, called from the `onAuthStateChanged` handler once data has
      loaded. For each habit missing `createdAt` it takes the earliest date the habit is *known*
      to have existed — its first check-in, its first `scheduleHistory[].effectiveUntil`, or its
      `anchorDate` — and falls back to today when there's no evidence at all. It saves once and
      then no-ops forever (habits all have the field), so it's safe to run on every app open.
    - The fallback-to-today branch is the one lossy case: a habit that existed for a while but was
      *never once* checked in gets stamped today, which erases its past misses. A "Started" date
      input briefly existed in the habit Edit modal as a manual correction for this, but **the user
      asked for it to be removed** — a start date should be recorded at creation and never typed in
      by hand. Don't re-add it. If a start date ever does need correcting, the route is Settings >
      Backup & Restore (edit `createdAt` in the exported JSON, then restore), not a new field.

## Deployment

Upload `index.html` to the repo root on GitHub (manually, or however Claude Code's GitHub
integration is set up in this session) — GitHub Pages redeploys automatically. Bump the
`CACHE_NAME` version in `service-worker.js` if a change needs to force-bypass the PWA's offline
cache on users' devices.
