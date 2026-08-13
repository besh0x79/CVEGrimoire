# CVEGrimoire

<p align="center">
  <a href="https://ibb.co/XfwxsGP1"><img src="https://i.ibb.co/DPqfCjFh/Chat-GPT-Image-Aug-13-2026-10-51-51-PM.png" alt="CVEGrimoire Poster" border="0"></a>
</p>

> **A grimoire of flaws uncovered through the hunt.**

I created **CVEGrimoire** to document the CVEs I come across while solving Hack The Box machines.

Whenever I find that a machine is vulnerable to a specific CVE, I take the time to research it, understand why it works, reproduce the vulnerability, and write my own notes and PoC when possible.

This repository is basically my **personal vulnerability grimoire** — a place where I keep the things I learn instead of letting them disappear after solving a machine.

---

## What I Document

For every CVE I add, I try to cover the parts that actually helped me understand it:

- **Discovery** — How I found the vulnerable software and version.
- **Research** — What I found about the CVE and its root cause.
- **Analysis** — Understanding what is actually happening behind the exploit.
- **Exploitation** — How the vulnerability can be triggered.
- **PoC** — My own implementation or recreation when possible.
- **Impact** — What the vulnerability allows an attacker to achieve.

---

## My Workflow

Most entries follow a simple path:

```text
Find
 ↓
Identify
 ↓
Research
 ↓
Understand
 ↓
Reproduce
 ↓
Exploit
 ↓
Document
```

I don't want this repository to be just a list of CVE numbers.

I want to remember **how I found them, how they work, and what I learned from them.**

---

## Structure

```text
CVEGrimoire/
│
├── CVE-XXXX-XXXX/
│   ├── README.md
│   └── exploit.py
│
├── CVE-XXXX-XXXX/
│   └── README.md
│
└── README.md
```

Each folder represents a vulnerability I've researched and documented.

---

## Disclaimer

Everything here is for **educational purposes and authorized security testing**.

The research is performed in controlled environments such as Hack The Box. Don't use these techniques against systems without explicit authorization.

---

**One machine. One vulnerability. One more page.**
