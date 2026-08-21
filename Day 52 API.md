# API.md — Data Access Layer

**Important context:** this project has **no custom REST/HTTP server**. The client talks directly to Firebase's managed SDKs (Firestore + Auth). So "endpoints" below are the **SDK calls that function as the API surface** — each documented exactly like a REST endpoint would be, so the contract is just as clear even though there's no backend code to write. No implementation yet, per the brief — this is the contract only.

## Endpoint 1 — Sign In

- **Purpose:** Authenticate a user before showing dashboard data.
- **Call:** `signInWithEmailAndPassword(auth, email, password)` (Firebase Auth SDK, called from `login.html`)
- **Request:** `{ email: string, password: string }`
- **Response (success):** Firebase `UserCredential` object containing `user.uid`, `user.email`
- **Validation:** Email format checked client-side before submit; password non-empty
- **Authentication:** N/A (this call *is* the authentication step)
- **Error cases:**
  - `auth/invalid-email` → "Please enter a valid email address"
  - `auth/wrong-password` / `auth/user-not-found` → "Incorrect email or password"
  - `auth/too-many-requests` → "Too many attempts — try again later"

## Endpoint 2 — Sign Out

- **Purpose:** End the user's session and return to login.
- **Call:** `signOut(auth)`
- **Request:** none (uses current session)
- **Response:** Promise resolves; app redirects to `login.html`
- **Validation:** N/A
- **Authentication:** Must have an active session (button only shown when logged in)
- **Error cases:** Network failure → show non-blocking toast, retry on next click

## Endpoint 3 — Auth State Check (Route Guard)

- **Purpose:** Protect `index.html` from unauthenticated access.
- **Call:** `onAuthStateChanged(auth, callback)`
- **Request:** none
- **Response:** `user` object or `null`
- **Validation:** N/A
- **Authentication:** This *is* the check — if `null`, redirect to `login.html` before rendering any data
- **Error cases:** None expected; treat any error as "not authenticated" and redirect

## Endpoint 4 — Fetch All Regions (Dashboard Load)

- **Purpose:** Load all 5 cities' full 10-year datasets for the dashboard.
- **Call:** `getDocs(collection(db, 'regions'))`
- **Request:** none (fetches entire collection — only 5 documents, safe to load in full)
- **Response:** Array of 5 documents matching the `SCHEMA.md` shape
- **Validation:** N/A (read-only)
- **Authentication:** Requires signed-in user; enforced by Firestore Security Rules (`request.auth != null`)
- **Error cases:**
  - `permission-denied` → show "You don't have access to this data" and redirect to login
  - Network/timeout → show retry-able error state instead of blank page (per Day 9 polish requirement)

## Endpoint 5 — Fetch Single Region (Detail View)

- **Purpose:** Support the "view details" link from the region table (Day 5/7).
- **Call:** `getDoc(doc(db, 'regions', cityId))`
- **Request:** `cityId: string` (e.g., `"dubai"`)
- **Response:** Single document matching `SCHEMA.md` shape
- **Validation:** `cityId` must be one of the 5 known slugs (checked client-side before calling)
- **Authentication:** Requires signed-in user; enforced by Security Rules
- **Error cases:**
  - Document not found → "No data available for this city"
  - `permission-denied` → same handling as Endpoint 4

## Endpoint 6 — Data Entry / Corrections (Admin, out-of-band)

- **Purpose:** Populate or correct city data (Days 2–4, and any post-launch bug fixes).
- **Call:** Manual upload via Firebase Console **or** a one-off local script using the Firebase Admin SDK — **not** exposed in the deployed client app.
- **Request:** Full document matching `SCHEMA.md` shape
- **Response:** Write confirmation
- **Validation:** Run the Day 4 validation script against the JSON file *before* uploading
- **Authentication:** Firebase project owner credentials only (never exposed to end users)
- **Error cases:** Schema mismatch → validation script blocks the upload before it reaches Firestore

## Notes on What Is *Not* an Endpoint

The following are **pure client-side computations on already-fetched data** — they are not network calls and therefore have no request/response contract:
- Scenario Planner forecast engine (2024–2040 projection)
- CAGR calculator
- KPI calculations (bed density, bed gap, private-share %)
- Chart rendering (Chart.js)

This distinction matters for the architecture: after Endpoint 4 runs once at page load, the entire interactive experience (Scenario Planner, filters, charts) requires **zero additional network requests**.
