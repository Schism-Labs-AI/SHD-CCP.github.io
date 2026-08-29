# Firebase Setup — SHD-CCP Website

This site uses its **own** Firebase project, `shd-ccp-website`, separate from
the shared `gitlabs-3e081` project used by the other BiochainAI sites. This
keeps SHD-CCP's users, auth config, and any future data fully standalone.

## What's already configured

- Project: `shd-ccp-website` (`shd-ccp-website.firebaseapp.com`)
- Auth provider: Google Sign-In (enabled in Firebase Console → Authentication → Sign-in method)
- Authorized domain: the GitHub Pages domain has been added under
  Authentication → Settings → Authorized domains

## Access control (login gate + ban list)

`index.html` now gates all content behind Google Sign-In:

- **Signed out** → visitors see a "Sign In Required" panel in place of the
  page content (no navigation away, no content in the DOM until they sign in).
- **Signed in** → the client reads `bannedUsers/{uid}` from Firestore.
  - Document **exists** → the user is signed out and redirected to
    `access-denied.html`.
  - Document **does not exist** → the gate is hidden and the page content
    is revealed.
  - Read **fails** (offline, rules misconfigured, etc.) → the gate stays up
    with a "couldn't verify, please refresh" message. A failed check is
    never treated as a ban, so an outage can't silently lock everyone out
    to `access-denied.html`.

**You must enable Firestore** on the `shd-ccp-website` project (Firebase
Console → Build → Firestore Database → Create database) for the ban check to
work at all — until then every sign-in will fail the read and get stuck on
the gate. Use the rules in the Firestore section below.

### To ban / unban someone

Firebase Console → Firestore Database → `bannedUsers` collection:
- **Ban**: add a document whose **Document ID** is the user's Auth UID (find
  it under Authentication → Users). Fields don't matter for the check itself,
  but adding `reason` (string) and `bannedAt` (timestamp) is useful for your
  own records.
- **Unban**: delete that document.

There is no in-app admin UI for this by design — the client can only ever
read its own `bannedUsers/{ownUid}` document (see rules below), so bans have
to be managed from the console (or a trusted server/Admin SDK) rather than
from the page itself.

## Recommended security rules / settings

**1. Google Cloud API key restrictions** (defense in depth — the `apiKey` in
`index.html` is not a secret, it just identifies the Firebase project, but
restricting it stops abuse from other origins):
- Go to [Google Cloud Console → APIs & Services → Credentials](https://console.cloud.google.com/apis/credentials?project=shd-ccp-website)
- Edit the "Browser key (auto created by Firebase)" key
- Under **Application restrictions**, choose **Websites** and add:
  - `https://biochainai.github.io/shd-ccp/*` (GitHub Pages project URL)
  - Any custom domain, if one is added later (e.g. `https://shd-ccp.example.com/*`)
  - `http://localhost:*` if you test locally

**2. Authorized domains** (Authentication → Settings → Authorized domains):
- Keep only the exact domains this site is served from (GitHub Pages domain,
  plus `localhost` for local testing). Remove any domains that belong to the
  other BiochainAI sites.

**3. Firestore / Storage rules** — the page reads one Firestore collection,
`bannedUsers`, to enforce the ban list described above. Everything else is
default-denied. Apply these rules exactly (Firebase Console → Firestore
Database → Rules):

```
// Firestore
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Default deny — require auth for everything, then scope per-collection.
    match /{document=**} {
      allow read, write: if false;
    }

    // Ban list: a signed-in user may check ONLY their own document, so the
    // client can never enumerate who else is banned. Writes are blocked —
    // manage bans from the Console (or Admin SDK), never from the page.
    match /bannedUsers/{uid} {
      allow read: if request.auth != null && request.auth.uid == uid;
      allow write: if false;
    }

    // Example, if a profile/user-data collection is added later: users can
    // only read/write their own document.
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

```
// Storage
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /{allPaths=**} {
      allow read, write: if false;
    }
  }
}
```

**4. Sign-in providers** — only Google Sign-In is enabled. Do not enable
Anonymous auth unless the site actually needs guest access; anonymous users
otherwise just inflate the Auth user count with no real identity.

**5. Quotas / billing alerts** — since this project is now independent of
the other sites, set a budget alert on the `shd-ccp-website` GCP project so
usage on this site doesn't silently share/compete with the others' quotas.
