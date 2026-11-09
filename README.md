# HTB Writeups

Walkthroughs and notes from my work on [Hack The Box](https://www.hackthebox.com/) — machines, challenges, and Academy modules. These document how I approach a target: enumeration, the reasoning behind each step, the dead ends, and the reusable takeaways.

> 🔗 **My HTB profile:** https://profile.hackthebox.com/profile/019d98f7-df29-7381-a60c-4159ff40b319

> _Cybersecurity professional with dual degrees in Cybersecurity and Software Engineering. I use Hack The Box to sharpen my offensive security skills — these writeups document the methodology behind each solve, not just the flag. Open to opportunities in offensive/application security._

## Index

### Academy
| Module / Assessment | Topic | Writeup |
|---|---|---|
| SSRF — Final Assessment | SSRF -> SSTI -> RCE (Twig) | [TruckAPI](academy/ssrf-final-assessment.md) |
| File Upload Attacks — Skills Assessment | Upload filter bypass -> XXE -> RCE | [File Upload Chain](academy/file-upload-skills-assessment.md) |
| Command Injections — Skills Assessment | Filter-bypass OS command injection | [Command Injection](academy/command-injection-skills-assessment.md) |
| Cross-Site Scripting (XSS) — Skills Assessment | Blind XSS -> admin cookie theft | [Blind XSS](academy/xss-skills-assessment.md) |
| Login Brute Forcing — Skills Assessment | Basic-auth -> SSH -> pivot -> FTP (Hydra) | [Brute Force Chain](academy/brute-force-skills-assessment.md) |

### Machines
| Box | Difficulty | OS | Writeup |
|---|---|---|---|
| _(retired boxes go here)_ | | | |

### Challenges
| Challenge | Category | Writeup |
|---|---|---|
| _(challenge writeups go here)_ | | |

## Skills demonstrated
- **Web** — SSRF, SSTI, template-engine RCE, file-upload bypass, XXE, command injection, XSS (blind / session hijacking)
- **Access** — credential brute forcing (Hydra), username enumeration, SSH/FTP, loopback pivoting
- _(add as you go: AD, privesc, reversing, crypto, ...)_

## Scope
> [!IMPORTANT]
> Only **retired** machines, challenges, and Academy content are published here — never active machines, in line with HTB's rules. The focus is method, not spoilers.
