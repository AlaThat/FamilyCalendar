# Setup guide — Our Week

Your Firebase config is already in `index.html` (the keys you gave me — these are safe to have in a public repo; see the note below). Here's what's left.

## 1. Turn on Firestore + lock it down

1. console.firebase.google.com → your project → **Build → Firestore Database → Create database**. Any region, "Start in test mode" is fine for now, we're about to fix it.
2. Once created, go to the **Rules** tab and replace the contents with:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /{document=**} {
         allow read, write: if request.auth != null;
       }
     }
   }
   ```
   This means: only someone who's signed in (through the login screen you'll set up next) can read or write anything. Click **Publish**.

## 2. Create your two logins

The app now has a sign-in screen — no one can see or edit your family's data without an account you create.

1. Firebase console → **Build → Authentication → Get started**.
2. Sign-in method tab → enable **Email/Password**.
3. Users tab → **Add user** → create one for yourself and one for your husband (any email/password combo works, doesn't need to be a real inbox — e.g. `ala@ourweek.local`, though a real email is fine too).
4. That's it. Whoever's signing in on a device (including the iPad, once) uses one of those two logins.

**Note on the config keys you pasted:** Firebase's own docs confirm these are meant to be public — they identify your project, not grant access. Real access control is the Firestore Rules + login above. If you ever want to rotate them anyway (e.g. this conversation being public), you can generate a new web app in the same Firebase project and swap the values in `index.html` — the two are unrelated.

## 3. Put it online (free hosting)

1. Create a free GitHub account if you don't have one, and a new repository (e.g. `our-week`).
2. Upload `index.html`.
3. Repo → **Settings → Pages** → source = main branch → save.
4. You'll get a URL like `https://yourname.github.io/our-week/`.

Since the repo is public, anyone can *see the code* — but per step 2, they can't see or touch your actual calendar data without one of the two logins you created. That's the intended split: code public, data private.

## 4. Set up the iPad as a wall display

1. Open the app's URL in Safari on the iPad, sign in with one of the two logins.
2. Share → **Add to Home Screen**.
3. Open from the home screen icon (it'll stay signed in). It opens as a normal Safari tab,
   address bar and all — that's intentional, see the note below.
4. Settings → Accessibility → Guided Access → turn on, set a passcode → triple-click the side button to lock the iPad into the app.
5. Settings → Display & Brightness → Auto-Lock → **Never**, keep it plugged in.

**Home Screen tiles don't need to be manually refreshed.** The app checks for its own updates
every 30 minutes and reloads itself automatically once no form is open, so a tile left running
for days will still pick up changes on its own without you doing anything. Safari's own reload
button is also still right there as a manual fallback if you ever want it sooner.

**Why this isn't a chromeless/"app-like" tile:** an earlier version removed Safari's address bar
(via `apple-mobile-web-app-capable`), which briefly broke the on-screen keyboard on the sign-in
screen — a real, long-standing WebKit bug in standalone home-screen web apps on iOS (documented
across iOS 15 through at least 18), not something specific to this app's code. Reverted, and
should stay reverted unless Apple has actually fixed the underlying bug — see CLAUDE.md.

## 5. (Optional) Work calendar blocking via email + Power Automate

Skip this section entirely if you don't want the "Add to work calendar" checkbox to actually touch Outlook yet — the app works fine without it.

### 5a. Set your trigger email address
In the app: **Settings → Work calendar email** → enter the inbox address you want trigger emails sent to (your own work email, most likely). This is stored in your private Firestore data, not in the public code.

### 5b. EmailJS (free tier)
1. emailjs.com → free account.
2. Add an email service — connect any inbox as the "from" (Gmail, Outlook, etc.) — this is the identity emails will be sent *from*, separate from the "to" address you set in step 5a.
3. Create a template with three dynamic fields: set the **To Email** field to `{{to_email}}`, and put `{{subject}}` / `{{body}}` in the subject/content — EmailJS supports dynamic values in the To field, not just the body.
4. Copy your **Public Key**, **Service ID**, and **Template ID** into the `emailjsConfig` block near the top of `index.html`'s script. The public key is safe to expose (confirmed via EmailJS's own docs — worst case someone could trigger your existing template, not send arbitrary content).

### 5c. Power Automate flow — create the block
1. Power Automate → **Create → Automated cloud flow**.
2. Trigger: **Office 365 Outlook — When a new email arrives (V3)**, Subject Filter = `[FAMILYBLOCK]`.
3. Action: **Office 365 Outlook — Create event (V4)** — pull start/end/date from the email body (`DATE:`, `START:`, `END:` lines) using a **Compose** step or `split()` expressions. Put the `TAG:` line in the event's body — this is what the delete-flow matches on later.
4. Test by checking a work-block box in the app.

### 5d. Power Automate flow — remove the block (optional)
Trigger on subject `[FAMILYBLOCK-REMOVE]` → **Get calendar view of events (V3)** for a date range around the event → filter for the one whose body contains the matching `TAG:` → **Delete event (V4)**.

## What's built now

