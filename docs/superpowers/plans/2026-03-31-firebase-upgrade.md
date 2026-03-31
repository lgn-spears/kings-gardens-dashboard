# Firebase Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Firebase (auth + Firestore) to the King's Gardens dashboard so all user-entered data syncs across devices in real-time, plus add a bulletin board, shift schedule, and mobile improvements.

**Architecture:** Bolt Firebase JS SDK (compat CDN) onto the existing single `index.html`. Firebase Auth for email/password login. Firestore for shared data (messages, calendar, observations, schedules) with `onSnapshot` real-time listeners. No build system — all inline.

**Tech Stack:** Firebase Auth, Cloud Firestore, Firebase JS SDK v10 (compat CDN), vanilla JS, single HTML file, GitHub Pages hosting.

**Spec:** `docs/superpowers/specs/2026-03-31-dashboard-firebase-upgrade-design.md`

---

## File Map

All changes are to a single file:
- **Modify:** `index.html` — the entire application

Changes are organized into logical sections within this file:
1. `<head>`: Firebase CDN scripts
2. `<style>`: New CSS for auth overlay, bulletin board, schedule card, mobile
3. `<body>` top: Auth overlay HTML
4. `.header`: User badge
5. `.grid`: New/modified cards (bulletin board, schedule)
6. `<script>`: Firebase init, auth logic, Firestore CRUD + listeners

---

## Task 1: Add Firebase SDK and Config Placeholder

**Files:**
- Modify: `index.html` (head section, lines 7-8)
- Modify: `index.html` (script section, line 366)

- [ ] **Step 1: Add Firebase CDN scripts to `<head>`**

After the Google Fonts `<link>` tag (line 8), add:

```html
<script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-firestore-compat.js"></script>
```

- [ ] **Step 2: Add Firebase config and initialization at the top of the `<script>` block**

At the very beginning of the `<script>` tag (before the `// ── CLOCK ──` comment), add:

```js
// ── FIREBASE ─────────────────────────────────────
const firebaseConfig = {
  // TODO: Replace with your Firebase project config from
  // https://console.firebase.google.com → Project Settings → Your apps → Config
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT.firebasestorage.app",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID"
};
firebase.initializeApp(firebaseConfig);
const auth = firebase.auth();
const db = firebase.firestore();
```

- [ ] **Step 3: Verify the page still loads without errors**

