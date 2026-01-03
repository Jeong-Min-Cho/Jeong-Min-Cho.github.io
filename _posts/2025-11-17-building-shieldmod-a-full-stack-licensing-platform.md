---
layout: post
title: "Building ShieldMod: A Full-Stack Licensing Platform (And Still Going)"
date: 2025-11-17 10:00 +0900
categories: [Programming, Full-Stack]
tags: [ShieldMod, SaaS, FastAPI, Next.js, TypeScript, Python, C++, Architecture]
image:
  path: /assets/img/posts/shieldmod/system-architecture.svg
  alt: ShieldMod System Architecture
---

## The Beginning of a 565,528 Line Journey

It's been almost 2 months since I launched ShieldMod in early November - a licensing distribution system for game modding developers. The project is still actively evolving, and before I dive into the technical horror stories (yes, there will be many), let me set the stage for what this project involved.

![This is Fine](/assets/img/posts/shieldmod/post-1.png)
*Me, starting this project with "general ideas to implement"*

---

## Situation: The Problem I Set Out to Solve

Software distribution with proper licensing is a complex problem. Existing solutions were either:
- **Too expensive** for indie mod developers
- **Too basic** without proper hardware binding
- **Too insecure** with easily bypassable protections
- **Too ugly** with UX from the early 2000s

I needed a system that could:
1. **Authenticate users** across multiple platforms (web browsers and desktop applications)
2. **Manage licenses** with activation limits, expiration dates, and usage tracking
3. **Distribute files securely** so only authorized users can access them
4. **Track devices** to prevent license sharing while allowing legitimate multi-device use
5. **Monitor everything** with comprehensive audit logs for security and debugging

---

## Task: What I Needed to Build

From the start, I had four core principles:

1. **Secure** - Because licensing systems are literally attack magnets
2. **Modern** - No PHP spaghetti code here
3. **User-friendly** - Even my non-technical users should get it
4. **Easy to Use** - For developers integrating the system

The platform would need four interconnected components:

<div style="width: 100%; max-width: 900px; margin: 2rem auto;">
  <img src="/assets/img/posts/shieldmod/system-architecture.svg" alt="System Architecture" style="width: 100%; height: auto;">
</div>

---

## Action: How I Built It

### The Tech Stack

