# Stage 3: Firebase Auth Integration

## Overview

Add optional Firebase Authentication (Google sign-in + email/password). Required to publish public expansion packs. Everything else works anonymously. Firebase stores user credentials on their servers; we only store a `firebase_uid` linking to our local user record.

**Dependencies:** Stage 1 (users table must exist)
**Parallel with:** Can start once Stage 1 user table is deployed

---

## Architecture

```
Client                          Server                         Firebase
  │                               │                               │
  │ Firebase SDK sign-in ─────────┼──────────────────────────────→│
  │ ←── Firebase ID token ────────┼───────────────────────────────│
  │                               │                               │
  │ API request + Bearer token ──→│                               │
  │                               │ firebase-admin.verifyIdToken()│
  │                               │──────────────────────────────→│
  │                               │←── decoded token ─────────────│
  │                               │ upsert user (firebase_uid)    │
  │ ←── response + user data ─────│                               │
```

- **Client:** Firebase JS SDK loaded via CDN. Handles all auth UI flows.
- **Server:** `firebase-admin` SDK validates ID tokens. Auth middleware attaches user to `req.user` if token present.
- **Optional auth pattern:** Middleware does NOT reject requests without tokens. Protected actions (publish pack) check `req.user` exists and return 401 if not.

---

## Firebase Project Setup

**Pre-requisites (manual, one-time):**
1. Create Firebase project at console.firebase.google.com
2. Enable Authentication → Sign-in providers: Google, Email/Password
3. Download service account key JSON → store at `carkedit-api/firebase-service-account.json` (gitignored)
4. Copy web app config (apiKey, authDomain, projectId) for client

**Environment variables:**

```bash
# carkedit-api/.env
FIREBASE_SERVICE_ACCOUNT_PATH=./firebase-service-account.json
# OR inline the project ID for environments where file isn't available
FIREBASE_PROJECT_ID=carkedit-online
```

---

## Server Changes

### New Dependency

```bash
cd carkedit-api && npm install firebase-admin
```

### New File: `src/middleware/auth.ts`

```typescript
import { getAuth } from 'firebase-admin/auth';
import { Request, Response, NextFunction } from 'express';

// Extends Express Request to include user
declare global {
  namespace Express {
    interface Request {
      firebaseUser?: { uid: string; email?: string; name?: string; picture?: string };
      localUser?: { id: string; firebase_uid: string; display_name: string };
    }
  }
}

/**
 * Optional auth middleware — attaches user if Bearer token present.
 * Does NOT reject requests without tokens.
 */
export function optionalAuth() {
  return async (req: Request, res: Response, next: NextFunction) => {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
      return next();
    }

    try {
      const token = authHeader.split('Bearer ')[1];
      const decoded = await getAuth().verifyIdToken(token);
      req.firebaseUser = {
        uid: decoded.uid,
        email: decoded.email,
        name: decoded.name,
        picture: decoded.picture,
      };

      // Upsert local user record
      req.localUser = upsertUserFromFirebase(req.firebaseUser);
    } catch (err) {
      // Invalid token — proceed without user (don't reject)
      console.warn('Invalid Firebase token:', err.message);
    }
    next();
  };
}

/**
 * Required auth middleware — returns 401 if no valid token.
 * Use on protected endpoints (publish pack, etc.)
 */
export function requireAuth() {
  return async (req: Request, res: Response, next: NextFunction) => {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    try {
      const token = authHeader.split('Bearer ')[1];
      const decoded = await getAuth().verifyIdToken(token);
      req.firebaseUser = {
        uid: decoded.uid,
        email: decoded.email,
        name: decoded.name,
        picture: decoded.picture,
      };
      req.localUser = upsertUserFromFirebase(req.firebaseUser);
      next();
    } catch (err) {
      return res.status(401).json({ error: 'Invalid or expired token' });
    }
  };
}
```

### Modified: `src/index.ts`