Open `index.html` in a browser. Open DevTools console. Confirm:
- No script loading errors (Firebase SDK loads from CDN)
- The existing dashboard still renders (clock, weather, etc.)
- Console may show a Firebase "invalid config" warning — that's expected until real keys are added

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add Firebase SDK and config placeholder"
```

---

## Task 2: Authentication — CSS

**Files:**
- Modify: `index.html` (style section)

- [ ] **Step 1: Add auth overlay CSS**

Before the closing `</style>` tag, add:

```css
/* AUTH OVERLAY */
.auth-overlay{position:fixed;inset:0;background:rgba(30,60,40,.92);z-index:100;display:flex;align-items:center;justify-content:center;backdrop-filter:blur(6px)}
.auth-overlay.hidden{display:none}
.auth-box{background:var(--white);border-radius:16px;padding:32px 28px;width:100%;max-width:380px;box-shadow:0 8px 40px rgba(0,0,0,.3)}
.auth-logo{text-align:center;margin-bottom:20px}
.auth-logo-icon{width:56px;height:56px;background:var(--green);border-radius:14px;display:inline-flex;align-items:center;justify-content:center;font-size:30px;margin-bottom:8px}
.auth-logo-title{font-size:22px;font-weight:600;color:var(--text)}
.auth-logo-sub{font-size:12px;color:var(--text-muted)}
.auth-tabs{display:flex;gap:0;margin-bottom:20px;border:1px solid var(--border);border-radius:8px;overflow:hidden}
.auth-tab{flex:1;padding:9px 0;text-align:center;font-size:13px;font-weight:600;cursor:pointer;border:none;background:var(--surface);color:var(--text-muted);transition:all .15s}
.auth-tab.active{background:var(--green);color:#fff}
.auth-field{margin-bottom:12px}
.auth-field label{display:block;font-size:11px;font-weight:600;text-transform:uppercase;letter-spacing:.5px;color:var(--text-muted);margin-bottom:4px}
.auth-field input{width:100%;padding:10px 12px;font-size:14px}
.auth-submit{width:100%;padding:12px;font-size:14px;font-weight:600;margin-top:4px}
.auth-error{background:var(--red-pale);color:var(--red);font-size:12px;padding:8px 12px;border-radius:7px;margin-bottom:12px;display:none}
.auth-error.show{display:block}

/* USER BADGE */
.user-badge{font-size:11px;color:var(--text-muted);text-align:right;margin-bottom:2px}
.user-badge strong{color:var(--green)}
.user-badge a{color:var(--text-muted);cursor:pointer;text-decoration:underline}
.user-badge a:hover{color:var(--red)}
```

- [ ] **Step 2: Verify the page still loads**

Open `index.html` in a browser. Confirm the dashboard still renders normally — the new CSS is defined but not yet applied to anything.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add auth overlay and user badge CSS"
```

---

## Task 3: Authentication — HTML and Logic

**Files:**
- Modify: `index.html` (body top, header, script section)

- [ ] **Step 1: Add auth overlay HTML**

Immediately after `<body>`, before the `<div class="header">`, add:

```html
<div class="auth-overlay" id="authOverlay">
  <div class="auth-box">
    <div class="auth-logo">
      <div class="auth-logo-icon">🌿</div>
      <div class="auth-logo-title">King's Gardens</div>
      <div class="auth-logo-sub">Staff Dashboard — Sign in to continue</div>
    </div>
    <div class="auth-tabs">
      <button class="auth-tab active" id="tabLogin" onclick="switchAuthTab('login')">Sign In</button>
      <button class="auth-tab" id="tabRegister" onclick="switchAuthTab('register')">Register</button>
    </div>
    <div class="auth-error" id="authError"></div>
    <div id="authLoginForm">
      <div class="auth-field"><label>Email</label><input type="email" id="loginEmail" placeholder="you@example.com"></div>
      <div class="auth-field"><label>Password</label><input type="password" id="loginPass" placeholder="Your password"></div>
      <button class="btn-green auth-submit" onclick="doLogin()">Sign In</button>
    </div>
    <div id="authRegisterForm" style="display:none">
      <div class="auth-field"><label>Display Name</label><input type="text" id="regName" placeholder="Your name"></div>
      <div class="auth-field"><label>Email</label><input type="email" id="regEmail" placeholder="you@example.com"></div>
      <div class="auth-field"><label>Password</label><input type="password" id="regPass" placeholder="At least 6 characters"></div>
      <button class="btn-green auth-submit" onclick="doRegister()">Create Account</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Add user badge to header**

In the header `<div>` that contains the clock (the right-side div), add a user badge div above the clock. Replace this block:

```html
  <div>
    <div class="clock" id="clock">--:-- --</div>
    <div class="date-line" id="dateLine">Loading…</div>
  </div>
```

With:

```html
  <div>
    <div class="user-badge" id="userBadge"></div>
    <div class="clock" id="clock">--:-- --</div>
    <div class="date-line" id="dateLine">Loading…</div>
  </div>
```

- [ ] **Step 3: Add auth logic to the script section**

After the Firebase init block (after `const db = firebase.firestore();`), add:

```js
// ── AUTH ──────────────────────────────────────────
let currentUser = null;

function switchAuthTab(tab) {
  document.getElementById('tabLogin').classList.toggle('active', tab === 'login');
  document.getElementById('tabRegister').classList.toggle('active', tab === 'register');
  document.getElementById('authLoginForm').style.display = tab === 'login' ? '' : 'none';
  document.getElementById('authRegisterForm').style.display = tab === 'register' ? '' : 'none';
  document.getElementById('authError').classList.remove('show');
}

function showAuthError(msg) {
  const el = document.getElementById('authError');
  el.textContent = msg;
  el.classList.add('show');
}

async function doLogin() {
  const email = document.getElementById('loginEmail').value.trim();
  const pass = document.getElementById('loginPass').value;
  if (!email || !pass) { showAuthError('Enter email and password.'); return; }
  try {
    await auth.signInWithEmailAndPassword(email, pass);
  } catch (e) {
    showAuthError(e.code === 'auth/user-not-found' ? 'No account found with that email.' :
                  e.code === 'auth/wrong-password' ? 'Incorrect password.' :
                  e.code === 'auth/invalid-credential' ? 'Invalid email or password.' :
                  e.message);
  }
}

async function doRegister() {
  const name = document.getElementById('regName').value.trim();
  const email = document.getElementById('regEmail').value.trim();
  const pass = document.getElementById('regPass').value;
  if (!name || !email || !pass) { showAuthError('Fill in all fields.'); return; }
  if (pass.length < 6) { showAuthError('Password must be at least 6 characters.'); return; }
  try {
    const cred = await auth.createUserWithEmailAndPassword(email, pass);
    await cred.user.updateProfile({ displayName: name });
  } catch (e) {
    showAuthError(e.code === 'auth/email-already-in-use' ? 'That email is already registered.' : e.message);
  }
}

function signOut() { auth.signOut(); }

auth.onAuthStateChanged(user => {
  currentUser = user;
  const overlay = document.getElementById('authOverlay');
  const badge = document.getElementById('userBadge');
  if (user) {
    overlay.classList.add('hidden');
    badge.innerHTML = 'Signed in as <strong>' + (user.displayName || user.email) + '</strong> · <a onclick="signOut()">Sign out</a>';
    initFirestoreListeners();
  } else {
    overlay.classList.remove('hidden');
    badge.innerHTML = '';
  }
});
```

- [ ] **Step 4: Add a placeholder `initFirestoreListeners` function**

After the auth block, add:

```js
let listenersInitialized = false;
function initFirestoreListeners() {
  if (listenersInitialized) return;
  listenersInitialized = true;
  // Will be filled in by subsequent tasks
}
```

- [ ] **Step 5: Verify auth overlay shows on load**

Open `index.html` in a browser. Confirm:
- The green auth overlay covers the entire page
- "Sign In" tab is active by default
- Switching to "Register" tab shows the name/email/password fields
- The dashboard is hidden behind the overlay
- (Login won't work yet until Firebase config is real — that's expected)

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add authentication overlay, login/register, and user badge"
```

---

## Task 4: Bulletin Board — Replace Staff Message

**Files:**
- Modify: `index.html` (HTML card, CSS, script)

- [ ] **Step 1: Add bulletin board CSS**

Before the closing `</style>` tag, add:

```css
/* BULLETIN BOARD */
.bulletin-feed{display:flex;flex-direction:column;gap:8px;max-height:300px;overflow-y:auto;margin-bottom:10px}
.bulletin-item{background:var(--surface);border:1px solid var(--border);border-radius:8px;padding:10px 12px}
.bulletin-meta{display:flex;justify-content:space-between;align-items:center;margin-bottom:4px}
.bulletin-author{font-weight:600;color:var(--green);font-size:13px}
.bulletin-time{font-size:11px;color:var(--text-muted)}
.bulletin-body{font-size:13px;color:var(--text);line-height:1.55}
.bulletin-del{cursor:pointer;color:var(--text-muted);font-size:12px;margin-left:8px;padding:0 2px}
.bulletin-del:hover{color:var(--red)}
.bulletin-compose{display:flex;gap:8px;align-items:flex-start}
.bulletin-compose textarea{flex:1;min-height:50px;resize:vertical}
```

- [ ] **Step 2: Replace the Staff Message card HTML**

Replace the entire `<!-- MESSAGE TO STAFF -->` card block:

```html
  <!-- MESSAGE TO STAFF -->
  <div class="card">
    <div class="card-header">
      <span class="card-title">Message to Staff</span>
      <span class="msg-ts" id="msgTs"></span>
    </div>
    <div class="msg-body" id="msgBody">Welcome to King's Gardens! Check the board for today's tasks and have a great day. 🌱</div>
    <div class="msg-edit">
      <textarea id="msgInput" placeholder="Type a new message for your team…"></textarea>
      <button class="btn-green" onclick="postMessage()">Post</button>
    </div>
  </div>
```

With:

```html
  <!-- BULLETIN BOARD -->
  <div class="card">
    <div class="card-header">
      <span class="card-title">Team Bulletin Board</span>
      <span class="badge badge-live">● Live</span>
    </div>
    <div class="bulletin-compose">
      <textarea id="bulletinInput" placeholder="Post an update for the team…"></textarea>
      <button class="btn-green" onclick="postBulletin()" style="white-space:nowrap;align-self:flex-end;height:50px">Post</button>
    </div>
    <div class="bulletin-feed" id="bulletinFeed">
      <div style="font-size:12px;color:var(--text-muted);padding:8px 0">Loading messages…</div>
    </div>
  </div>
```

- [ ] **Step 3: Replace the `postMessage` function with bulletin board Firestore logic**

Remove the old `postMessage` function:

```js
// ── STAFF MESSAGE ─────────────────────────────────
function postMessage(){
  const v=document.getElementById('msgInput').value.trim();
  if(!v)return;
  document.getElementById('msgBody').textContent=v;
  document.getElementById('msgInput').value='';
  document.getElementById('msgTs').textContent='Updated '+new Date().toLocaleString('en-US',{month:'short',day:'numeric',hour:'numeric',minute:'2-digit'});
}
```

Replace with:

```js
// ── BULLETIN BOARD ───────────────────────────────
async function postBulletin() {
  const input = document.getElementById('bulletinInput');
  const body = input.value.trim();
  if (!body || !currentUser) return;
  input.value = '';
  await db.collection('messages').add({
    authorUid: currentUser.uid,
    authorName: currentUser.displayName || currentUser.email,
    body: body,
    createdAt: firebase.firestore.FieldValue.serverTimestamp()
  });
}

async function deleteBulletin(id) {
  await db.collection('messages').doc(id).delete();
}

function renderBulletin(docs) {
  const el = document.getElementById('bulletinFeed');
  if (!docs.length) {
    el.innerHTML = '<div style="font-size:12px;color:var(--text-muted);padding:8px 0">No messages yet. Post the first update!</div>';
    return;
  }
  el.innerHTML = docs.map(doc => {
    const d = doc.data();
    const ts = d.createdAt ? d.createdAt.toDate().toLocaleString('en-US', { month: 'short', day: 'numeric', hour: 'numeric', minute: '2-digit' }) : '';
    const canDelete = currentUser && d.authorUid === currentUser.uid;
    return '<div class="bulletin-item">' +
      '<div class="bulletin-meta">' +
        '<span class="bulletin-author">' + (d.authorName || 'Staff') + '</span>' +
        '<span>' +
          '<span class="bulletin-time">' + ts + '</span>' +
          (canDelete ? '<span class="bulletin-del" onclick="deleteBulletin(\'' + doc.id + '\')" title="Delete">✕</span>' : '') +
        '</span>' +
      '</div>' +
      '<div class="bulletin-body">' + d.body + '</div>' +
    '</div>';
  }).join('');
}
```

- [ ] **Step 4: Wire up the Firestore listener inside `initFirestoreListeners`**

Replace the placeholder `initFirestoreListeners` function with:

```js
let listenersInitialized = false;
function initFirestoreListeners() {
  if (listenersInitialized) return;
  listenersInitialized = true;

  // Bulletin board
  db.collection('messages').orderBy('createdAt', 'desc').limit(50)
    .onSnapshot(snap => {
      renderBulletin(snap.docs);
    }, err => {
      document.getElementById('bulletinFeed').innerHTML =
        '<div style="font-size:12px;color:var(--red)">Could not load messages: ' + err.message + '</div>';
    });
}
```

- [ ] **Step 5: Remove old CSS that is no longer needed**

Remove these CSS rules (they were for the old staff message card and are no longer referenced):

```css
/* MESSAGE */
.msg-body{...}
.msg-edit{...}
.msg-ts{...}
```

The exact CSS to remove:

```css
/* MESSAGE */
.msg-body{background:var(--surface);border-radius:8px;padding:10px 12px;font-size:14px;line-height:1.65;min-height:56px;white-space:pre-wrap;margin-bottom:10px}
.msg-edit{display:flex;gap:8px;align-items:flex-start}
.msg-edit textarea{flex:1;min-height:60px;resize:vertical}
.msg-ts{font-size:10px;color:var(--text-muted)}
```

- [ ] **Step 6: Verify bulletin board renders**

Open `index.html` in a browser. Confirm:
- The old "Message to Staff" card is gone
- The "Team Bulletin Board" card appears with a compose area and "Loading messages…" text
- (Posts won't work until Firebase config is real — that's expected)

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: replace staff message with real-time bulletin board"
```

---

## Task 5: Calendar — Wire to Firestore

**Files:**
- Modify: `index.html` (script section — calendar functions)

- [ ] **Step 1: Replace the in-memory calendar logic**

Remove the old `calEvs` variable and the `addEvent`, `deleteEvent`, `renderEvList` functions. Replace the entire calendar JS section:

Remove:

```js
// ── DELIVERY CALENDAR ─────────────────────────────
let calView=new Date(),calSel=null,calEvs={};
function fmtKey(d){return`${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`}
function changeMonth(dir){calView=new Date(calView.getFullYear(),calView.getMonth()+dir,1);renderCal()}
function selectDay(k){calSel=k;renderCal();const d=new Date(k+'T12:00:00');document.getElementById('calDateLbl').textContent=d.toLocaleDateString('en-US',{weekday:'long',month:'long',day:'numeric',year:'numeric'});renderEvList()}

function renderCal(){
  const y=calView.getFullYear(),m=calView.getMonth();
  document.getElementById('calMonthLbl').textContent=calView.toLocaleDateString('en-US',{month:'long',year:'numeric'});
  const fd=new Date(y,m,1).getDay(),dim=new Date(y,m+1,0).getDate(),tk=fmtKey(new Date());
  let h=['S','M','T','W','T','F','S'].map(d=>`<div class="cal-dow">${d}</div>`).join('');
  for(let i=0;i<fd;i++)h+='<div></div>';
  for(let d=1;d<=dim;d++){
    const k=`${y}-${String(m+1).padStart(2,'0')}-${String(d).padStart(2,'0')}`;
    const c=['cal-day',k===tk?'today':'',k===calSel&&k!==tk?'selected':'',calEvs[k]?.length?'has-event':''].filter(Boolean).join(' ');
    h+=`<div class="${c}" onclick="selectDay('${k}')">${d}</div>`;
  }
  document.getElementById('calGrid').innerHTML=h;
}

function renderEvList(){
  const el=document.getElementById('eventList'),evs=calEvs[calSel]||[];
  if(!evs.length){el.innerHTML='<div style="font-size:12px;color:var(--text-muted)">No deliveries scheduled for this date.</div>';return;}
  const dc={delivery:'dot-delivery',vendor:'dot-vendor',misc:'dot-misc'};
  el.innerHTML=evs.map((e,i)=>`<div class="event-item"><span class="ev-dot ${dc[e.type]||'dot-misc'}"></span><span class="ev-name">${e.name}</span><span class="ev-time">${e.time}</span><span class="ev-del" onclick="deleteEvent(${i})" title="Remove">✕</span></div>`).join('');
}

function addEvent(){
  if(!calSel){alert('Select a date first.');return;}
  const name=document.getElementById('evName').value.trim();
  if(!name)return;
  const time=document.getElementById('evTime').value.trim(),type=document.getElementById('evType').value;
  if(!calEvs[calSel])calEvs[calSel]=[];
  calEvs[calSel].push({name,time,type});
  document.getElementById('evName').value='';document.getElementById('evTime').value='';
  renderCal();renderEvList();
}

function deleteEvent(i){calEvs[calSel].splice(i,1);renderCal();renderEvList()}
```

Replace with:

```js
// ── DELIVERY CALENDAR ─────────────────────────────
let calView = new Date(), calSel = null, calEvsByDate = {};

function fmtKey(d) {
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`;
}
function changeMonth(dir) {
  calView = new Date(calView.getFullYear(), calView.getMonth() + dir, 1);
  renderCal();
  subscribeCalMonth();
}
function selectDay(k) {
  calSel = k;
  renderCal();
  const d = new Date(k + 'T12:00:00');
  document.getElementById('calDateLbl').textContent = d.toLocaleDateString('en-US', { weekday: 'long', month: 'long', day: 'numeric', year: 'numeric' });
  renderEvList();
}

function renderCal() {
  const y = calView.getFullYear(), m = calView.getMonth();
  document.getElementById('calMonthLbl').textContent = calView.toLocaleDateString('en-US', { month: 'long', year: 'numeric' });
  const fd = new Date(y, m, 1).getDay(), dim = new Date(y, m + 1, 0).getDate(), tk = fmtKey(new Date());
  let h = ['S', 'M', 'T', 'W', 'T', 'F', 'S'].map(d => '<div class="cal-dow">' + d + '</div>').join('');
  for (let i = 0; i < fd; i++) h += '<div></div>';
  for (let d = 1; d <= dim; d++) {
    const k = y + '-' + String(m + 1).padStart(2, '0') + '-' + String(d).padStart(2, '0');
    const c = ['cal-day', k === tk ? 'today' : '', k === calSel && k !== tk ? 'selected' : '', calEvsByDate[k]?.length ? 'has-event' : ''].filter(Boolean).join(' ');
    h += '<div class="' + c + '" onclick="selectDay(\'' + k + '\')">' + d + '</div>';
  }
  document.getElementById('calGrid').innerHTML = h;
}

function renderEvList() {
  const el = document.getElementById('eventList'), evs = calEvsByDate[calSel] || [];
  if (!evs.length) {
    el.innerHTML = '<div style="font-size:12px;color:var(--text-muted)">No deliveries scheduled for this date.</div>';
    return;
  }
  const dc = { delivery: 'dot-delivery', vendor: 'dot-vendor', misc: 'dot-misc' };
  el.innerHTML = evs.map(e =>
    '<div class="event-item">' +
      '<span class="ev-dot ' + (dc[e.type] || 'dot-misc') + '"></span>' +
      '<span class="ev-name">' + e.name + '</span>' +
      '<span class="ev-time">' + e.time + '</span>' +
      '<span class="ev-del" onclick="deleteCalEvent(\'' + e.id + '\')" title="Remove">✕</span>' +
    '</div>'
  ).join('');
}

async function addEvent() {
  if (!calSel) { alert('Select a date first.'); return; }
  const name = document.getElementById('evName').value.trim();
  if (!name || !currentUser) return;
  const time = document.getElementById('evTime').value.trim();
  const type = document.getElementById('evType').value;
  await db.collection('calendar').add({
    date: calSel,
    name: name,
    time: time,
    type: type,
    createdBy: currentUser.uid
  });
  document.getElementById('evName').value = '';
  document.getElementById('evTime').value = '';
}

async function deleteCalEvent(id) {
  await db.collection('calendar').doc(id).delete();
}

let calUnsub = null;
function subscribeCalMonth() {
  if (calUnsub) calUnsub();
  const y = calView.getFullYear(), m = calView.getMonth();
  const startKey = y + '-' + String(m + 1).padStart(2, '0') + '-01';
  const endKey = y + '-' + String(m + 1).padStart(2, '0') + '-31';
  calUnsub = db.collection('calendar')
    .where('date', '>=', startKey)
    .where('date', '<=', endKey)
    .onSnapshot(snap => {
      calEvsByDate = {};
      snap.docs.forEach(doc => {
        const d = doc.data();
        if (!calEvsByDate[d.date]) calEvsByDate[d.date] = [];
        calEvsByDate[d.date].push({ id: doc.id, ...d });
      });
      renderCal();
      renderEvList();
    });
}
```

- [ ] **Step 2: Add calendar listener to `initFirestoreListeners`**

Inside `initFirestoreListeners`, after the bulletin board listener, add:

```js
  // Calendar
  subscribeCalMonth();
```

- [ ] **Step 3: Update the INIT section**

In the `// ── INIT ──` section at the bottom, remove the `renderCal()` and `selectDay(fmtKey(new Date()))` calls since they'll now be triggered by `initFirestoreListeners`. The init section should become:

```js
// ── INIT ──────────────────────────────────────────
fetchNWS();
fetchGDD();
autoConnectStation();
renderCal();
setInterval(fetchNWS, 15 * 60 * 1000);
setInterval(fetchGDD, 60 * 60 * 1000);
```

Note: keep `renderCal()` for initial render (shows the calendar grid before auth), but remove `selectDay(fmtKey(new Date()))` — that will be called after auth succeeds, inside `initFirestoreListeners`:

```js
  // Calendar
  selectDay(fmtKey(new Date()));
  subscribeCalMonth();
```

- [ ] **Step 4: Verify calendar still renders**

Open `index.html` in a browser. Confirm:
- Calendar grid shows the current month
- (Events won't load until Firebase config is real — that's expected)

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: wire delivery calendar to Firestore with real-time sync"
```

---

## Task 6: Frost Observation Log — Wire to Firestore

**Files:**
- Modify: `index.html` (script section — comment/observation functions)

- [ ] **Step 1: Replace the in-memory observation log logic**

Replace the `postComments` variable (line near `let alertActive=false, postComments=[];`) with:

```js
let alertActive = false;
```

Replace the `addComment` function:

```js
function addComment(){
  const ae=document.getElementById('commentAuthor'),ie=document.getElementById('commentInput');
  if(!ae||!ie)return;
  const body=ie.value.trim();
  if(!body){ie.focus();return;}
  postComments.push({who:ae.value.trim()||'Staff',body,ts:new Date().toLocaleString('en-US',{month:'short',day:'numeric',hour:'numeric',minute:'2-digit'})});
  ie.value='';
  renderComments();
}
```

With:

```js
async function addComment() {
  const ie = document.getElementById('commentInput');
  if (!ie || !currentUser) return;
  const body = ie.value.trim();
  if (!body) { ie.focus(); return; }
  ie.value = '';
  await db.collection('observations').add({
    authorUid: currentUser.uid,
    authorName: currentUser.displayName || currentUser.email,
    body: body,
    createdAt: firebase.firestore.FieldValue.serverTimestamp()
  });
}
```

- [ ] **Step 2: Replace the `renderComments` function**

Replace:

```js
function renderComments(){
  const el=document.getElementById('commentList');
  if(!el)return;
  if(!postComments.length){el.innerHTML='<div style="font-size:12px;color:var(--text-muted);font-style:italic;padding:4px 0">No observations logged yet.</div>';return;}
  el.innerHTML=postComments.map(c=>`<div class="comment-item"><div class="comment-meta">${c.who} &nbsp;·&nbsp; ${c.ts}</div><div class="comment-body">${c.body}</div></div>`).join('');
  el.scrollTop=el.scrollHeight;
}
```

With:

```js
function renderComments(docs) {
  const el = document.getElementById('commentList');
  if (!el) return;
  if (!docs || !docs.length) {
    el.innerHTML = '<div style="font-size:12px;color:var(--text-muted);font-style:italic;padding:4px 0">No observations logged yet.</div>';
    return;
  }
  el.innerHTML = docs.map(doc => {
    const c = doc.data();
    const ts = c.createdAt ? c.createdAt.toDate().toLocaleString('en-US', { month: 'short', day: 'numeric', hour: 'numeric', minute: '2-digit' }) : '';
    return '<div class="comment-item"><div class="comment-meta">' + (c.authorName || 'Staff') + ' &nbsp;·&nbsp; ' + ts + '</div><div class="comment-body">' + c.body + '</div></div>';
  }).join('');
  el.scrollTop = el.scrollHeight;
}
```

- [ ] **Step 3: Update `injectCommentCard` to remove the author name input**

In `injectCommentCard`, the author name will now come from `currentUser`. Replace the author input row:

```html
      <div style="display:flex;gap:8px;margin-bottom:8px;align-items:center">
        <input id="commentAuthor" placeholder="Your name" style="width:160px">
        <span style="font-size:11px;color:var(--text-muted)">then type your note →</span>
      </div>
```

With:

```html
      <div style="font-size:12px;color:var(--text-muted);margin-bottom:8px">
        Posting as <strong style="color:var(--green)">${currentUser ? (currentUser.displayName || currentUser.email) : 'Staff'}</strong>
      </div>
```

Note: Since `injectCommentCard` builds HTML via a template string, this will use the current user at injection time.

- [ ] **Step 4: Update `clearAlert` to not check `postComments`**

Replace:

```js
function clearAlert(){
  alertActive=false;
  document.getElementById('alertSlot').innerHTML='';
  if(postComments.length===0) document.getElementById('commentSlot').innerHTML='';
}
```

With:

```js
function clearAlert() {
  alertActive = false;
  document.getElementById('alertSlot').innerHTML = '';
  // Keep comment card visible if there are observations — Firestore listener handles rendering
}
```

- [ ] **Step 5: Always show the observation log card**

The existing code only injects the observation log card when a frost alert fires. Since observations now persist in Firestore, always show the card. In `initFirestoreListeners`, add:

```js
  // Frost observations — always inject the card and listen for updates
  if (!document.getElementById('commentCard')) injectCommentCard();
  db.collection('observations').orderBy('createdAt', 'asc')
    .onSnapshot(snap => {
      if (!document.getElementById('commentCard')) injectCommentCard();
      renderComments(snap.docs);
    });
```

Also update the card title in `injectCommentCard` to be less frost-specific. In the `injectCommentCard` function, change the title/description:

Replace:
```html
        <span class="card-title">📋 Frost / Freeze Event — Staff Observation Log</span>
        <span class="badge badge-red">Log your observations</span>
```

With:
```html
        <span class="card-title">Staff Observation Log</span>
        <span class="badge badge-live">● Live</span>
```

And replace the description paragraph:
```html
      <p style="font-size:12px;color:var(--text-muted);margin-bottom:12px;line-height:1.65">
        Record plant damage, actions taken, areas affected, greenhouse temps, and anything else worth noting during or after this event. Entries are timestamped automatically.
      </p>
```

With:
```html
      <p style="font-size:12px;color:var(--text-muted);margin-bottom:12px;line-height:1.65">
        Record observations, plant conditions, actions taken, or anything the team should know. Entries are timestamped and visible to everyone.
      </p>
```

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: wire frost observation log to Firestore with real-time sync"
```

---

## Task 7: Shift Schedule — CSS

**Files:**
- Modify: `index.html` (style section)

- [ ] **Step 1: Add schedule card CSS**

Before the closing `</style>` tag, add:

```css
/* SCHEDULE */
.sched-wrap{overflow-x:auto}
.sched-table{width:100%;border-collapse:collapse;min-width:560px}
.sched-table th{font-size:10px;font-weight:700;text-transform:uppercase;letter-spacing:.5px;color:var(--text-muted);padding:6px 4px;text-align:center;border-bottom:2px solid var(--border)}
.sched-table th:first-child,.sched-table td:first-child{text-align:left;padding-left:8px;position:sticky;left:0;background:var(--white);z-index:1}
.sched-table th:last-child,.sched-table td:last-child{text-align:left;padding-left:8px}
.sched-table td{padding:8px 4px;text-align:center;border-bottom:1px solid var(--border);font-size:13px}
.sched-on{background:var(--green-pale);color:var(--green);padding:2px 8px;border-radius:10px;font-size:11px;font-weight:600;display:inline-block}
.sched-off{color:var(--border);font-size:11px}
.sched-name{font-weight:500;color:var(--text)}
.sched-site{font-size:12px;color:var(--text-mid)}
.sched-nav{display:flex;align-items:center;justify-content:space-between;margin-bottom:10px}
.sched-nav-lbl{font-size:13px;font-weight:600}
.sched-actions{display:flex;gap:6px;align-items:center;margin-top:10px;flex-wrap:wrap}
.sched-actions input,.sched-actions select{font-size:12px;padding:5px 8px}
.sched-actions input[type="text"]{min-width:100px}
.sched-row-del{cursor:pointer;color:var(--text-muted);font-size:12px;padding:0 2px}
.sched-row-del:hover{color:var(--red)}
.sched-day-toggle{cursor:pointer;user-select:none;padding:4px 8px;border-radius:6px;transition:background .1s}
.sched-day-toggle:hover{background:var(--green-pale)}
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "feat: add shift schedule CSS"
```

---

## Task 8: Shift Schedule — HTML and Logic

**Files:**
- Modify: `index.html` (grid section, script section)

- [ ] **Step 1: Add schedule card HTML**

In the `.grid` div, between the `<!-- COMMENT SLOT -->` div and the `<!-- GROWING DEGREE DAYS -->` card, add:

```html
  <!-- SHIFT SCHEDULE -->
  <div class="card span3">
    <div class="card-header">
      <span class="card-title">Weekly Schedule</span>
      <span class="badge badge-blue">Crew</span>
    </div>
    <div class="sched-nav">
      <button class="cal-nav-btn" onclick="changeSchedWeek(-1)">‹</button>
      <span class="sched-nav-lbl" id="schedWeekLbl">Loading…</span>
      <button class="cal-nav-btn" onclick="changeSchedWeek(1)">›</button>
    </div>
    <div class="sched-wrap">
      <table class="sched-table">
        <thead>
          <tr>
            <th style="min-width:110px">Crew Member</th>
            <th>Mon</th><th>Tue</th><th>Wed</th><th>Thu</th><th>Fri</th><th>Sat</th>
            <th style="min-width:120px">Job Site</th>
            <th></th>
          </tr>
        </thead>
        <tbody id="schedBody">
          <tr><td colspan="9" style="font-size:12px;color:var(--text-muted);text-align:center;padding:16px">Loading schedule…</td></tr>
        </tbody>
      </table>
    </div>
    <div class="sched-actions">
      <input type="text" id="schedName" placeholder="Name">
      <input type="text" id="schedSite" placeholder="Job site">
      <button class="btn-blue" onclick="addSchedRow()" style="white-space:nowrap">+ Add Crew</button>
    </div>
  </div>
```

- [ ] **Step 2: Add schedule logic to script section**

After the calendar section, add:

```js
// ── SHIFT SCHEDULE ───────────────────────────────
let schedWeekStart = getMonday(new Date());
let schedUnsub = null;
let schedEntries = [];

function getMonday(d) {
  const dt = new Date(d);
  const day = dt.getDay();
  const diff = dt.getDate() - day + (day === 0 ? -6 : 1);
  return new Date(dt.getFullYear(), dt.getMonth(), diff);
}

function fmtSchedKey(d) {
  return d.getFullYear() + '-' + String(d.getMonth() + 1).padStart(2, '0') + '-' + String(d.getDate()).padStart(2, '0');
}

function changeSchedWeek(dir) {
  schedWeekStart = new Date(schedWeekStart.getFullYear(), schedWeekStart.getMonth(), schedWeekStart.getDate() + dir * 7);
  subscribeSchedWeek();
}

function subscribeSchedWeek() {
  if (schedUnsub) schedUnsub();
  const key = fmtSchedKey(schedWeekStart);
  const endDate = new Date(schedWeekStart);
  endDate.setDate(endDate.getDate() + 5);
  const weekLabel = schedWeekStart.toLocaleDateString('en-US', { month: 'short', day: 'numeric' }) +
    ' – ' + endDate.toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' });
  document.getElementById('schedWeekLbl').textContent = weekLabel;

  schedUnsub = db.collection('schedules').doc(key)
    .onSnapshot(doc => {
      if (doc.exists) {
        schedEntries = doc.data().entries || [];
      } else {
        schedEntries = [];
      }
      renderSched();
    });
}

function renderSched() {
  const el = document.getElementById('schedBody');
  const days = ['mon', 'tue', 'wed', 'thu', 'fri', 'sat'];
  if (!schedEntries.length) {
    el.innerHTML = '<tr><td colspan="9" style="font-size:12px;color:var(--text-muted);text-align:center;padding:16px">No crew scheduled this week. Add someone below.</td></tr>';
    return;
  }
  el.innerHTML = schedEntries.map((entry, i) => {
    const dayCells = days.map(d =>
      '<td><span class="sched-day-toggle" onclick="toggleSchedDay(' + i + ',\'' + d + '\')">' +
        (entry.days && entry.days[d] ? '<span class="sched-on">On</span>' : '<span class="sched-off">—</span>') +
      '</span></td>'
    ).join('');
    return '<tr>' +
      '<td class="sched-name" onclick="editSchedRow(' + i + ')" style="cursor:pointer" title="Click to edit">' + (entry.personName || '') + '</td>' +
      dayCells +
      '<td class="sched-site" onclick="editSchedRow(' + i + ')" style="cursor:pointer" title="Click to edit">' + (entry.jobSite || '') + '</td>' +
      '<td><span class="sched-row-del" onclick="deleteSchedRow(' + i + ')" title="Remove">✕</span></td>' +
    '</tr>';
  }).join('');
}

async function saveSchedEntries() {
  const key = fmtSchedKey(schedWeekStart);
  await db.collection('schedules').doc(key).set({
    weekStart: key,
    entries: schedEntries,
    lastEditedBy: currentUser ? (currentUser.displayName || currentUser.email) : 'Unknown',
    lastEditedAt: firebase.firestore.FieldValue.serverTimestamp()
  });
}

async function addSchedRow() {
  const nameEl = document.getElementById('schedName');
  const siteEl = document.getElementById('schedSite');
  const name = nameEl.value.trim();
  if (!name || !currentUser) return;
  const site = siteEl.value.trim();
  schedEntries.push({
    personName: name,
    days: { mon: true, tue: true, wed: true, thu: true, fri: true, sat: false },
    jobSite: site,
    notes: ''
  });
  nameEl.value = '';
  siteEl.value = '';
  await saveSchedEntries();
}

async function toggleSchedDay(rowIndex, day) {
  if (!schedEntries[rowIndex] || !currentUser) return;
  if (!schedEntries[rowIndex].days) schedEntries[rowIndex].days = {};
  schedEntries[rowIndex].days[day] = !schedEntries[rowIndex].days[day];
  await saveSchedEntries();
}

async function deleteSchedRow(rowIndex) {
  if (!currentUser) return;
  schedEntries.splice(rowIndex, 1);
  await saveSchedEntries();
}

function editSchedRow(rowIndex) {
  const entry = schedEntries[rowIndex];
  if (!entry) return;
  const days = ['mon', 'tue', 'wed', 'thu', 'fri', 'sat'];
  const el = document.getElementById('schedBody');
  const row = el.rows[rowIndex];
  // Replace name cell with input
  row.cells[0].innerHTML = '<input type="text" id="editSchedName" value="' + (entry.personName || '') + '" style="width:100%;font-size:13px">';
  // Replace job site cell with input
  row.cells[7].innerHTML = '<input type="text" id="editSchedSite" value="' + (entry.jobSite || '') + '" style="width:100%;font-size:12px">';
  // Replace delete button with save button
  row.cells[8].innerHTML = '<button class="btn-ghost" onclick="saveSchedEdit(' + rowIndex + ')" style="font-size:11px;padding:4px 8px;white-space:nowrap">Save</button>';
}

async function saveSchedEdit(rowIndex) {
  const nameEl = document.getElementById('editSchedName');
  const siteEl = document.getElementById('editSchedSite');
  if (!nameEl || !currentUser) return;
  schedEntries[rowIndex].personName = nameEl.value.trim() || schedEntries[rowIndex].personName;
  schedEntries[rowIndex].jobSite = siteEl ? siteEl.value.trim() : schedEntries[rowIndex].jobSite;
  await saveSchedEntries();
}
```

- [ ] **Step 3: Add schedule listener to `initFirestoreListeners`**

Inside `initFirestoreListeners`, add:

```js
  // Schedule
  subscribeSchedWeek();
```

- [ ] **Step 4: Verify schedule card renders**

Open `index.html` in a browser. Confirm:
- The "Weekly Schedule" card appears between the observation slot and GDD
- Week navigation label shows the current week range
- Prev/Next buttons are visible
- The "Add Crew" form is at the bottom

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add shift schedule card with weekly Firestore sync"
```

---

## Task 9: Mobile Responsiveness

**Files:**
- Modify: `index.html` (style section — media queries)

- [ ] **Step 1: Enhance the existing media queries**

Replace the existing media queries:

```css
@media(max-width:1000px){.grid{grid-template-columns:1fr 1fr}.span3,.alert-card,.comment-card,.gh-card{grid-column:span 2}}
@media(max-width:640px){.grid{grid-template-columns:1fr}.span2,.span3,.alert-card,.comment-card,.gh-card{grid-column:span 1}.cal-wrap,.alert-top,.gh-top{flex-direction:column}}
```

With:

```css
@media(max-width:1000px){
  .grid{grid-template-columns:1fr 1fr}
  .span3,.alert-card,.comment-card,.gh-card{grid-column:span 2}
}
@media(max-width:640px){
  .grid{grid-template-columns:1fr;padding:10px 12px;gap:12px}
  .span2,.span3,.alert-card,.comment-card,.gh-card{grid-column:span 1}
  .cal-wrap,.alert-top,.gh-top{flex-direction:column}
  .header{padding:10px 14px;flex-direction:column;gap:8px;text-align:center}
  .header-left{justify-content:center}
  .user-badge{text-align:center}
  .clock{text-align:center}
  .date-line{text-align:center}
  .bulletin-feed{max-height:200px}
  .sched-table th:first-child,.sched-table td:first-child{min-width:90px}
  .sched-actions{flex-direction:column}
  .sched-actions input[type="text"]{width:100%}
  .event-add{flex-direction:column}
  .event-add input,.event-add select{width:100%}
  .gdd-bases{grid-template-columns:1fr}
  button,.btn-green,.btn-blue,.btn-ghost{min-height:44px}
  input,textarea,select{min-height:44px;font-size:16px}
  .auth-box{margin:16px;padding:24px 20px}
}
```

- [ ] **Step 2: Verify mobile layout**

Open `index.html` in a browser and use DevTools device toolbar (responsive mode) at 375px width. Confirm:
- All cards stack to single column
- Header centers with user badge
- Schedule table scrolls horizontally, crew name column stays sticky
- Calendar event form stacks vertically
- Buttons and inputs are large enough to tap (44px min)
- Auth overlay box has margins and doesn't overflow

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: improve mobile responsiveness for phones"
```

---

## Task 10: Final Cleanup and Setup Instructions

**Files:**
- Modify: `index.html` (footer, minor cleanup)

- [ ] **Step 1: Update the footer**

Replace the existing footer:

```html
<div class="footer">
  Weather auto-refreshes every 15 min &nbsp;·&nbsp;
  <a href="#" onclick="fetchNWS();
fetchGDD();
setInterval(fetchGDD,60*60*1000);return false">Refresh now</a> &nbsp;·&nbsp;
  <a href="#" onclick="demoAlert();return false">Demo: freeze alert</a> &nbsp;·&nbsp;
  <a href="#" onclick="demoGH();return false">Demo: greenhouse warning</a> &nbsp;·&nbsp;
  <a href="#" onclick="demoClear();return false">Demo: clear all</a>
</div>
```

With:

```html
<div class="footer">
  Weather auto-refreshes every 15 min &nbsp;·&nbsp; All data syncs in real-time &nbsp;·&nbsp;
  <a href="#" onclick="fetchNWS();fetchGDD();return false">Refresh weather</a> &nbsp;·&nbsp;
  <a href="#" onclick="demoAlert();return false">Demo: freeze alert</a> &nbsp;·&nbsp;
  <a href="#" onclick="demoGH();return false">Demo: greenhouse warning</a> &nbsp;·&nbsp;
  <a href="#" onclick="demoClear();return false">Demo: clear all</a>
</div>
```

- [ ] **Step 2: Add a README.md with setup instructions**

Create `README.md` in the project root:

```markdown
# King's Gardens — Staff Dashboard

Real-time collaborative dashboard for King's Gardens landscaping crew in Lexington, KY.

## Features

- **Live weather** from NWS + personal Ambient Weather station
- **Frost/freeze alerts** with automatic greenhouse closure warnings
- **Team bulletin board** — post updates visible to all crew
- **Weekly shift schedule** — crew assignments with job sites
- **Delivery calendar** — track vendor visits and truck deliveries
- **Growing Degree Days** — phenology milestones with prior-year comparison

## Setup

### 1. Create a Firebase project

1. Go to [Firebase Console](https://console.firebase.google.com)
2. Click **Add project** — name it (e.g., "kings-gardens-dashboard")
3. Skip Google Analytics (optional)
4. Once created, click the **web** icon (`</>`) to add a web app
5. Copy the `firebaseConfig` object

### 2. Enable services

- **Authentication**: Go to Authentication → Sign-in method → Enable **Email/Password**
- **Firestore**: Go to Firestore Database → Create database → Start in **production mode**

### 3. Set Firestore security rules

In Firestore → Rules, paste:

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

### 4. Add your config

Open `index.html` and replace the placeholder `firebaseConfig` object (near the top of the `<script>` tag) with your project's config.

### 5. Deploy

Push to GitHub. GitHub Pages serves `index.html` automatically.

## Tech

Single `index.html` — no build step. Firebase JS SDK loaded via CDN. Hosted on GitHub Pages.
```

- [ ] **Step 3: Commit**

```bash
git add index.html README.md
git commit -m "feat: update footer and add setup README"
```

---

## Task 11: End-to-End Verification

- [ ] **Step 1: Create a test Firebase project**

Follow the README steps to create a Firebase project and paste the real config into `index.html`.

- [ ] **Step 2: Test authentication**

1. Open `index.html` in a browser
2. Register a new account (name, email, password)
3. Confirm: overlay disappears, header shows "Signed in as [Name]"
4. Click Sign out — overlay reappears
5. Sign back in — dashboard loads

- [ ] **Step 3: Test bulletin board**

1. Post a message from browser tab A
2. Open a second tab (or incognito window, sign in as another user)
3. Confirm: the message appears in both tabs in real-time
4. Delete a message — confirm it disappears from both tabs

- [ ] **Step 4: Test calendar**

1. Select a date, add a delivery
2. In the second tab, navigate to the same date
3. Confirm: the delivery appears in both tabs
4. Delete it — confirm it disappears from both

- [ ] **Step 5: Test shift schedule**

1. Add a crew member with a job site
2. Toggle some days on/off
3. In the second tab, confirm changes appear in real-time
4. Navigate to next week — confirm it's empty
5. Navigate back — confirm data is still there

- [ ] **Step 6: Test frost observation log**

1. Click "Demo: freeze alert" in the footer
2. Post an observation
3. In the second tab, confirm the observation appears
4. Refresh the page — confirm observations persist

- [ ] **Step 7: Test mobile**

1. Open DevTools responsive mode at 375px width
2. Confirm: single column layout, header centered, schedule scrolls horizontally
3. Confirm: all buttons/inputs are tappable (44px min height)
4. Confirm: auth overlay fits on small screen

- [ ] **Step 8: Final commit (config removal)**

Remove the real Firebase config and restore the placeholder before pushing to GitHub (Wes will add his own):

```bash
git add index.html
git commit -m "chore: restore config placeholder for distribution"
```
