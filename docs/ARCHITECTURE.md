# Architecture

A full, multi-user web app with **no servers, no database, and no hosting
bill** — built only from features every GitHub repo already has.

```
                         ┌───────────────────────────┐
                         │        GitHub Pages        │
        visits  ───────► │   serves index.html (the   │
        browser          │   whole app, static file)  │
                         └─────────────┬─────────────┘
                                       │ runs in the user's browser
                                       │ (their own PAT as credential)
                                       ▼
                         ┌───────────────────────────┐
            read posts ◄─┤        GitHub REST API      ├─► create post
            react        │      api.github.com         │   (new issue)
                         └─────────────┬─────────────┘
                                       │ each post = one Issue
                                       ▼
                         ┌───────────────────────────┐
                         │        GitHub Issues        │  ← the "database"
                         │   label `post` = a record   │
                         └─────────────┬─────────────┘
                                       │ on: issues opened
                                       ▼
                         ┌───────────────────────────┐
                         │       GitHub Actions        │  → Slack / Discord /
                         │      .github/workflows      │     email / log issue
                         └───────────────────────────┘
```

## The four pieces

### 1. GitHub Pages — the frontend
A single static `index.html` is served from your repo at
`https://<owner>.github.io/<repo>/`. No build step, no framework. Because it's
just a static file, hosting is free and there's no server to maintain or
secure.

### 2. GitHub Issues — the database
Each post is one Issue carrying the `post` label.

| App concept | GitHub primitive |
|---|---|
| A post | An Issue |
| Post title / body | Issue title / body |
| Author + timestamp | Issue `user` + `created_at` (set by GitHub) |
| Tags / categories | Extra labels on the issue |
| Upvotes | Issue 👍 reactions |
| Comments (if you add them) | Issue comments |
| "Archive a post" | Close the issue |

You get search, edit history, permalinks, an API, and a moderation UI (the
Issues tab) for free — none of which you had to build.

### 3. GitHub REST API — the data layer
The browser talks to `api.github.com` directly using the signed-in user's
token. Key calls used by `index.html`:

| Action | Request |
|---|---|
| Who is this user? | `GET /user` |
| List posts | `GET /repos/{owner}/{repo}/issues?labels=post&state=open` |
| Create a post | `POST /repos/{owner}/{repo}/issues` |
| Upvote | `POST /repos/{owner}/{repo}/issues/{n}/reactions` |

### 4. GitHub Actions — automation
`.github/workflows/notify.yml` triggers whenever an issue with the `post` label
is opened, and pushes a notification (Slack/Discord/email, or a log comment on a
pinned tracking issue). It runs on GitHub's runners with the built-in
`GITHUB_TOKEN` — no PAT and no secrets required for the default behavior.

## Data flow: posting something

1. User writes a title/body in the app and clicks **Post**.
2. Browser sends `POST /repos/.../issues` with their token → GitHub creates the
   Issue.
3. The new issue appears instantly in the feed (optimistic update) and is now a
   permanent record in the Issues tab.
4. GitHub fires the `issues: opened` event → the Action runs → the team gets
   notified.

## What you trade away

This stack is ideal for internal tools, team knowledge hubs, changelogs, link
boards, lightweight forums — anything where "every user has a GitHub account"
is true and traffic is human-scale.

It is **not** a fit when you need: anonymous/public writes, sub-second writes at
high volume, server-side secrets or business logic, or users who don't (and
shouldn't need to) have GitHub accounts. For those, you need a real backend.

See [AUTHENTICATION.md](AUTHENTICATION.md) for how sign-in works, and
[../SECURITY.md](../SECURITY.md) for the security model.
