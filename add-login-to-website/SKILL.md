---
name: add-login-to-website
description: >
  Add email/password login, user accounts, and cross-device sync to a static website using Supabase.
  Use this skill whenever the user wants to: add login or authentication to a website, let multiple users have their own data, sync data across devices without GitHub Gists or localStorage hacks, add "sign in / create account" to any HTML or JS project, or make a personal tool into a shareable multi-user app.
  Trigger even for partial requests like "add accounts", "let other people use this", "I want to log in on my phone too", or "make it so each person has their own data".
---

# Add Login to a Website

## Overview

This skill adds email/password authentication + per-user cloud storage to a static website using **Supabase** (free tier). No backend or build step needed — everything runs from a CDN script tag.

**What you get:**
- Sign in / Create account modal
- Each user gets their own private data
- Auto-sync across all their devices
- Works on any single-file HTML site

---

## Step 1 — Supabase project setup (one-time, ~5 minutes)

The user does this manually in the browser. You can log in to Supabase with GitHub — no separate password needed.

1. Go to **supabase.com** → sign in with GitHub → **New project**
2. Once the project is ready, open the **SQL Editor** and run:

```sql
CREATE TABLE garden_plans (
  user_id UUID REFERENCES auth.users NOT NULL PRIMARY KEY,
  data    JSONB NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT now()
);
ALTER TABLE garden_plans ENABLE ROW LEVEL SECURITY;
CREATE POLICY "own data only" ON garden_plans FOR ALL USING (auth.uid() = user_id);
```

> Rename the table to match your app (e.g. `notes`, `plans`, `settings`). The `data JSONB` column stores whatever JSON your app needs.

3. Go to **Authentication → URL Configuration** and set **Site URL** to your live domain (e.g. `https://my-app.vercel.app`). Also add it to **Redirect URLs**.

> ⚠️ If you skip this step, confirmation emails will link to `localhost:3000` and fail.

4. Go to **Settings → API** and copy:
   - **Project URL** — looks like `https://xxxxxxxxxxxx.supabase.co`
   - **Publishable/anon key** — starts with `sb_publishable_` or `eyJ…`

---

## Step 2 — Add Supabase to the HTML

Add the CDN script tag before `</head>`:

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js"></script>
```

At the top of your `<script>` block, initialise the client:

```js
const SUPABASE_URL = 'https://xxxxxxxxxxxx.supabase.co';
const SUPABASE_KEY = 'sb_publishable_...';
const sb = supabase.createClient(SUPABASE_URL, SUPABASE_KEY);
let currentUser = null;
```

The anon key is safe to expose in public HTML — row-level security enforces access control server-side.

---

## Step 3 — Cloud push/pull functions

Replace any existing sync (localStorage-only, Gist, etc.) with these two functions:

```js
async function pushToCloud() {
  if (!currentUser) return;
  const payload = { /* your app's state */ lastModified: Date.now() };
  await sb.from('your_table').upsert({
    user_id: currentUser.id,
    data: payload,
    updated_at: new Date().toISOString()
  });
}

async function pullFromCloud() {
  if (!currentUser) return null;
  const { data } = await sb.from('your_table')
    .select('data')
    .eq('user_id', currentUser.id)
    .single();
  return data ? data.data : null;
}
```

Call `pushToCloud().catch(() => {})` (fire-and-forget) inside your existing `saveState()` function after writing to localStorage.

---

## Step 4 — Auth state management

```js
function updateHeaderAuth() {
  const btn = document.getElementById('auth-open-btn');
  if (!btn) return;
  if (currentUser) {
    const name = currentUser.email.split('@')[0];
    btn.innerHTML = `👤 ${name} <span id="auth-logout-btn" style="font-size:0.62rem;opacity:0.7;padding:1px 5px;border-radius:3px;border:1px solid rgba(255,255,255,0.35)">sign out</span>`;
    document.getElementById('auth-logout-btn').addEventListener('click', e => {
      e.stopPropagation();
      sb.auth.signOut();
    });
  } else {
    btn.textContent = 'Sign in';
  }
}

async function onSignedIn(user) {
  currentUser = user;
  updateHeaderAuth();
  const cloud = await pullFromCloud();
  if (!cloud) {
    showFirstTimeSetup(); // see Step 6
  } else if (cloud.lastModified > localLastModified) {
    applyCloudState(cloud);   // update your app's in-memory state + re-render
    saveToLocalStorage(cloud);
  }
  document.getElementById('auth-overlay').classList.add('hidden');
}

function onSignedOut() {
  currentUser = null;
  updateHeaderAuth();
}

// Wire up auth listener — fires on page load if session exists too
sb.auth.onAuthStateChange((event, session) => {
  if (event === 'SIGNED_IN' && session?.user) onSignedIn(session.user);
  if (event === 'SIGNED_OUT') onSignedOut();
});