```typescript
// Add at top
import { initializeApp, cert } from 'firebase-admin/app';
import { optionalAuth, requireAuth } from './middleware/auth.js';

// Initialize Firebase Admin (before routes)
const serviceAccountPath = process.env.FIREBASE_SERVICE_ACCOUNT_PATH;
if (serviceAccountPath) {
  initializeApp({ credential: cert(serviceAccountPath) });
} else if (process.env.FIREBASE_PROJECT_ID) {
  initializeApp({ projectId: process.env.FIREBASE_PROJECT_ID });
}

// Apply optional auth to all pack routes
app.use('/api/carkedit/packs', optionalAuth());
app.use('/api/carkedit/users', optionalAuth());

// Protected routes — require auth for publishing
app.put('/api/carkedit/packs/:id/publish', requireAuth(), (req, res) => {
  // Set visibility: 'public', status: 'published'
  // Only if req.localUser.id === pack.creator_id
});
```

### Modified: `src/db/users.ts`

Add `upsertUserFromFirebase()`:

```typescript
export function upsertUserFromFirebase(firebaseUser: {
  uid: string;
  email?: string;
  name?: string;
  picture?: string;
}): User {
  const existing = db.prepare('SELECT * FROM users WHERE firebase_uid = ?').get(firebaseUser.uid);

  if (existing) {
    db.prepare(`
      UPDATE users SET
        display_name = COALESCE(?, display_name),
        email = COALESCE(?, email),
        avatar_url = COALESCE(?, avatar_url),
        updated_at = datetime('now')
      WHERE firebase_uid = ?
    `).run(firebaseUser.name, firebaseUser.email, firebaseUser.picture, firebaseUser.uid);
    return db.prepare('SELECT * FROM users WHERE firebase_uid = ?').get(firebaseUser.uid);
  }

  const id = `usr_${crypto.randomUUID()}`;
  db.prepare(`
    INSERT INTO users (id, firebase_uid, display_name, email, avatar_url)
    VALUES (?, ?, ?, ?, ?)
  `).run(id, firebaseUser.uid, firebaseUser.name || 'User', firebaseUser.email, firebaseUser.picture);

  return db.prepare('SELECT * FROM users WHERE id = ?').get(id);
}
```

### Anonymous-to-Authenticated User Migration

When a previously anonymous user signs in with Firebase:

```typescript
export function linkAnonymousUserToFirebase(anonymousUserId: string, firebaseUid: string): User {
  // Check if Firebase user already has a record
  const existingFirebase = db.prepare('SELECT * FROM users WHERE firebase_uid = ?').get(firebaseUid);

  if (existingFirebase) {
    // Migrate packs from anonymous to existing Firebase user
    db.prepare('UPDATE expansion_packs SET creator_id = ? WHERE creator_id = ?')
      .run(existingFirebase.id, anonymousUserId);
    db.prepare('DELETE FROM users WHERE id = ?').run(anonymousUserId);
    return existingFirebase;
  }

  // Link anonymous record to Firebase
  db.prepare('UPDATE users SET firebase_uid = ?, updated_at = datetime("now") WHERE id = ?')
    .run(firebaseUid, anonymousUserId);
  return db.prepare('SELECT * FROM users WHERE id = ?').get(anonymousUserId);
}
```

New endpoint:
```
POST /api/carkedit/users/link
Body: { "anonymous_user_id": "usr_xxx" }
Header: Authorization: Bearer <firebase-token>
```

---

## Client Changes

### Modified: `index.html`

Add Firebase SDK scripts before closing `</body>`:

```html
<!-- Firebase Auth SDK (loaded async, only initialized when needed) -->
<script type="module">
  import { initializeApp } from 'https://www.gstatic.com/firebasejs/11.0.0/firebase-app.js';
  import { getAuth } from 'https://www.gstatic.com/firebasejs/11.0.0/firebase-auth.js';

  const firebaseConfig = {
    apiKey: "...",
    authDomain: "carkedit-online.firebaseapp.com",
    projectId: "carkedit-online",
  };

  window.__firebaseApp = initializeApp(firebaseConfig);
  window.__firebaseAuth = getAuth(window.__firebaseApp);
</script>
```

