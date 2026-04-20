# Manual Steps — Zeyaroo Beta

Steps that can't be done in code. Check them off as you go.

---

## Session 5 — Firebase Projects

- [ ] **Enable Email/Password auth on zeyaroo-dev**
  - Firebase Console → zeyaroo-dev → Authentication → Sign-in method → Email/Password → Enable

- [ ] **Enable Email/Password auth on zeyaroo (prod)**
  - Firebase Console → zeyaroo → Authentication → Sign-in method → Email/Password → Enable

- [ ] **Add authorized domains on zeyaroo-dev**
  - Firebase Console → zeyaroo-dev → Authentication → Settings → Authorized domains
  - Add: `zeyaroo.dev.bytkloud.com`

- [ ] **Add authorized domains on zeyaroo (prod, when ready)**
  - Add: `zeyaroo.com`

- [ ] **Generate prod service account key for zeyaroo**
  - GCP Console → zeyaroo project → IAM → Service Accounts → firebase-adminsdk → Keys → Add Key
  - Store as `ZEYAROO_PROD_FIREBASE_CREDENTIALS_JSON` in Komodo variables

- [ ] **Update prod Firebase project in zeyaroo-prod Komodo stack**
  - Change `FIREBASE_PROJECT_ID=zeyaroo` and set `FIREBASE_CREDENTIALS_JSON=[[ZEYAROO_PROD_FIREBASE_CREDENTIALS_JSON]]`

---

## Session 4 — Legal + Security Essentials

- [ ] **Firebase: Password reset continue URL**
  - Firebase Console → Authentication → Templates → Password Reset
  - Set "Continue URL" to `https://zeyaroo.com/login`

---

<!-- Add new sessions above this line -->
Verification checklist:

Resize to < 768px → hamburger appears, sidebar hidden
Tap hamburger → sidebar slides in with backdrop
Tap nav link → sidebar closes, navigates
Tap backdrop → sidebar closes
Desktop (≥ 768px) → behavior unchanged