# Our Week — family calendar/chore board

Skylight-style wall calendar for an iPad Air 2 (hard-capped at iPadOS 15.8 — this is a hardware
limit, not a settings issue). Built as a single-file, zero-build static web app on purpose: no
npm, no bundler, just `index.html` opened directly or hosted as-is. Keep it that way unless
there's a strong reason to add tooling — the whole point was minimal setup for a non-developer
to host and maintain.

## First thing to do

**Nothing in this app has ever been run in a real browser.** Every prior round was built and
syntax-checked (`node --check`) but never actually executed or clicked through — the only
testing available was an in-app preview that silently blocks external network requests
(external `<script src>` CDN loads, `prompt()`/`confirm()`/`alert()` dialogs), which caused
several rounds of chasing phantom "bugs" that were actually just the sandbox. **Get this running
in a real browser (a local dev server is fine even without deploying) and click through every
feature listed below before trusting that any of it works.** This is the single highest-value
thing you can do first — it will likely surface real bugs that were never actually caught.

## Stack & why

- Plain HTML/CSS/JS, Firebase compat SDK (not the modular v9+ SDK) loaded via `<script src>`
  from `gstatic.com` — compat build was chosen specifically because it needs no bundler and
  works with plain script tags, for iPadOS 15 Safari compatibility.
- Firestore for data (family members, routines, chores, calendar events, reminders, earnings,
  settings) — all realtime via `onSnapshot` listeners.
- Firebase Auth (email/password) gates the whole app — only accounts the user creates manually
  in the Firebase console can read/write. This exists because the GitHub repo hosting this file
  is public; Firestore Security Rules + this login are the actual access control, not obscurity.
- EmailJS (free tier) sends a trigger email when a calendar event is marked "add to work
  calendar," which a Power Automate flow (external, not in this repo) picks up by subject line
  and creates/removes a matching Outlook calendar block. One-way only (app → Outlook), and only
  create/remove, not true update — editing a work-blocked event currently sends remove-then-add.
- No build step, no package.json, no framework. Keep it this way.

## Why it's built this way (don't relitigate without cause)

- **No true two-way Outlook sync**: the user's work Outlook is on a company-managed Microsoft
  365 tenant that blocks self-service Azure AD app registration, which is required for Graph
  API access. This was tested and confirmed blocked. Power Automate + EmailJS was the workaround
  that doesn't require that registration.
- **Email-trigger over polling**: Power Automate's generic HTTP trigger/action requires a
  Premium license; the standard Outlook connector (mail + calendar) does not. Using an
  email-arrival trigger avoids that licensing requirement entirely.
- **Icon field is free text, not an icon-only picker**: iOS's native emoji keyboard (the
  🌐/emoji key) already provides full emoji access from any text input — there's no separate
  "iOS emoji library" API for web apps to hook into. The curated grid under the text field is a
  convenience shortcut, not a replacement for typing.
- **Firebase config values are hardcoded in the client and that's intentional**: per Firebase's
  own documentation, these are not secrets — access control is Firestore Rules + Auth, not
  hiding the config. Do not "fix" this by moving them to environment variables or a backend.
- **Drag-and-drop was explicitly deferred**, not forgotten — touch-drag gesture reliability on
  old iPadOS Safari couldn't be verified without a real device/browser in hand. A tap-to-move
  quick-edit was proposed as a lower-risk substitute. Ask before building real drag-and-drop.
