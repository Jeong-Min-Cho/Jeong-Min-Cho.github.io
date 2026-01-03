---
layout: post
title: "Building ShieldMod: A Full-Stack Licensing Platform (And Still Going)"
date: 2025-12-01 10:00 +0900
categories: [Programming, Full-Stack]
tags: [ShieldMod, SaaS, FastAPI, Next.js, TypeScript, Python, C++, Architecture]
---

## The Beginning of a 565,528 Line Journey

It's been almost 2 months since I launched ShieldMod in early November - a licensing distribution system for game modding developers. The project is still actively evolving, and before I dive into the technical horror stories (yes, there will be many), let me set the stage for what this project involved.

![This is Fine](/assets/img/posts/shieldmod/post-1.png)
*Me, starting this project with "general ideas to implement"*

## What I Wanted to Build

From the start, I had four core principles:

1. **Secure** - Because licensing systems are literally attack magnets
2. **Modern** - No PHP spaghetti code here
3. **User-friendly** - Even my non-technical users should get it
4. **Easy to Use** - For developers integrating the system

Simple goals, right?

## The Tech Stack

### Frontend: The Pretty Face
- **Next.js 14** (React framework)
- **TypeScript** (because `any` is not a type, it's a cry for help)
- **TanStack Query** (state management that doesn't make you cry)
- **Tailwind CSS** (utility-first or die)
- **shadcn/ui** (beautiful components)
- **Framer Motion** (smooth animations)

### Backend: The Brain
- **Python 3.11+** with **FastAPI 0.109** (async all the things!)
- **PostgreSQL 15** (the only database that matters)
- **Redis 7** (session storage, caching, rate limiting)
- **SQLAlchemy 2.0** (async ORM)
- **JWT** (token-based auth)
- **AES-256-GCM** (file encryption)
- **Docker Compose** (containerization)

### Desktop Client & Module: The Muscle
- **C++** (because performance matters when you're hooking into games)

## The Numbers Don't Lie

Starting in early October 2025, launching in November, and still actively developing - here's what the codebase looks like so far:

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
| API Routers | 28 endpoint modules |
| Database Models | 32 SQLAlchemy models |
| React Components | 129 components |
| Custom React Hooks | 30 hooks |
| TypeScript Type Files | 22 type definitions |

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

## Lessons Learned (Spoiler Alert)

If I had to summarize these months of development:

> **"Of course I didn't plan everything since the beginning, but I have general ideas to implement"**
>
> — Me, underestimating everything

![Distracted Boyfriend](https://i.imgflip.com/1ur9b0.jpg)
*Me looking at new frameworks while my bugs pile up*

The truth is, **starting is not that easy**. You can have the best tech stack, the cleanest architecture diagrams, and the most detailed specifications - but the moment you write `git init`, reality starts fighting back.

Stay tuned for the next post where I'll share how a simple logout button caused 200+ API requests in under 5 seconds and crashed browser tabs across the globe.

---

*This is part 1 of my "Building ShieldMod" series. Subscribe to follow along as I share the triumphs and tragedies of building a full-stack SaaS platform from scratch.*
