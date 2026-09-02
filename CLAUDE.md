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

- `family`: `{name, role: 'parent'|'child', color, order}`
- `routines`: `{title, icon, schedule: 'daily'|'weekday'|'weekend', timeOfDay:
  'morning'|'evening'|'anytime', assignedTo: [memberId]}`
- `routineLog`: doc id `${date}_${routineId}_${memberId}` → `{date, routineId, memberId, done}`
- `chores`: `{title, icon, type: 'pool'|'assigned', value, status: 'open'|'claimed'|'done',
  assignedTo, claimedBy}`
- `earnings`: `{memberId, amount, title, date, paid}` — one row per completed pool chore
- `events`: `{title, date, endDate, allDay, startTime, endTime, recurrence:
  'none'|'daily'|'weekly'|'biweekly'|'monthly-date'|'monthly-weekday'|'yearly', assignees:
  [memberId] (empty or all-members = "applies to everyone", shown in a distinct dark color),
  workBlock, tag}` — `tag` is used to correlate with the Outlook block via Power Automate
- `reminders`: `{text, done}`
- `settings/main`: `{payPeriodAnchor, payPeriodType: 'weekly'|'biweekly'|'monthly',
  workEmailTo}`

Recurrence + multi-day (`endDate`) are mutually exclusive in the current implementation —
recurring events are treated as single-day only. This is a known simplification, not a bug.

## Design language (established, keep consistent)

Paper-calendar aesthetic, not a SaaS dashboard — established deliberately per a design brief
requiring distinctive, non-templated choices. Warm cream/paper background (`#F6F1E4`), dark ink
text (`#2C2A24`), Fraunces serif for headings/date numbers, system sans for everything else
(no custom body webfont — performance on old iPad hardware). Family member colors are the only
real accent colors in the UI; a fixed dark neutral (`--everyone: #3A362C`) marks events that
apply to everyone. Bottom navigation (not top) — the on-screen headline is the current
month/week, not app branding.

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
- Editing a single occurrence of a recurring event (currently edits/deletes the whole series)

## Files

- `index.html` — the entire app
- `SETUP.md` — end-user setup guide (Firebase project, Firestore rules, Auth accounts, GitHub
  Pages hosting, iPad kiosk config via Guided Access, EmailJS + Power Automate flow setup).
  Keep this updated for a non-developer audience if you change setup steps.