- **The app does NOT use `apple-mobile-web-app-capable`** (the meta tag that makes a Home Screen
  tile open chromeless, without Safari's address bar). This was tried once to reclaim screen
  space and immediately broke the on-screen keyboard on the sign-in screen after a real
  delete-and-recreate test on the iPad. This is a real, long-standing WebKit bug in standalone
  home-screen web apps — documented across iOS 15 through at least 18, not something specific to
  this app's code — and it isn't reliably fixable from the web-app side. Don't re-add this meta
  tag without first confirming (via current, dated sources, not assumption) that Apple has
  actually fixed the underlying bug on the iPadOS version this device is capped at.
- **Importing/subscribing to external calendars (e.g. a partner's calendar feed) was explicitly
  deferred**: browsers block direct cross-origin fetches of arbitrary ICS URLs, so this needs a
  server-side fetch — which means enabling Firebase's paid Blaze plan (usage would stay in the
  free tier, but it requires a credit card on file). This breaks the "no cost" constraint the
  user has stated repeatedly. Confirm with the user before building this.

## Known real bugs already found and fixed this round — don't reintroduce

- Every Firestore write was originally unguarded — a rejected write (bad rules, bad auth) failed
  silently and, because realtime listeners fully replace local arrays on snapshot, the
  optimistically-added item would visibly vanish. All writes are now wrapped in a `dbWrite()`
  helper that surfaces a toast on failure. Keep this pattern for any new write call.
- `prompt()`/`confirm()` were used for the original add-family/add-routine/add-chore flows and
  got silently blocked in the sandboxed preview. Replaced with real modal forms. Don't reach for
  `prompt()`/`confirm()`/`alert()` anywhere in this app — use the toast (`showToast()`) or a
  modal instead, both because of the preview sandbox and because they're better UX on iPad
  anyway.
- The user pasted Firestore Security Rules into the **Realtime Database** rules editor instead
  of **Firestore Database** rules editor — those are different Firebase products with different
  rule syntaxes. If debugging rules issues, confirm which product's console page is open first.
- iOS "Smart Punctuation" converts straight quotes to curly quotes on paste, which breaks the
  Firestore rules parser. Not a code bug, but worth knowing if the user reports rules errors
  again.

## Data model (Firestore collections)

- `family`: `{name, role: 'parent'|'child', color, textColor, order}`. `textColor` is the text
  color used on that person's calendar event pills (Month/Week views, where the pill background
  is their `color`) — a hex string, defaulting to white (`#FFFFFF`) via `textColorOf()`'s fallback
  when unset (legacy members, or any doc saved before this field existed), same fallback-on-read
  pattern as everywhere else in this app rather than a migration. Chosen via a native color input
  plus a row of white/black/gray-variant preset swatches (`colorFieldHtml()`/`wireColorField()`,
  a small reusable component also used for the "everyone" text color below), since white-on-white
  or white-on-a-light-color background pills were unreadable before this existed.
- `routines`: `{title, icon, timeOfDay: 'morning'|'evening'|'anytime', assignedTo: [memberId],
  perMemberDays: {memberId: [0-6]} (Sun=0..Sat=6), order}`. `assignedTo` is no longer kids-only —
  routines can be assigned to parents too (e.g. a "take vitamins" routine), and a member with at
  least one routine assigned gets their own Today-tab card; a member with none doesn't, so
  households that skip adult routines see no empty cards. Each assignee gets their own day
  pattern on the same routine (e.g. one kid's bath nights differ from another's, or one goes to
  school Mon-Fri while another goes Mon/Wed/Fri) — this was deliberately NOT modeled by putting a
  fixed schedule on the family member, since a single person can need different day patterns for
  different routines (school days vs. bath nights) at the same time. Always read a routine's
  schedule through `effectiveDaysForMember(routine, memberId)`, which checks
  `perMemberDays[memberId]` first, then falls back to `effectiveDays(routine)` for a member with
  no explicit entry (a newly-added assignee, or the whole routine on a doc saved before per-member
  days existed — including last round's shape, a single shared top-level `days` array — or the
  original `schedule: 'daily'|'weekday'|'weekend'` string from before that). Editing a legacy
  routine and saving transparently upgrades it to `perMemberDays` for its current assignees; no
  batch migration needed. The Add/Edit Routine modal builds one day-checkbox row per currently
  selected assignee, defaulting to all 7 days for an assignee with no existing entry, and
  refreshes live as assignees are toggled on/off.
