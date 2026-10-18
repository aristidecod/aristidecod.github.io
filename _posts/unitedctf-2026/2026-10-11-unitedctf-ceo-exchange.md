---
title: "UnitedCTF - CEO Exchange"
date: 2026-10-11 12:00:00 -0400
categories: [Writeups, Web]
tags: [unitedctf-2026, web, sql-truncation, mysql, collation, latin1_swedish_ci, xs-leak, ssrf, csrf, csp]
media_subpath: /assets/img/posts/ceo-exchange/
toc: true
---

## Context

CEO Exchange is a Node.js / Express note-sharing app, themed as a secret
communication platform used by corporate "entities". The goal is a
privilege-escalation chain: get onto the platform with a low-privilege account,
then climb to the account that actually holds the secrets.

It's a **white-box** challenge — we're handed `ceo-exchange.zip` with the full
source, so the approach is to read the code rather than probe blindly. The series
has three flags:

- **CEO-Exchange-1** — break into the pre-seeded **ceo** trial account.
- **CEO-Exchange-2** — read the **admin**'s private note.
- **CEO-Exchange-3** — gain full access to the server.

## Reconnaissance

### The stack

With the source in hand, the first stop is the `docker-compose.yml`:

![docker-compose showing mysql:5.6](f1-docker-mysql56.png)
_The database is `mysql:5.6`._

The notable detail is the database version: **`mysql:5.6`**, an end-of-life
release that runs in non-strict SQL mode by default. That default is what flag 1
relies on.

Three users exist at boot: `admin` (holds FLAG2 as a note), `ceo` (a trial
account that sees FLAG1), and whatever we register ourselves.

### The registration logic

Everything starts at the `/request` (registration) endpoint: a keyword blocklist,
a uniqueness check, then the INSERT.

![/request handler source](f1-request-handler.png)
_The `/request` handler._

## CEO-Exchange-1 — SQL truncation

### The two bugs that combine

Looking at the table schema:

![users table schema with VARCHAR(50)](f1-schema-varchar50.png)
_Every text column is `VARCHAR(50)`, including `username` and `password`._

Two MySQL behaviours matter here:

1. **Silent truncation.** On `mysql:5.6` the default SQL mode is *not* strict.
   Inserting a string longer than `VARCHAR(50)` does **not** error — MySQL cuts it
   to 50 characters and emits a warning that the app never checks.
2. **`PAD SPACE` comparison.** For `CHAR`/`VARCHAR`, MySQL ignores **trailing
   spaces** when comparing. So `'ceo'` and `'ceo      '` are considered **equal**.

The `ceo` user already exists with a random password we don't know, so we can't
log into it directly. But FLAG1 is shown to anyone whose session username is
`ceo`. If we can create a *second* row that MySQL treats as `ceo` but with a
password **we** choose, we can authenticate as it.

### The payload

Register with the username set to `ceo`, padded with spaces, then a trailing
non-space character:

```
username = "ceo" + (47 spaces) + "x"      (51 characters total)
email    = ceo                            (any value we don't own yet)
password = (our own password)
```

![registration form with padded username](f1-register-truncation.png)
_The username is `ceo`, a long run of spaces, then `x` on the far right (51 chars)._

### Logging in

Now we log in as `ceo` with the password we just set: MySQL matches our padded row
(trailing spaces ignored), the password narrows it to our row, and the session
stores the username straight from the request body — so we're `ceo`.

The notes template only prints the flag inside a `user.username === "ceo"` block:

![EJS template showing the ceo-only flag](f1-flag-display.png)
_FLAG1 is rendered only when the session user is `ceo`._

Our padded account satisfies that check, and **FLAG1** appears:

![notes page as the ceo account showing the flag](f1-flag-ceo.png)
_Logged in as the ceo trial account — the flag sits next to the welcome line._

## CEO-Exchange-2 — collation mismatch

FLAG2 is seeded as a note owned by `admin`:

![notes seeding code](f2-notes-seed.png)
_FLAG2 is inserted as a note belonging to `admin`._

The notes page only returns notes whose `username` matches the logged-in session,
so reading it means getting our session seen as `admin`. Two obstacles stand in
the way — and each falls to a string-equality trick.

### Dodging the blocklist — `à` vs `a`

The blocklist rejects any username containing `adm`, but it's a byte-exact
JavaScript `includes()`. Replacing the `a` with `à` (grave accent) gives `àdmin`,
which slips past — yet MySQL's default `latin1_swedish_ci` collation folds `à`
onto `a`, so it still compares equal to `admin`. (`á â ã` work too; `ä` and `å`
don't — they're separate letters in Swedish.)

### Dodging the uniqueness check — truncation again