### New File: `js/managers/auth-manager.js`

```javascript
import {
  signInWithPopup,
  GoogleAuthProvider,
  signInWithEmailAndPassword,
  createUserWithEmailAndPassword,
  signOut,
  onAuthStateChanged,
} from 'https://www.gstatic.com/firebasejs/11.0.0/firebase-auth.js';

const googleProvider = new GoogleAuthProvider();

export function initAuth(onUserChanged) {
  onAuthStateChanged(window.__firebaseAuth, async (firebaseUser) => {
    if (firebaseUser) {
      const token = await firebaseUser.getIdToken();
      const localUser = await linkOrFetchUser(firebaseUser, token);
      onUserChanged({ firebaseUser, localUser, token });
    } else {
      onUserChanged({ firebaseUser: null, localUser: null, token: null });
    }
  });
}

export async function signInWithGoogle() {
  return signInWithPopup(window.__firebaseAuth, googleProvider);
}

export async function signInWithEmail(email, password) {
  return signInWithEmailAndPassword(window.__firebaseAuth, email, password);
}

export async function signUpWithEmail(email, password) {
  return createUserWithEmailAndPassword(window.__firebaseAuth, email, password);
}

export async function logOut() {
  return signOut(window.__firebaseAuth);
}

async function linkOrFetchUser(firebaseUser, token) {
  const anonymousId = localStorage.getItem('carkedit-user-id');
  if (anonymousId) {
    // Link anonymous user to Firebase account
    const res = await fetch('/api/carkedit/users/link', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
      body: JSON.stringify({ anonymous_user_id: anonymousId }),
    });
    const user = await res.json();
    localStorage.setItem('carkedit-user-id', user.id);
    return user;
  }
  // Just upsert the Firebase user
  const res = await fetch('/api/carkedit/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
    body: JSON.stringify({
      firebase_uid: firebaseUser.uid,
      display_name: firebaseUser.displayName || 'User',
      email: firebaseUser.email,
      avatar_url: firebaseUser.photoURL,
    }),
  });
  const user = await res.json();
  localStorage.setItem('carkedit-user-id', user.id);
  return user;
}

export function getAuthToken() {
  return window.__firebaseAuth?.currentUser?.getIdToken();
}
```

### New File: `js/components/auth-button.js`

```javascript
export function renderAuthButton(state) {
  if (state.authUser) {
    return `
      <div class="auth-bar">
        <span class="auth-bar__name">${state.authUser.display_name}</span>
        <button class="btn btn--small" onclick="window.game.logOut()">Log Out</button>
      </div>
    `;
  }
  return `
    <div class="auth-bar">
      <button class="btn btn--small btn--primary" onclick="window.game.showLogin()">Sign In</button>
    </div>
  `;
}
```

### State Additions (`js/state.js`)

```javascript
authUser: null,           // Local user object { id, display_name, email, avatar_url }
firebaseUser: null,       // Firebase user (for display name, photo)
authToken: null,          // Current Firebase ID token (refreshed automatically)
authLoading: true,        // True until first auth state resolved
showLoginModal: false,    // Show login/signup modal
loginMode: 'signin',      // 'signin' | 'signup'
```

### Modified: `js/managers/pack-manager.js`

Update all fetch calls to include auth token header when available:

```javascript
async function authFetch(url, options = {}) {
  const state = getState();
  if (state.authToken) {
    options.headers = {
      ...options.headers,
      Authorization: `Bearer ${state.authToken}`,
    };
  }
  return fetch(url, options);
}
```

---

## Login Modal

A simple overlay modal triggered by "Sign In" button:

```
┌─────────────────────────────────┐
│         Sign In                 │
│                                 │
│  [🔵 Sign in with Google]       │
│                                 │
│  ─── or ───                     │
│                                 │
│  Email:    [________________]   │
│  Password: [________________]   │
│                                 │
│  [Sign In]                      │
│  Don't have an account? Sign Up │
│                                 │
│  [✕ Close]                      │
└─────────────────────────────────┘
```

