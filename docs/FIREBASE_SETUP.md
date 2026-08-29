# Firebase Setup — SHD-CCP Website

This site uses its **own** Firebase project, `shd-ccp-website`, separate from
the shared `gitlabs-3e081` project used by the other BiochainAI sites. This
keeps SHD-CCP's users, auth config, and any future data fully standalone.

## What's already configured

- Project: `shd-ccp-website` (`shd-ccp-website.firebaseapp.com`)
- Auth provider: Google Sign-In (enabled in Firebase Console → Authentication → Sign-in method)
- Authorized domain: the GitHub Pages domain has been added under
  Authentication → Settings → Authorized domains

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

**3. Firestore / Storage rules** — this landing page currently only uses
Firebase Auth (sign-in/out) and does not read or write any Firestore or
Storage data. If a database or file storage is added later, start from a
locked-down, auth-required baseline and tighten from there:

```
// Firestore
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Default deny — require auth for everything, then scope per-collection.
    match /{document=**} {
      allow read, write: if false;
    }

    // Example: users can only read/write their own profile document.
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
