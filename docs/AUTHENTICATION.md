# Authentication, in depth

This is the heart of the project: **how do you authenticate users in an app
that has no server?**

The trick is to lean on GitHub itself as the identity provider. Each user
already has a GitHub account. If they can prove they hold a valid token for it,
they're authenticated — no login server, no password database, no sessions on
your side.

---

## The flow

```
FIRST VISIT (one time)
┌──────────────────────────────────────────────────────────────────────┐
│ 1. User pastes a fine-grained Personal Access Token (PAT).             │
│ 2. App calls  GET https://api.github.com/user  with that token.        │
│       • 200 OK  → token is valid; response tells us who they are       │
│                   (login, name, avatar_url).                           │
│       • 401     → bad token, rejected.                                 │
│ 3. User chooses a PIN.                                                  │
│ 4. App encrypts the token using the PIN (PBKDF2 → AES-GCM) and stores   │
│    the ciphertext + public profile in localStorage on THIS device.     │
└──────────────────────────────────────────────────────────────────────┘

EVERY LATER VISIT
┌──────────────────────────────────────────────────────────────────────┐
│ 1. App sees a vault in localStorage → shows "Welcome back, @user".     │
│ 2. User types their PIN.                                               │
│ 3. App derives the key from the PIN and decrypts the token locally.    │
│       • wrong PIN → AES-GCM authentication fails → "Wrong PIN".         │
│ 4. App re-validates with GET /user (in case the token was revoked).    │
│ 5. Authenticated. The token is held in memory for API calls.           │
└──────────────────────────────────────────────────────────────────────┘
```

The token is entered **once per device**. After that, the PIN is the only thing
the user types.

---

## Why a token *is* authentication

A GitHub Personal Access Token is a bearer credential: whoever holds it can act
as that account, within the token's scopes. So a successful

```http
GET /user
Authorization: Bearer <token>
```

does two jobs at once:

1. **Authentication** — only the real account holder could have generated a
   valid token, so a 200 response proves identity.
2. **Profile** — the response body hands us the username and avatar for free.

We don't need to store *who* the user is on a server. GitHub already knows, and
the token lets us ask.

---

## Why we don't just keep the token in `sessionStorage`

The simplest version of this pattern asks for the token on every visit and
keeps it in `sessionStorage` (gone when the tab closes). That's secure but
annoying — users paste a 90-character token constantly.

So we trade a little convenience for a small, well-contained piece of crypto:
store the token **encrypted**, unlock it with a short PIN. The key never exists
at rest; it's re-derived from the PIN each time.

---

## The crypto, line by line

All of this uses the browser-native [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API)
— no libraries. From `index.html`:

**1. Turn the PIN into a strong key (PBKDF2).** A 4-digit PIN has almost no
entropy on its own, so we stretch it with 250,000 iterations of PBKDF2 over a
random 16-byte salt:

```js
async function deriveKey(pin, salt) {
  const baseKey = await crypto.subtle.importKey(
    "raw", enc.encode(pin), "PBKDF2", false, ["deriveKey"]
  );
  return crypto.subtle.deriveKey(
    { name: "PBKDF2", salt, iterations: 250000, hash: "SHA-256" },
    baseKey, { name: "AES-GCM", length: 256 }, false, ["encrypt", "decrypt"]
  );
}
```

**2. Encrypt the token (AES-GCM).** A fresh random salt + IV per token; we store
all three pieces:

```js
async function encryptToken(token, pin) {
  const salt = crypto.getRandomValues(new Uint8Array(16));
  const iv   = crypto.getRandomValues(new Uint8Array(12));
  const key  = await deriveKey(pin, salt);
  const ct   = await crypto.subtle.encrypt({ name: "AES-GCM", iv }, key, enc.encode(token));
  return { salt: toB64(salt), iv: toB64(iv), ct: toB64(ct) };
}
```

**3. Decrypt — and verify the PIN at the same time.** AES-GCM is *authenticated*
encryption. If the derived key is wrong (wrong PIN), `decrypt` doesn't return
garbage — it **throws**. That's our PIN check, for free:

```js
async function decryptToken(blob, pin) {
  const key = await deriveKey(pin, fromB64(blob.salt));
  const plain = await crypto.subtle.decrypt(
    { name: "AES-GCM", iv: fromB64(blob.iv) }, key, fromB64(blob.ct)
  );
  return dec.decode(plain);   // throws on a wrong PIN
}
```

### What's stored in `localStorage`

```jsonc
{
  "login": "octocat",                  // plaintext — to greet the user pre-unlock
  "name": "The Octocat",
  "avatar_url": "https://…",
  "enc": {                             // the token, encrypted
    "salt": "…base64…",
    "iv":   "…base64…",
    "ct":   "…base64…"                 // <-- the actual token lives here, encrypted
  },
  "createdAt": "2026-06-29T…Z"
}
```

The raw token appears **nowhere** at rest. Only inside `enc.ct`, encrypted.

---

## Comparison: obfuscation vs. encryption

A tempting shortcut is "obfuscation" like `btoa(token.split('').reverse().join(''))`.
**Don't.** That's reversible by anyone in one line — it's encoding, not
security. The difference:

| | Obfuscation (`btoa`/reverse) | Encryption (this project) |
|---|---|---|
| Needs a secret to reverse? | No | Yes (the PIN) |
| Safe to expose publicly? | No | The ciphertext alone, yes |
| Protects a lost device? | No | Yes (PIN required) |

And critically: **encrypted or not, the token still must never be committed to
the repo.** Encryption protects the at-rest copy on the user's own device; it
does not make a repo a safe place for secrets. See [SECURITY.md](../SECURITY.md).

---

## Frequently asked

**Is a 4-digit PIN too weak?**
For *remote* attacks, there's nothing to attack — the ciphertext isn't
published anywhere. An attacker would need your unlocked device *and* the
ciphertext *and* the PIN. If you want more, raise `CONFIG.PIN_LENGTH` or accept
an alphanumeric passphrase; the code doesn't care.

**What if the token expires or is revoked?**
The re-validation `GET /user` on unlock fails, and the app prompts the user to
re-link with a fresh token.

**Can one browser hold several accounts?**
This template stores a single vault for simplicity. Supporting multiple is a
small change: key the vault by username and add an account picker.

**Why fine-grained tokens?**
They can be scoped to a single repo and a single permission (Issues), so a leak
is contained. Classic `ghp_` tokens work too but are broader — prefer
fine-grained.
