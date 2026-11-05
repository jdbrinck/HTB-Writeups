# HTB Writeups

Walkthroughs and notes from my work on [Hack The Box](https://www.hackthebox.com/) — machines, challenges, and Academy modules. I keep these as a record of how I approach a target: enumeration, the thinking behind each step, the dead ends, and the reusable patterns I take away.

> _About me: <one or two lines — who you are, what you're focused on (e.g. web app security, AD, etc.), and what you're looking for>._

---

## Index

### Academy

| Module / Assessment | Topic | Writeup |
|---|---|---|
| SSRF — Final Assessment | SSRF -> SSTI -> RCE (Twig) | [TruckAPI](academy/ssrf-final-assessment.md) |

### Machines

| Box | Difficulty | OS | Writeup |
|---|---|---|---|
| _(retired boxes go here)_ | | | |

### Challenges

| Challenge | Category | Writeup |
|---|---|---|
| _(challenge writeups go here)_ | | |

---

## Skills demonstrated

As this grows, this section is a quick map of what's covered — handy for anyone skimming:

- **Web** — SSRF, SSTI, template-engine RCE
- _(add as you go: injection, AD, privesc, reversing, crypto, ...)_

## Structure

```
htb-writeups/
├── templates/writeup-template.md   # copy this to start a new writeup
├── academy/                        # HTB Academy module / assessment notes
├── machines/                       # retired-machine walkthroughs
├── challenges/                     # challenge walkthroughs
└── assets/                         # screenshots / diagrams
```

## A note on scope

> [!IMPORTANT]
> Everything published here is for **retired** machines, challenges, or Academy content. Writeups for active HTB machines are intentionally left out, in line with HTB's rules.

Each writeup aims to show *method*, not just answers — the goal is to demonstrate how I work a problem, not to spoil live content.
