# Task Radar: overview

> **Source code is private.** This repository describes the system's logic and architecture.
>
> 🕶️ *"Task Radar" is a codename. The platform it watches stays unnamed, the same way
> you don't tell the fish which lake you fish in.*

**A monitor that watches an online work platform for newly published paid tasks and alerts within
seconds, because the best-paid tasks are often taken within minutes. Built to run unattended for weeks.**

![Python](https://img.shields.io/badge/Python-3-3776AB)
![Playwright](https://img.shields.io/badge/Playwright-headless-2EAD33)
![systemd](https://img.shields.io/badge/systemd-user%20units-informational)
![Telegram](https://img.shields.io/badge/Telegram-alerts-26A5E4)

## What makes it hard

Polling a page is easy. Staying logged in, fast and alive for weeks without anyone touching it is not.

## How it works

**Fast detection**
- The platform renders its task list server-side and embeds the data in the page, so the monitor reads it
  over **plain HTTP with session cookies** (~0.7 s) instead of driving a headless browser (3–5 s)
- That allows a short polling interval with jitter
- **Automatic fallback** to a headless browser if the HTTP response ever stops matching the expected
  shape (markup change, block, network)
- Correctly tells "new task available" from "task already taken" (redirect patterns, banners), and
  reports the pay rate in every alert

**Self-renewing session**
- Sessions expire server-side after a fixed lifetime. A timer re-logs in shortly before expiry, at a
  **deterministic** random offset derived from the login timestamp, so the target doesn't drift on every check
- Login runs headless; the **2FA code is fetched from the mailbox over IMAP**, then the form is completed
  with no human in the loop
- Login time is tracked separately from cookie-file writes, because the monitor re-syncs cookies
  continuously and would otherwise hide the session's real age
- On failure: screenshot, Telegram alert, documented manual path

**High availability**
- Primary runs on a home Linux server as a `systemd --user` service with restart policy
- A **hot standby** on a second machine checks the primary's heartbeat every minute; after 5 minutes of
  silence it takes over, and it hands back automatically when the primary recovers, announcing both
  transitions in Telegram

## Architecture

```
 primary (Linux, systemd) ── HTTP poll ──► platform
     │  heartbeat (state file)                 ▲
     │                                         │ headless fallback / re-login
 standby (Windows) ── watches heartbeat ── takes over on silence
     │
 Telegram alerts · IMAP 2FA · secrets kept only on the host, never in git
```

## My role

Designed and built it, and operate it. Main engineering themes: reverse-engineering server-rendered
data, session lifecycle management, and failover.
