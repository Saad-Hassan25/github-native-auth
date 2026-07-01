# 🔐 GitHub-Native Auth Kit

> A complete, multi-user web app with **no backend** - authentication, database,
> and automation all run on GitHub itself.

![stack](https://img.shields.io/badge/stack-GitHub%20Pages%20%2B%20Issues%20%2B%20Actions-34d399?style=for-the-badge)
![backend](https://img.shields.io/badge/backend-none-blue?style=for-the-badge)
![license](https://img.shields.io/badge/license-MIT-555?style=for-the-badge)

This is a small, well-commented template that shows how to build a real app —
sign-in, a feed, posting, reactions, notifications — using **only** features
that come with every GitHub repository. No server, no database, no hosting cost.

The interesting part is **authentication without a backend**, so that gets a
[dedicated deep-dive](docs/AUTHENTICATION.md).

| Layer | Powered by | Role |
|---|---|---|
| Frontend | **GitHub Pages** | Serves one static `index.html` |
| Database | **GitHub Issues** | One issue = one post |
| Auth | **User's own PAT** | Verified via the GitHub API, then PIN-locked on-device |
| Automation | **GitHub Actions** | Notifies a channel when a post is created |

---

## ✨ How authentication works (the short version)

1. A user signs in **once** by pasting a fine-grained GitHub token.
2. The app calls `GET /user` to confirm who they are.
3. They set a **PIN**; the app encrypts the token with it (Web Crypto:
   PBKDF2 + AES-GCM) and stores the ciphertext in **their browser only**.
4. Next time, they just enter the PIN — the token is decrypted locally.

The token is **never** written to the repo or any server. There is **no shared
admin token**. Full explanation + code walkthrough → **[docs/AUTHENTICATION.md](docs/AUTHENTICATION.md)**.

---

## 🚀 Quick start (5 minutes)

### 1. Create your repo
Fork this, or copy these files into a new repository.

### 2. Configure the app
Open [`index.html`](index.html) and edit the `CONFIG` block near the top:

```js
const CONFIG = {
  APP_NAME: "Pulse",
  REPO: "OWNER/REPO",          // leave as-is to auto-detect on GitHub Pages
  LABEL: "post",               // issues with this label are posts
  AVAILABLE_TAGS: ["idea", "research", "tool", "question", "win"],
  PIN_LENGTH: 4,
  TOKEN_HELP_URL: "https://github.com/settings/tokens?type=beta"
};
```

**`REPO` is usually zero-config:** if you leave it as `"OWNER/REPO"` and serve
the site from GitHub Pages (`https://<owner>.github.io/<repo>/`), the app derives
`owner/repo` from the URL automatically. You only need to set it explicitly when
using a **custom domain** or running **locally** (where the URL can't be parsed).

### 3. Enable GitHub Pages
**Settings → Pages →** Source: *Deploy from a branch* → `main` / `root` → **Save**.
Your site goes live at `https://<owner>.github.io/<repo>/`.

### 4. Create the label
**Issues → Labels → New label →** name it `post` (must match `CONFIG.LABEL`).
Tag labels (`idea`, `research`, …) are created automatically the first time
they're used.

### 5. (Optional) Tracking issue for the workflow
The default Action logs new posts as comments on **issue #1**. Create one issue
titled e.g. "📋 Activity log" and pin it. To skip this, edit
[`.github/workflows/notify.yml`](.github/workflows/notify.yml).

### 6. Tell users how to get a token
Each user creates their own **fine-grained** token at
[github.com/settings/tokens?type=beta](https://github.com/settings/tokens?type=beta):

- **Repository access:** only your repo
- **Permissions → Issues:** Read and write
- **Expiration:** your call (90 days is reasonable)

They paste it once in the app, set a PIN, and they're done.

---

## 🔔 Notifications (optional)

[`.github/workflows/notify.yml`](.github/workflows/notify.yml) runs whenever a
labeled post is created. Out of the box it logs to the tracking issue. To alert
a channel, uncomment one block and add the matching repo secret
(**Settings → Secrets and variables → Actions**):

- **Slack** → `SLACK_WEBHOOK_URL`
- **Discord** → `DISCORD_WEBHOOK_URL`
- **Email** → `SMTP_SERVER`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `TEAM_EMAILS`

Or skip the workflow entirely: anyone can **Watch → Issues** on the repo for
GitHub's built-in email alerts.

---

## 🗂️ What's in here

```
.
├── index.html                     # The whole app (Pages frontend + auth + feed)
├── README.md                      # You are here
├── SECURITY.md                    # Security model & threat notes — read before deploying
├── LICENSE                        # MIT
├── docs/
│   ├── AUTHENTICATION.md          # ★ Deep dive: no-backend auth, with code
│   └── ARCHITECTURE.md            # How Pages + Issues + Actions fit together
└── .github/
    ├── workflows/notify.yml       # Action: notify on new post
    └── ISSUE_TEMPLATE/            # Post form + issue config
```

---

## 🧪 Run it locally

It's a static file, so any static server works:

```bash
python -m http.server 8080
# then open http://localhost:8080
```

Sign in with a token scoped to your repo and you'll be talking to the real
GitHub API from `localhost`.

---

## ⚠️ Before you deploy

- **Never commit a token.** Not even an "encrypted" or "obfuscated" one — a
  static repo is public to its readers. See [SECURITY.md](SECURITY.md).
- **Public repo = public posts.** Issues in a public repo are world-readable on
  GitHub regardless of this app. Use a **private** repo if posts should be
  members-only.
- Tokens are **per user** and scoped to **Issues on one repo** — that's the
  whole blast radius if one leaks.

---

## When to use this (and when not to)

**Great for:** internal team tools, knowledge hubs, changelogs, link boards,
small forums — where everyone already has a GitHub account.

**Not for:** anonymous/public writes, high-volume real-time apps, anything
needing server-side secrets or users without GitHub accounts. → use a real
backend.

---

## License

MIT — see [LICENSE](LICENSE). Use it however you like.