- `routineLog`: doc id `${date}_${routineId}_${memberId}` → `{date, routineId, memberId, done}`
- `chores`: `{title, icon, type: 'pool'|'assigned', value, status: 'open'|'claimed'|'done',
  assignedTo, claimedBy, scheduleType: 'recurring'|'once', perMemberDays, date, order}` — for
  `type==='pool'`, `assignedTo` is unused (`null`), `claimedBy` is a single memberId, and
  `status` tracks the claim lifecycle as before; scheduling fields don't apply to pool chores
  (anyone can grab one any day). For `type==='assigned'`, `assignedTo` is `[memberId]` (can be
  more than one person, like routines — a shared chore, so any assignee checking it off marks it
  done for all of them, since assigned chores are unpaid), and scheduling reuses the exact
  per-person day-of-week machinery built for routines: `scheduleType==='recurring'` chores carry
  `perMemberDays` and are read through `effectiveDaysForMember(chore, memberId)` exactly like a
  routine; `scheduleType==='once'` chores instead carry a single shared `date` (not per-member —
  a one-time chore happens on one calendar date regardless of who's doing it) and show up only on
  that date, for things like "pick up all the stuffies tomorrow" that shouldn't recur weekly.
  `choreAppliesOnDate(chore, memberId, dateISO)` is the single place that branches on
  `scheduleType` to answer "does this chore apply to this person on this date," mirroring
  `effectiveDaysForMember`'s role for routines; `choreScheduleSummary(chore)` renders the
  Settings-list summary text ("One time · Sep 4" vs. the routine-style days summary). Assigned
  chores no longer use `status` for completion — see `choreLog` below — `status` on an assigned
  chore doc is legacy/unused once edited under this scheme, kept around rather than migrated,
  same reasoning as routines' legacy-shape fallback.
- `choreLog`: doc id `${date}_${choreId}` → `{date, choreId, done}` — per-day completion state
  for assigned chores, the same reset-every-day pattern as `routineLog`/`recurringReminderLog`.
  This replaced a real latent bug: before `choreLog` existed, an assigned chore's completion was
  a permanent field on the chore doc with no daily reset, which would have been badly broken once
  assigned chores could recur (checking one off Monday would have kept it looking "done" forever,
  including on Wednesday). Pool chores don't use `choreLog` — their `status`/`claimedBy` lifecycle
  already isn't a daily thing.
- `earnings`: `{memberId, amount, title, date, paid}` — one row per completed pool chore
- `events`: `{title, date, endDate, allDay, startTime, endTime, recurrence:
  'none'|'weekly'|'monthly-date'|'monthly-weekday'|'yearly', weeklyDays, weeklyInterval, assignees:
  [memberId] (empty or all-members = "applies to everyone", shown in a distinct dark color; 2+
  members but not everyone shows as an equal-width multi-color gradient split, one band per
  assignee), notes, birthYear, workBlock, tag, order}` — `tag` is used to correlate with the
  Outlook block via Power Automate. `notes` is free text (location, details) shown in the
  day-list modal (tap a day, or tap an event) — not inline on the calendar pill itself in any of
  Month/Week/Day, same as the title-only pills everywhere else. `birthYear` only applies when
  `recurrence==='yearly'`; when set, the display
  title gets a computed `(turning N)` suffix (`N` = the displayed occurrence's year minus
  `birthYear`) — never stored on the title itself, so it stays correct every year with no upkeep.
  `order` (set to `Date.now()` at creation) is a tie-breaker among same-day all-day events only —
  timed events always sort by `startTime`, computed holiday/season/DST entries always sort before
  real events. It's a single global field per event, so reordering a recurring all-day event's
  same-day tie-break rank shifts it on every day it occurs, not just the one you reordered it
  from — an accepted simplification rather than a per-occurrence order override, since same-day
  all-day collisions on a recurring event are a rare edge case.

  **Weekly recurrence** (`recurrence==='weekly'`) carries an explicit `weeklyDays: [0-6]` +
  `weeklyInterval: N` — an event can repeat on more than one day per week (e.g. practice every
  Mon+Thu) and/or every N weeks instead of every week, mirroring the Outlook-style recurrence
  picker rather than the old fixed daily/weekly/biweekly options. This replaced the separate
  `'daily'`/`'biweekly'` recurrence values (each of which could only repeat on a single fixed
  day); those old values are read through the same legacy-fallback pattern as routines/chores
  (`eventWeeklyDays()`/`eventWeeklyInterval()`, never migrated) rather than a batch migration —
  `'daily'` becomes "every day of the week, every week," a `'weekly'` or `'biweekly'` doc with no
  `weeklyDays` becomes "just its own day-of-week, every 1 or 2 weeks." The interval is counted in
  whole calendar weeks (Sun-anchored) from the event's own start date, not in raw day counts, so
  a multi-day pattern (Mon+Thu) skips the *same* in-between week for every selected day, matching
  how Outlook's own "every N weeks" picker behaves.

- `eventExceptions`: doc id `${eventId}_${originalDate}` → `{eventId, date, deleted}` or
  `{eventId, date, deleted:false, overrides:{...any event fields...}}` — same
  composite-string-is-the-doc-id convention as `routineLog`/`choreLog`/`recurringReminderLog`, so
  saving is always a plain upsert with no need to check whether a doc already exists first, just
  storing a richer override object instead of a boolean. Backs **editing or deleting a single
  occurrence vs. the whole series**: tapping a recurring event (a pill in Month/Week/Day view, or
  Edit/Delete in the day-list modal) opens `openRecurrenceScopeModal()` first — "this date" vs
  "the whole series" — before anything else happens; a non-recurring event skips straight to its
  own edit form as before, since there's only one occurrence to talk about. "The whole series"
  behaves exactly like editing/deleting always did (writes/removes the base `events` doc). "This
  date" instead reads/writes one `eventExceptions` doc: `deleted:true` skips that occurrence
  entirely; `overrides` holds this occurrence's title, date/time, notes, assignees, and/or
  work-block flag where they differ from the series — including moving it to a different date,
  e.g. "this week's practice is on Wednesday instead." `eventsOnDate()` layers these on top of the
  normal recurrence math: for a date the event would normally occur on, an active exception
  suppresses it (deleted) or substitutes its overridden fields in place; separately, every event's
  exceptions are also scanned for one whose *overridden* date lands on the date being rendered, so
  a moved occurrence shows up on its new date instead of its original slot. `occurrenceDate` is
  stamped onto every rendered occurrence (the original, un-moved slot date — i.e. the exception
  key) specifically so editing/deleting an already-moved occurrence again still finds the right
  exception doc regardless of which date it's currently displayed on. Deleting the whole series
  also deletes its exception docs, so they don't linger as orphans. Occurrence-level edits
  deliberately omit `recurrence`/`weeklyDays`/`weeklyInterval`/`endDate`/`birthYear` from what can
  be overridden — those describe the *series*, not one date, so the edit form hides the Repeats
  section entirely when editing just one occurrence.
- `reminders`: `{text, done, order}` — one-off, manually added/removed from the Reminders tab,
  done state persists until deleted.
- `recurringReminders`: `{text, schedule: 'daily'|'weekday'|'weekend', order}` — configured in
  Settings; shows up automatically on the Reminders tab's list (marked with a ↻) on its scheduled
  days. (Kept the simpler 3-option `schedule` here rather than switching to `days` like routines
  — reminders don't have the "different days per kid" need that motivated the routine change.)
- `recurringReminderLog`: doc id `${date}_${reminderId}` → `{date, reminderId, done}` — same
  per-day-reset pattern as `routineLog`, just without a `memberId` since reminders aren't
  per-person.
- `settings/main`: `{payPeriodAnchor, payPeriodType: 'weekly'|'biweekly'|'monthly', workEmailTo,
  everyoneColor, everyoneTextColor}`. `everyoneColor`/`everyoneTextColor` (hex strings, default
  `#3A362C`/`#FFFFFF`, same fallback-on-read pattern as the rest of `settings/main`) are the
  background/text color for "everyone" events — anything with no assignees or assigned to the
  whole family, including the computed holidays/seasons/DST entries (they're synthesized as
  zero-assignee events, so they pick this up automatically with no extra code) and the "Everyone"
  chip in the calendar's person filter. Previously this was a hardcoded, non-editable
  `--everyone` CSS custom property; it's now still a CSS var (`--everyone` / `--everyone-text`,
  every existing `var(--everyone)` usage kept working unchanged) but `applyEveryoneColorVars()`
  re-syncs both from `state.settings` on every `renderAll()`, so a Settings save takes effect
  everywhere the var is used without hunting down each call site individually.

**Manual reordering.** Routines, chores, and recurring reminders each show ▲/▼ arrows in their
Settings list (disabled at the ends of the list); manual one-off reminders show the same arrows
directly on the Reminders tab, since they have no Settings screen of their own. `moveInList()`
handles all four: on every arrow click it swaps the two adjacent items and renumbers the *whole*
visible list sequentially (0, 1, 2, ...) rather than just swapping two raw `order` values — this
self-heals any pre-existing item with a missing/duplicate `order` (e.g. data from before this
feature existed) on the very first click, with no separate migration needed. New items default
to `order: <current list length>` so they append at the end. All-day calendar events use the
same up/down arrow UI in the day-list modal, but a lighter-weight pairwise swap via
`moveEventOrder()` instead of a full renumber, since events aren't one flat list the way
routines/chores/reminders are (see the `order` caveat above).

**Computed calendar content — not stored anywhere.** US federal holidays (all 11, always
shown — confirmed with the user, no Settings toggle), seasons (equinoxes/solstices, via Jean
Meeus's low-precision algorithm — verified against published 2026 times to the minute before
shipping), and US Daylight Saving Time changes (exact, per US law: 2nd Sunday in March / 1st
Sunday in November) are computed fresh from date math in `computedEventsOnDate()` and merged
into `eventsOnDate()` as read-only "everyone" all-day entries (`holiday:true`). They're never
written to Firestore, need no yearly upkeep, and can't be edited or deleted — `findEventById`
only searches real `state.events`, so tapping one just opens that day's list, same as tapping
empty cell space. Moon phase icons (🌑🌓🌕🌗, only on the 4 exact phase days each cycle — not
every day) are separate: computed per-date by `moonPhaseIconForDate()` and shown next to the
date number in Month/Week views, not injected as calendar entries. If any of this is ever
revisited, get explicit scope from the user first (which holiday set, on-by-default vs. a
toggle, how moon phases should display) — guessing on exactly this cost a full round-trip
once already (see the Home Screen chrome saga above).

Recurrence + multi-day (`endDate`) are mutually exclusive in the current implementation —
recurring events are treated as single-day only. This is a known simplification, not a bug.

**Week and Day views are an Outlook-style time grid, not a list.** `renderTimeGrid(dates)` is the
single shared implementation behind both — `renderWeekGrid()` passes 7 dates, `renderDayGrid()`
passes 1 — so Day view is really just a 1-column week. An all-day row is pinned above a shared
scrollable hour grid (`TIME_GRID_HOUR_HEIGHT = 48`px/hour, 24h = 1152px tall); timed events are
absolutely positioned by actual start/end time (`timedEventGeometry()`, with a 30-minute floor on
*visual* height only — never touches the stored `startTime`/`endTime`) rather than just listed in
order. Overlapping timed events are packed into side-by-side columns by `packTimedEvents()` — a
standard interval-packing sweep (sort by start time, reuse a column once its previous occupant
has ended, open a new one otherwise, grouped into clusters of mutually-touching events so an
unrelated event later in the day isn't forced to share column width with an earlier cluster).
Today's column gets a `.now-line` current-time indicator; the grid's default scroll position is
"now minus 2 hours" if today is in view, else 7 AM, so opening the view doesn't dump you at
midnight. Month view is untouched by any of this — it stays the compact multi-week grid it always
was, matching how Outlook's own Month view also doesn't show a time grid.

## Design language (established, keep consistent)

Paper-calendar aesthetic, not a SaaS dashboard — established deliberately per a design brief
requiring distinctive, non-templated choices. Warm cream/paper background (`#F6F1E4`), dark ink
text (`#2C2A24`), Fraunces serif for headings/date numbers, system sans for everything else
(no custom body webfont — performance on old iPad hardware). Family member colors are the only
real accent colors in the UI; a dark neutral by default (`--everyone`, editable in Settings, see
`everyoneColor`/`everyoneTextColor` above) marks events that apply to everyone. Bottom navigation
(not top) — there is no top header/branding bar at all
(removed deliberately to reclaim vertical space on the iPad); the on-screen headline is the
current month/week/day. The "Sign out" control lives at the bottom of the Settings tab instead
of a header, since there's nowhere else on-screen it belongs.

**Five tabs: Calendar, Today, Chores, Reminders, Settings.** Today is routines-only now — it used
to also show a person-selector strip and the reminders list, both removed: the person-selector
strip never actually filtered anything (dead UI), and reminders got their own tab since they're
conceptually unrelated to routines. Chores leads with "Today's chores" (assigned chores due
today, same per-member card layout as Today's routines) above the existing "Up for grabs" pool
and earnings strip — this used to live on the Today tab. Reminders holds both the manual one-off
list and its add/reorder controls, unchanged in behavior from before, just relocated.

## User context (for tone/scope calibration, not for hardcoding into the app)

Household has two working parents (one with a second job) and two young kids (ages 3 and 5) —
routines/chores UI needs to stay icon-first and simple for them. The user is technical enough to
navigate Firebase/Power Automate consoles but is not a developer — explain infrastructure
concepts plainly, don't assume prior Firebase/Firestore/security-rules knowledge. She has been
explicit and consistent about keeping cost near zero and pushing back when a request would
change that (e.g. Blaze plan for calendar import). Never hardcode any family member's real name,
email, or personal content into source files — all of that lives only in Firestore, entered
through the app's own Settings UI. The generic placeholder data in `index.html` (Parent 1/Kid 1
etc.) is intentional and should stay generic.

## Explicitly requested, not yet built

- Real drag-and-drop event rescheduling (deferred — see above)
- External calendar import/subscription (deferred — see above)
- True update flow for work-calendar Outlook blocks (currently remove-then-add)

## Files

- `index.html` — the entire app
- `SETUP.md` — end-user setup guide (Firebase project, Firestore rules, Auth accounts, GitHub
  Pages hosting, iPad kiosk config via Guided Access, EmailJS + Power Automate flow setup).
  Keep this updated for a non-developer audience if you change setup steps.