- **Bottom navigation**, month/week/day name as the primary on-screen headline — there's no top header or app-branding bar at all, to keep the whole screen for the calendar
- **Month, Week, and Day views**, toggle at the top of Calendar
- **Tap any event directly** (in Month, Week, or Day view) to open its details/edit form immediately — no need to open the day first and then find it in a list. Tapping empty space on a day still opens that day's list and the add-event form, same as before.
- **Notes field on events** — free text for a location or other details, shown in the day list and in Day view
- **Birthday age tracking** — yearly-recurring events (birthdays, etc.) can have a birth year; the calendar then shows "(turning N)" next to the title, computed fresh from the event's actual date each time, so it's correct every year with no yearly upkeep
- **Person filter** — tap a family member's name to see just their items; events assigned to everyone (or left unassigned, like a holiday) show in a distinct dark color and always appear regardless of filter
- **Real recurrence options** — daily, weekly, every 2 weeks, monthly (same date), monthly (same weekday — "1st Tuesday"), yearly — not just yearly
- Full edit/delete on calendar events, multi-day events, all-day events with no one assigned, changing an event's date without deleting and re-adding it
- **Shared assigned chores** — an unpaid chore can be assigned to more than one kid at once, the same tap-to-toggle picker as routines; either kid checking it off marks it done for both
- **Start time shown on calendar pills** (Month/Week/Day) for anything that isn't an all-day event
- **Recurring reminders** in Settings, in addition to the one-off reminders you can still add from the Today tab — pick which days they show up and they reset automatically each day, no re-adding
- **Multi-person event color** — an event assigned to more than one person but not everyone now shows as an even color-band split between just those people, instead of silently showing only the first person's color
- **US federal holidays** (all 11), **seasons** (equinoxes/solstices), and **Daylight Saving Time changes** show up automatically every year — computed from date math, not stored anywhere, so there's nothing to maintain and no Firestore writes. No Settings toggle for these; they're always shown.
- **Moon phase icons** (🌑🌓🌕🌗) next to the date number in Month and Week views, only on the four days each month a phase actually lands (new/first quarter/full/last quarter) — not on every day
- **Events on a given day now show all-day events first, then the rest by start time** — and if more than one all-day event lands on the same day, ▲/▼ arrows in that day's list let you pick which shows first
- **Manual reordering** — routines, chores, and recurring reminders each have ▲/▼ arrows in their Settings list to fix the order they show in without deleting and re-adding anything; one-off reminders on the Today tab have the same arrows directly on their rows
- **Routine days are now per-day checkboxes** (Sun–Sat, default all checked) instead of just Every day/Weekdays/Weekends — if two kids need genuinely different day patterns for what's conceptually the same routine (e.g. one has school Mon–Fri, the other Mon/Wed/Fri), create two routines with the same title, each assigned to just the one kid with its own days checked
- Reminders on Today, routines split Morning/Evening/Anytime, configurable pay period, sign-in gate, no personal data in the public source
- Home Screen tiles check for their own updates automatically every 30 minutes and reload themselves once no form is open — see "Set up the iPad as a wall display" above

## Deliberately not built yet — flagging the tradeoff, not silently skipping

- **Drag-and-drop rescheduling**: touch-drag gestures are fragile on old iPadOS Safari and can't be verified without a real browser in hand. A tap-to-move quick edit is a safer substitute with the same practical benefit — say if you want true drag-and-drop attempted anyway.
- **Importing another calendar service's feed (e.g. subscribing to a partner's calendar)**: blocked by browser CORS rules without a server-side fetch, which means enabling Firebase's paid Blaze plan (a credit card on file, even though usage would stay in the free tier). Confirm before this gets built, since it changes the "no cost" premise.
- **"First Tuesday of the month" recurrence** is now supported. Recurring + multi-day combined ("every Monday for 3 days") is not — recurring events are treated as single-day only for now.

## Troubleshooting: something you add disappears, or seems to not save

This almost always means the write to the database got rejected — most commonly because you haven't published the Firestore Security Rules yet (step 1), or the login session isn't valid. The app now shows a red banner at the bottom of the screen when this happens instead of failing silently, and logs the real error to the browser console (Safari: Settings → Advanced → turn on Web Inspector, then connect to a Mac to view it — or just watch for the red banner, it tells you enough). If you see that banner, re-check steps 1 and 2 below before assuming a feature is missing.

## What's built now

- **Calendar** as the default screen — tap any day to add, edit, or delete events
- **Recurring events** (yearly, for birthdays/holidays), **multi-day events** (conference trips), **all-day events** with no one assigned (holiday reminders)
- **Reminders** — a simple running list on the Today tab, unrelated to a specific date
- **Routines** split into Morning / Evening / Anytime per kid
- **Icon picker** — a curated tap-to-select emoji grid instead of typing (works fine on old iPad keyboards)
- **Configurable pay period** — set any real payday date + weekly/biweekly/monthly, and "this pay period" earnings are calculated from that
- **Sign-in gate** — only your two Firebase accounts can read or write anything
- Nothing in the public source code identifies your family — no names, no real chore content, no email addresses. All of that lives only in your private Firestore data, entered through Settings.

## Still manual / not built

- Editing an *individual occurrence* of a yearly recurring event isn't supported — editing changes all future years, deleting removes the whole series
- The "from" identity on trigger emails is whatever inbox you connect in EmailJS's dashboard, not something swappable from the app's Settings
- No automated Outlook *update* — if you edit a work-block event's time, the app resends a remove-then-add pair rather than a true update, so double-check Outlook after editing one