// Restore session on page reload
sb.auth.getSession().then(({ data: { session } }) => {
  if (session?.user) onSignedIn(session.user);
});
```

---

## Step 5 — Auth modal (sign in / create account)

Add a **Sign in** button to your header:
```html
<button id="auth-open-btn" onclick="_openAuthModal()">Sign in</button>
```

Create the modal in JS. Key rules for this modal:
- **Wrap inputs in `<form>`** — required for iOS Safari to offer "Save Password"
- Use `type="submit"` on the button and handle `form.addEventListener('submit', ...)`
- Use `autocomplete="email"` and `autocomplete="current-password"` / `autocomplete="new-password"`
- Show a **Confirm password** field only during signup, and validate they match before calling Supabase

```js
function initAuthModal() {
  const el = document.createElement('div');
  el.id = 'auth-overlay';
  el.className = 'modal-overlay hidden';
  el.innerHTML = `
    <div class="modal-card">
      <div id="auth-title">Sign in</div>
      <form id="auth-form" autocomplete="on">
        <input id="auth-email" name="email" type="email" placeholder="Email" autocomplete="email">
        <input id="auth-password" name="password" type="password" placeholder="Password" autocomplete="current-password">
        <input id="auth-password2" name="password2" type="password" placeholder="Confirm password" autocomplete="new-password" style="display:none">
        <div id="auth-status"></div>
        <button type="submit" id="auth-submit">Sign in</button>
        <div>
          <span id="auth-toggle-text">No account?</span>
          <span id="auth-toggle-link" style="cursor:pointer;text-decoration:underline">Create one</span>
        </div>
      </form>
    </div>`;
  document.body.appendChild(el);

  let mode = 'signin';
  const setMode = m => {
    mode = m;
    document.getElementById('auth-title').textContent = m === 'signin' ? 'Sign in' : 'Create account';
    document.getElementById('auth-submit').textContent = m === 'signin' ? 'Sign in' : 'Create account';
    document.getElementById('auth-toggle-text').textContent = m === 'signin' ? 'No account?' : 'Already have one?';
    document.getElementById('auth-toggle-link').textContent = m === 'signin' ? 'Create one' : 'Sign in';
    document.getElementById('auth-status').textContent = '';
    document.getElementById('auth-password').autocomplete = m === 'signin' ? 'current-password' : 'new-password';
    document.getElementById('auth-password2').style.display = m === 'signup' ? '' : 'none';
    if (m === 'signin') document.getElementById('auth-password2').value = '';
  };

  document.getElementById('auth-toggle-link').addEventListener('click', () => setMode(mode === 'signin' ? 'signup' : 'signin'));

  document.getElementById('auth-form').addEventListener('submit', async e => {
    e.preventDefault();
    const email = document.getElementById('auth-email').value.trim();
    const password = document.getElementById('auth-password').value;
    if (!email || !password) { document.getElementById('auth-status').textContent = 'Enter email and password.'; return; }
    if (mode === 'signup') {
      const pw2 = document.getElementById('auth-password2').value;
      if (password !== pw2) { document.getElementById('auth-status').textContent = 'Passwords do not match.'; return; }
    }
    document.getElementById('auth-status').textContent = mode === 'signin' ? 'Signing in…' : 'Creating account…';
    if (mode === 'signin') {
      const { error } = await sb.auth.signInWithPassword({ email, password });
      if (error) document.getElementById('auth-status').textContent = error.message;
    } else {
      const { error } = await sb.auth.signUp({ email, password });
      if (error) document.getElementById('auth-status').textContent = error.message;
      else document.getElementById('auth-status').textContent = '✓ Check your email to confirm, then sign in.';
    }
  });

  window._openAuthModal = () => { setMode('signin'); el.classList.remove('hidden'); };
}
```

---

## Step 6 — First-time setup for new users

When `pullFromCloud()` returns `null`, the user is new. Show a setup screen instead of an empty app:

```js
function showFirstTimeSetup() {
  // Create a modal that collects what the user needs to configure
  // On confirm: initialise their data, call saveState(), re-render, close modal
}
```

What to ask depends on the app — number of beds, project name, preferences, etc. Collect just enough to make the first render meaningful.

---

## Gotchas to avoid

| Problem | Fix |
|---|---|
| Confirmation email links to `localhost:3000` | Set Site URL in Supabase → Authentication → URL Configuration |
| iOS Safari doesn't offer to save password | Wrap inputs in `<form>`, use `type="submit"` on the button |
| New device overwrites cloud with empty local data | On first connect, pull from cloud if `cloud.lastModified > local.lastModified` |
| Users can read each other's data | Enable RLS + add `auth.uid() = user_id` policy |
| Sign-in persists across page reloads | Call `sb.auth.getSession()` on startup and restore session if found |