#### Frontend: The Pretty Face
- **Next.js 14** (React framework with App Router)
- **TypeScript 5** (because `any` is not a type, it's a cry for help)
- **TanStack Query v5** (server state management that doesn't make you cry)
- **TanStack Table v8** (data table handling)
- **Tailwind CSS 3.4** (utility-first or die)
- **shadcn/ui** (beautiful Radix UI components)
- **React Hook Form + Zod** (form validation)
- **Framer Motion** (smooth animations)
- **i18next** (Korean/English internationalization)

#### Backend: The Brain
- **Python 3.11+** with **FastAPI 0.109** (async all the things!)
- **PostgreSQL 15** (the only database that matters)
- **Redis 7** (session storage, caching, rate limiting)
- **SQLAlchemy 2.0** (async ORM)
- **Alembic** (database migrations)
- **JWT (python-jose)** (token-based auth)
- **Argon2id / bcrypt** (password hashing)
- **AES-256-GCM** (file encryption)
- **Docker Compose** (containerization)

#### Desktop Client & Module: The Muscle
- **C++17/C++20** (because performance matters)
- **DirectX 11 + ImGui** (modern dark-themed GUI)
- **WinHTTP** (HTTPS communication)
- **ECDSA P-256** (signature verification)
- **HKDF-SHA256** (key derivation)
- **Lua C API** (scripting engine)

### Key Technical Implementations

#### 1. Dual Authentication System

Supporting both web and desktop clients required two authentication methods simultaneously:

**Web Dashboard (Session-based):**
```
Browser → POST /auth/login → Set HTTP-only cookie → Done
```

**Desktop Client (JWT-based):**
```
Client → POST /auth/client/login (with HWID) → JWT tokens → Periodic refresh
```

The JWT system uses short-lived access tokens (15 minutes) and longer refresh tokens (7 days), with proactive refresh 2 minutes before expiry.

#### 2. Secure File Distribution

Files are encrypted server-side with **AES-256-GCM**, and each user gets a unique decryption key derived via **HKDF**:

<div style="width: 100%; max-width: 800px; margin: 2rem auto;">
  <img src="/assets/img/posts/shieldmod/file-encryption-flow.svg" alt="File Encryption Flow" style="width: 100%; height: auto;">
</div>

This means:
- Each user gets a unique decryption key
- Keys are derived, not stored (reducing attack surface)
- Stolen files are useless without valid credentials
- Plaintext **never touches the disk**

#### 3. Multi-Device Management

Users can register multiple devices (up to a configurable limit), each tracked independently with:
- HWID binding (CPU ID, Motherboard Serial, BIOS Serial - NOT MAC addresses!)
- Device nicknames ("Gaming PC", "Laptop", etc.)
- First seen / last seen timestamps
- Monthly removal quotas

#### 4. Security: Defense in Depth

<div style="width: 100%; max-width: 800px; margin: 2rem auto;">
  <img src="/assets/img/posts/shieldmod/security-architecture.svg" alt="Security Architecture" style="width: 100%; height: auto;">
</div>

---

## Result: The Numbers Don't Lie

Starting in early October 2025, launching in November, and still actively developing:

| Metric | Value |
|--------|-------|
| **Total Lines of Code** | 565,528 lines |
| **Total Commits** | 2,721 commits across 4 repos |
| **Bug Fix Commits** | 1,060 fix commits (39% of total!) |
| **Database Migrations** | 25 Alembic migrations |

Yes, you read that right. **39% of my commits were bug fixes.**

![Two Buttons Sweating](/assets/img/posts/shieldmod/post-2.jpg)
*My daily struggle: "Ship the feature" vs "Fix that edge case first"*

### Codebase Breakdown

| Component | Language | Lines of Code | Commits | Fix Commits |
|-----------|----------|---------------|---------|-------------|
| Backend API | Python | 49,627 | 256 | 86 |
| Admin Dashboard | TypeScript/React | 166,901 | 331 | 136 |
| Desktop Client | C++ | 214,701 | 267 | 83 |
| Client Module | C++ | 134,299 | 1,867 | 755 |

That Client Module with 755 fix commits? That's where the *real* fun happened. More on that in future posts.

### Component Statistics

| Category | Count |
|----------|-------|
| API Routers | 31 endpoint modules |
| Database Models | 32 SQLAlchemy models |
| React Components | 129 components |
| Custom React Hooks | 30 hooks |
| TypeScript Type Files | 22 type definitions |
| Pages (Dashboard) | 22 pages |
| Pages (User Portal) | 20 pages |

### Key Features Delivered

**Admin Dashboard:**
- Real-time statistics (users, sessions, licenses)
- Registration trend charts (30-day)
- License expiration analytics
- User management with ban/unban + HWID reset
- Device comparison analytics
- Comprehensive audit logs with filtering and export

**User Portal:**
- Personal dashboard with usage stats
- Device management (add/remove with quotas)
- License information display
- Session management
- Telegram integration setup

**Desktop Client:**
- Modern ImGui interface with DirectX 11
- Secure credential storage via Windows Credential Manager
- In-memory file decryption
- Automatic token refresh
- Multi-device support

**Client Module:**
- Lua scripting for extensibility
- Heartbeat license validation with grace periods
- Crash recovery with comprehensive logging
- Telegram notifications

---

## The Development Timeline

```
October 2025 ──────────────────────────────────────────────────────
    ├── Week 1: "I have general ideas" phase
    ├── Week 2: Backend API foundation + Database design
    ├── Week 3: Frontend dashboard MVP + Desktop client starts
    └── Week 4: Integration hell + The Great Debugging Period™

November 2025 ─────────────────────────────────────────────────────
    ├── Week 1: Launch! 🚀
    ├── Week 2-3: Bug fixes from real users (the fun begins)
    └── Week 4: Multi-device support feature

December 2025 ─────────────────────────────────────────────────────
    ├── Ongoing improvements and new features
    └── More debugging (it never ends)

January 2026 ──────────────────────────────────────────────────────
    └── Still going strong... 💪
```

---

## What's Coming Next

Over the next few posts, I'll be sharing the most painful, educational, and sometimes hilarious bugs I encountered. Here's a preview:

1. **The Infinite Redirect Loop Nightmare** - How logout became an endless dance between `/login` and `/dashboard`

2. **HWID Instability: When MAC Addresses Betray You** - 47 support tickets in one week because I trusted network adapters

3. **Race Conditions in Token Refresh** - When 5 requests all decide to refresh the token at the same time

4. **The 307 Redirect That Broke Everything** - FastAPI's helpful default that destroyed all PUT requests

5. **ECDSA Signature Format Mismatch** - 6 hours of debugging because Python and C++ disagree on what a signature looks like

Each post will include:
- The actual problem (with real logs and error messages)
- The investigation process
- The solution (with working code)
- Memes (because debugging without humor is just suffering)

---

## Lessons Learned (Spoiler Alert)

If I had to summarize these months of development:

> **"Of course I didn't plan everything since the beginning, but I have general ideas to implement"**
>
> — Me, underestimating everything

![Distracted Boyfriend](https://i.imgflip.com/1ur9b0.jpg)
*Me looking at new frameworks while my bugs pile up*

The truth is, **starting is not that easy**. You can have the best tech stack, the cleanest architecture diagrams, and the most detailed specifications - but the moment you write `git init`, reality starts fighting back.

**Key Takeaways:**
1. **Security is an architecture** — It's not something you add at the end; it must be designed in from the start.
2. **Async is essential** — For any system handling concurrent users, blocking I/O is a bottleneck.
3. **Type safety saves time** — The hours spent defining types are repaid in fewer bugs and easier refactoring.
4. **Cross-platform is hard** — Web and desktop have fundamentally different models; abstracting the differences requires careful design.
5. **Monitoring is not optional** — Without comprehensive logs, debugging production issues is nearly impossible.

Stay tuned for the next post where I'll share how a simple logout button caused 200+ API requests in under 5 seconds and crashed browser tabs across the globe.

---

*This is part 1 of my "Building ShieldMod" series. Subscribe to follow along as I share the triumphs and tragedies of building a full-stack SaaS platform from scratch.*