The collation equality cuts both ways: `àdmin` alone would collide with the
existing `admin` at the uniqueness check and be rejected. So we reuse flag 1's
**truncation trick** — pad `àdmin` with spaces and a trailing `x` so the full
string slips past the check, then gets truncated back down to `àdmin` on insert:

```
username = "àdmin" + (spaces) + "x"   (> 50 characters)
email    = hello@hello.com            (unused, must be unique)
password = (our own password)
```

![registration with àdmin padded](f2-register-admin.png)
_`àdmin`, padding, then `x` — accent bypasses the blocklist, length bypasses the uniqueness check._

### Logging in as admin

We log in with username `admin` (plain) and our password:

![login form with admin](f2-login-admin.png)
_Authenticating as `admin` with the password we chose._

The query `WHERE username = 'admin' AND password = ?` matches our stored row —
`'admin'` equals `'àdmin␣␣␣'` through the collation (`à`=`a`) and `PAD SPACE`
(trailing spaces ignored) — while the password narrows it to **our** row, not the
real admin's. The session username is copied from the request body, so it's a
plain `admin`.

The notes query then runs with `username = 'admin'` and returns the real admin's
seeded note — **FLAG2**:

![admin notes page with the confidential flag](f2-flag-admin.png)
_Logged in as admin: the confidential note (FLAG2) and the admin-only ad upload panel._

## Bonus — the intended path: SSRF + XS-Leak

The collation trick above is the quick, fully internal solution. There's also a
more involved path through the app's ad-report bot — worth documenting even though
I only got it working locally.

### The ad-report bot

The ceo account can "report an ad" to an admin. The `/visit` handler spins up a
headless browser, logs in as admin, and navigates to the URL we submit:

![bot logging in as admin then visiting the submitted URL](bonus-visit-goto.png)
_The bot logs in as `admin`, then navigates to our URL in that same session._

That gives us an admin-authenticated browser pointed at a URL of our choice. We
can't just inject a script into the site — a strict CSP blocks inline scripts:

![Content-Security-Policy response header](bonus-csp.png)
_`default-src 'self'` — no inline scripts, no third-party origins._

But if we send the bot to a page **we** host, our own JavaScript runs in that
admin session anyway.

### Bypassing the URL filter — `//`

The handler only accepts a URL starting with `/`:

![startsWith('/') filter on the report URL](bonus-visit-filter.png)
_The report URL must start with `/`, then it's resolved against `http://localhost`._

A protocol-relative URL clears both hurdles: `//our-domain/` starts with `/`, and
`new URL('//our-domain/', 'http://localhost')` resolves to `http://our-domain/`.
The bot leaves localhost and hits our server.

### The oracle — `frames.length`

We still can't *read* the admin's notes: the Same-Origin Policy blocks our page
from reading anything cross-origin. But one property survives cross-origin —
`window.length`, the number of sub-frames in a window.

Our page opens the admin's notes filtered on a guess with `window.open` (a popup,
so the admin's `SameSite=Lax` cookie rides along):
`http://localhost/notes?filter=GUESS`. The notes page's `main.js` injects the ad
iframe **only when at least one note survives the filter**:

![main.js creating the ad iframe when a note matches](bonus-oracle-iframe.png)
_A matching filter leaves a note visible, which triggers the ad iframe._

So `window.length` on that popup is our one-bit oracle: `1` means the guess is a
substring of the admin's note, `0` means it isn't. Feed it `FLAG-`, then
`FLAG-0`…`FLAG-F`, and rebuild the flag one character at a time.

### Local yes, remote no

Against a local instance the oracle works cleanly. Against the real remote
challenge it never did: the popup opened and navigated to `localhost`, but
`window.length` was always `0`. The admin session cookie wasn't travelling with a
request fired from our externally-hosted (public) page to `localhost` — the
browser blocks that, so the bot always hit the unauthenticated redirect and no ad
iframe was ever created.

### Tunnels

For reference — reaching the bot requires exposing the local server (here on port
8000) through a tunnel. Three tools, and how each behaved with the headless bot:

| Tool | Command | Note |
|------|---------|------|
| ngrok | `ngrok http 8000` | HTTPS, but a "Visit Site" interstitial blocks the headless bot |
| bore | `bore local 8000 --to bore.pub` | Plain HTTP, no interstitial, but the bot never reached it |
| cloudflared | `cloudflared tunnel --url http://localhost:8000` | HTTPS, reached the bot reliably |

## CEO-Exchange-3 — full server access

_Coming soon._

## Takeaways

- **Never trust that your validation layer and your storage layer agree.** Flag 1
  is a validation-vs-storage gap (full string checked, truncated string stored);
  flag 2 is a comparison gap (byte-exact JS vs collation-aware SQL).
- On MySQL, enable **`STRICT_ALL_TABLES`** so over-long inserts error instead of
  truncating, and do uniqueness checks against the value *as it will be stored*.
