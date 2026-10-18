---
title: "UnitedCTF - Visitor Portal"
date: 2026-10-11 10:00:00 -0400
categories: [Writeups, Web]
tags: [unitedctf-2026, web, authorization, cookie, privilege-escalation, burp]
media_subpath: /assets/img/posts/visitor-portal/
toc: true
---

## Context

An employee portal that gives visitors a **guest** session; the internal area at
`/interne` is reserved for **admin** staff. The hint: the server decides your role
from something you fully control.

## Recon

The landing page says a guest has nothing interesting to see and links to the
internal area. The useful part is a comment left in the HTML source:

![HTML comment revealing the role cookie](vp1-source-comment.png)
_A source comment: `/interne` is reserved for the `admin` role, and the role is carried by the `role` session cookie._

So authorization is decided entirely by a **client-side cookie** named `role`. The
server trusts whatever value we send — the exact "something you fully control"
from the description.

## Exploitation

Our session starts as `role=invite`. Requesting `/interne` and intercepting it in
Burp, we change the cookie to `role=admin`:

![Burp intercept with role=admin](vp1-burp-cookie.png)
_Rewriting the session cookie to `role=admin` on the request to `/interne`._

The server accepts it, confirms administrator access, and declassifies the
document — the flag:

![internal area granting admin access and the flag](vp1-flag.png)
_"Acces administrateur confirme" — the internal document is revealed._

## Takeaway

Authorization must be decided server-side from a value the client can't forge (a
signed session, a server-stored role), never from a plain cookie the browser owner
can rewrite. Here the `role` cookie *is* the access control, so flipping
`invite` → `admin` is all it takes.