---

## Protected Actions

After Stage 3, these actions require authentication:
- Publishing a pack (setting `status: 'published'` and `visibility: 'public'`)
- Future: saving public packs, writing reviews

These actions still work anonymously:
- Creating packs (private, draft)
- Adding/editing/deleting cards in own packs
- Playing games (including with expansion packs)

---

## File Changes Summary

### Modified Files

| File | Changes |
|------|---------|
| `carkedit-api/package.json` | Add `firebase-admin` dependency |
| `carkedit-api/src/index.ts` | Initialize Firebase Admin, apply auth middleware to routes |
| `carkedit-api/src/db/users.ts` | Add `upsertUserFromFirebase()`, `linkAnonymousUserToFirebase()` |
| `carkedit-online/index.html` | Add Firebase JS SDK module imports |
| `carkedit-online/js/state.js` | Add auth state fields |
| `carkedit-online/js/router.js` | Add auth handlers to `window.game`, init auth on app start |
| `carkedit-online/js/screens/menu.js` | Add auth button component |
| `carkedit-online/js/screens/card-designer.js` | Show "Sign in to publish" for anonymous users |
| `carkedit-online/js/managers/pack-manager.js` | Add auth token to fetch requests |

### New Files

| File | Purpose |
|------|---------|
| `carkedit-api/src/middleware/auth.ts` | Firebase token verification middleware |
| `carkedit-online/js/managers/auth-manager.js` | Firebase Auth client: sign-in, sign-out, token management |
| `carkedit-online/js/components/auth-button.js` | Login/logout UI component |
| `carkedit-online/css/auth.css` | Login modal and auth bar styles |
| `carkedit-api/.env.example` | Document required env vars |

### Gitignored

| File | Reason |
|------|--------|
| `carkedit-api/firebase-service-account.json` | Contains private key |
| `carkedit-api/.env` | Contains config |

---

## Testing

### Test Scenarios

```bash
PORT=4500

# 1. Server starts without Firebase config (graceful degradation)
FIREBASE_SERVICE_ACCOUNT_PATH= npm start
# Should log warning but not crash

# 2. Unauthenticated pack creation works
curl -s -X POST http://localhost:$PORT/api/carkedit/packs \
  -H "Content-Type: application/json" \
  -d '{"creator_id": "usr_anon", "title": "My Pack"}' | jq .

# 3. Unauthenticated publish fails
curl -s -X PUT http://localhost:$PORT/api/carkedit/packs/$PACK_ID/publish \
  -H "Content-Type: application/json" | jq .
# Should return 401

# 4. Authenticated publish works
TOKEN="<firebase-id-token>"
curl -s -X PUT http://localhost:$PORT/api/carkedit/packs/$PACK_ID/publish \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" | jq .
# Should return 200
```

### Manual UI Testing

- [ ] Google sign-in popup works
- [ ] Email sign-in/sign-up works
- [ ] User name displayed in menu after login
- [ ] Logout works
- [ ] Anonymous user can create/edit private packs
- [ ] Anonymous user sees "Sign in to publish" on pack editor
- [ ] After signing in, anonymous packs migrate to authenticated user
- [ ] Token refreshes automatically (test by waiting 1 hour)
- [ ] Server gracefully handles expired/invalid tokens

---

## Risks

| Risk | Mitigation |
|------|------------|
| Firebase SDK size (~100KB) | Load as ES module (tree-shakeable), async import |
| Token expiration during long sessions | Firebase SDK auto-refreshes; `getIdToken()` always returns valid token |
| Anonymous → authenticated migration loses data | Transaction-based migration; test thoroughly |
| Firebase project config exposed in client | This is expected — Firebase config is not secret (rate limiting + domain restrictions handle abuse) |
| Server runs without Firebase config | Graceful degradation — auth endpoints return 503, non-auth endpoints work normally |
