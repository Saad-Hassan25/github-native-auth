# Security Model

This project has **no backend**, so it's worth being precise about what is and
isn't protected. Read this before deploying for a real team.

## The one rule

> **A secret that ships to the browser is public.**

Anything embedded in `index.html`, committed to the repo, or fetched by the
page can be read by anyone who opens DevTools or "View Source." There is no way
around this in a static site. So the design never puts a shared/admin token in
the page or the repo.

## How tokens are handled

| Concern | Approach |
|---|---|
| Whose credential? | Every user brings **their own** Personal Access Token. There is no shared/admin token. |
| Where is the token stored? | Encrypted with the user's PIN (PBKDF2 → AES-GCM via Web Crypto) and kept in that browser's `localStorage`. |
| Does the token touch the repo or a server? | **No.** It goes only to `api.github.com` over HTTPS as the user's own request. |
| What protects it at rest? | The PIN. Without it, the stored blob is undecryptable ciphertext. |
| What if the device is lost? | The thief has ciphertext, not a token — and they'd still need the PIN. You can also revoke the token on GitHub. |

## What the PIN does and doesn't do

- ✅ Saves users from re-pasting a long token every visit.
- ✅ Encrypts the token at rest in the browser.
- ❌ It is **not** a server-checked password. It only unlocks the *local*
  encrypted blob. A short PIN is fine because the ciphertext isn't published
  anywhere — an attacker has nothing to brute-force against remotely.

## Token scope (least privilege)

Tell users to create a **fine-grained** token that is:

- limited to **only this repository**, and
- granted **Issues: Read and write** (plus **Contents: Read** only if you add
  features that read repo files).

That's the entire blast radius if a token leaks: issues in one repo.

## Threat notes

- **Public vs. private repo.** If the repo is public, anyone can read the
  issues (the "database") directly on GitHub regardless of the app. If posts
  should be private, use a **private** repo — then only members with access can
  read issues, and only their tokens work in the app.
- **Rate limits.** Authenticated GitHub API calls are limited (5,000/hour per
  user). The app uses each user's own quota, so it scales naturally.
- **XSS.** User content is HTML-escaped before rendering (`esc()` in
  `index.html`). If you add Markdown rendering, sanitize the output.
- **Self-XSS.** Because the token lives in `localStorage`, a script injected
  into the page could read it. Keep dependencies minimal (this app has zero)
  and never paste untrusted code into the console.

## Reporting

Found an issue in this template? Open a GitHub issue, or for anything sensitive
email the repository owner directly rather than filing publicly.
