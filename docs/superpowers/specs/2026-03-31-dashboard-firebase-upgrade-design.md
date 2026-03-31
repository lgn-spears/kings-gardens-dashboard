# King's Gardens Dashboard — Firebase Upgrade Design

## Context

King's Gardens is a small landscaping company (16-30 people) in Lexington, KY. They have a staff dashboard deployed as a single `index.html` on GitHub Pages. The dashboard currently has no shared data storage — all user-entered data (messages, calendar events, observation logs) lives in browser-tab memory and is lost on refresh or invisible to other users.

The goal is to add Firebase as a shared backend so the dashboard becomes a real collaborative tool, plus add shift scheduling, upgrade the message board to a bulletin feed, and make it mobile-friendly.

## Approach

Firebase bolt-on to the existing single `index.html` on GitHub Pages. No build system, no framework migration. Firebase JS SDK loaded via CDN `<script>` tags.

## Authentication

- **Firebase Authentication** with email/password provider
- On page load: check `firebase.auth().currentUser`
  - If not logged in: show a login/register overlay (hides the dashboard)
  - If logged in: render the dashboard normally
- **Registration**: name, email, password. Display name stored in Firebase Auth profile.
- **Login**: email + password
- **Header badge**: "Signed in as [Name] · Sign out" in the top-right area next to the clock
- **No roles or permissions** — every authenticated user can read and write everything
- **Firestore security rules**: require `request.auth != null` on all reads/writes

## Data Model (Firestore)

### `messages` collection — Bulletin Board
```
{
  authorUid: string,
  authorName: string,
  body: string,
  createdAt: Timestamp
}
```
- Ordered by `createdAt` descending
- Any user can create
- Users can delete their own posts (`authorUid == request.auth.uid`)
- Real-time listener: `onSnapshot` on the collection, ordered by createdAt desc, limit 50

### `calendar` collection — Delivery Calendar Events
```
{
  date: string (YYYY-MM-DD),
  name: string,
  time: string,
  type: string (delivery | vendor | misc),
  createdBy: string (uid)
}
```
- Queried by `date` field when a day is selected
- Any user can create
- Any user can delete any event (small team, trust model)
- Real-time listener: `onSnapshot` with where clause on date range for visible month

### `observations` collection — Frost Event Log
```
{
  authorUid: string,
  authorName: string,
  body: string,
  createdAt: Timestamp
}
```
- Ordered by `createdAt` ascending (chronological log)
- Any user can create
- Persists across sessions — observations survive page refresh and are visible to all users
- Real-time listener: `onSnapshot` ordered by createdAt asc

### `schedules` collection — Shift Schedule
```
Document ID: week start date (YYYY-MM-DD, always a Monday)

{
  weekStart: string (YYYY-MM-DD),
  entries: [
    {
      personName: string,
      days: { mon: bool, tue: bool, wed: bool, thu: bool, fri: bool, sat: bool },
      jobSite: string,
      notes: string
    }
  ],
  lastEditedBy: string (name),
  lastEditedAt: Timestamp
}
```
- One document per week, keyed by Monday date
- `entries` is an array of crew assignments for that week
- Any user can add/edit/remove entries
- Real-time listener on the currently viewed week's document
- Prev/Next week navigation creates or loads the corresponding document

### Not stored in Firebase
- **Weather station API keys**: remain in `localStorage` (personal per-browser)
- **NWS weather / GDD data**: fetched from public APIs on every load, not persisted

## UI Changes

### Login/Register Overlay
- Covers the entire page when not authenticated
- Two-tab form: "Sign In" and "Register"
- Sign In: email + password + submit
- Register: display name + email + password + submit
- Error messages for wrong password, account not found, etc.
- Styled to match the existing green theme

### Header
- Add user badge to the right side: "Signed in as **[Name]** · Sign out"
- On mobile, the badge stacks below the clock

### Bulletin Board (replaces Staff Message card)
- Same card position in the grid
- Compose area at top: textarea + "Post" button
- Below: scrollable feed of messages, newest first, max height with overflow scroll
- Each message: author name (green, bold), timestamp (muted), body text
- Delete button (x) on your own posts only
- Real-time: new posts from other users appear automatically

### Shift Schedule (new card, full width span)
- Position: between the bulletin board and GDD card (or between GDD and calendar — whichever flows better)
- Weekly grid table:
  - Header row: Crew Member | Mon | Tue | Wed | Thu | Fri | Sat | Job Site
  - Data rows: one per person, green "On" pill or dash for each day
  - Job site column: text showing assignment
- Navigation: Prev Week / Next Week buttons with "Week of [date]" label
- Add row: button to add a new crew member entry (name, toggle days, job site)
- Edit: click a row to edit it inline
- Delete: remove a row
- On mobile: table scrolls horizontally, crew member name column is sticky

### Existing Cards — Firebase Wiring
- **Delivery Calendar**: `addEvent()` writes to Firestore instead of `calEvs` object. `onSnapshot` listener replaces in-memory reads. Delete writes to Firestore.
- **Frost Observation Log**: `addComment()` writes to Firestore instead of `postComments` array. `onSnapshot` listener keeps the log synced. Author name auto-filled from logged-in user.
- **NWS Weather**: no changes (public API fetch)
- **Weather Station**: no changes (localStorage keys + Ambient Weather API)
- **GDD Tracker**: no changes (Open-Meteo API fetch)
- **Frost/Freeze Alerts**: no changes (derived from NWS data)
- **Greenhouse Warning**: no changes (derived from NWS data)

### Mobile Responsiveness
- Existing media queries at 1000px and 640px already collapse the grid
- Enhance the 640px breakpoint: all cards single column, full width
- Schedule table: horizontal scroll with sticky first column
- Calendar: stack the mini-cal above the event list (already handled)
- Bulletin board: full width, slightly shorter max-height
- Touch targets: ensure all buttons are at least 44px tap target

## Firebase Project Setup (manual, by Wes)

Before the code works, someone needs to:
1. Go to https://console.firebase.google.com
2. Create a new project (e.g., "kings-gardens-dashboard")
3. Enable **Authentication** > **Email/Password** provider
4. Create a **Firestore Database** (start in test mode, then apply security rules)
5. Copy the Firebase config object (apiKey, authDomain, projectId, etc.)
6. Paste it into the `firebaseConfig` const in `index.html`

Security rules to apply:
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

## What Stays the Same

- Single `index.html` file — no build system
- GitHub Pages hosting
- All weather/GDD functionality unchanged
- Visual design language (colors, fonts, card styles)
- Demo links in the footer (freeze alert, greenhouse warning)

## What Changes

- Firebase JS SDK added via CDN script tags
- Login/register overlay added
- Staff Message card becomes Bulletin Board feed
- New Shift Schedule card
- Calendar events, observations, and messages write to Firestore
- Real-time `onSnapshot` listeners replace in-memory state
- Mobile CSS improvements
