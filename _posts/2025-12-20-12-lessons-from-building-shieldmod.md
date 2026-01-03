---
layout: post
title: "12 Lessons from Building a Full-Stack Platform (With Memes)"
date: 2025-12-20 10:00 +0900
categories: [Programming, Lessons Learned]
tags: [ShieldMod, Full-Stack, Architecture, Debugging, Best Practices, Career]
---

## 565,528 Lines. 2,721 Commits. 1,060 Bug Fixes. Here's What I Learned.

After months of building ShieldMod (started October, launched November, still going strong into 2026) - a licensing platform spanning Python, TypeScript, C++, and my sanity - I've compiled the 12 biggest lessons. Each one came with its own special flavor of pain.

![Astronaut Meme](https://i.imgflip.com/46e43q.png)
*"Wait, it's all debugging?" "Always has been."*

---

## Lesson 1: Authentication State is Distributed

**The Bug:** Users got stuck in infinite redirect loops after logging out.

**The Lesson:** Your user's "logged in" state exists in at least 4 places:
- HTTP-only cookies
- React state
- React Query cache
- LocalStorage

When logging out, you must clear ALL of them atomically, in the right order.

```typescript
// Clear EVERYTHING before redirecting
queryClient.clear();
localStorage.clear();
deleteCookies(['session_id', 'token']);
window.location.href = '/login?expired=true'; // Full reload, not router.push
```

![Hide the Pain Harold](https://i.imgflip.com/28j0te.jpg)
*Me pretending the logout button works fine in demo*

---

## Lesson 2: Hardware IDs Are Fragile

**The Bug:** 30% of users had their HWID change after rebooting.

**The Lesson:** Only use hardware identifiers that are physically soldered onto the machine:

| Trust Level | Components |
|-------------|------------|
| TRUST | CPU ID, Motherboard Serial, BIOS Serial |
| CAREFUL | Windows Product ID, HDD Serial |
| NEVER | MAC Address, IP Address, Hostname |

MAC addresses change when you install Docker, VPNs, or basically look at them wrong.

![This is Fine](https://i.imgflip.com/wxica.jpg)
*Me after 47 HWID reset requests on a Monday morning*

---

## Lesson 3: Async is Hard

**The Bug:** Multiple API requests simultaneously refreshed the auth token, invalidating each other.

**The Lesson:** When multiple async operations depend on a shared resource, use a mutex pattern:

```typescript
let isRefreshing = false;
let subscribers: Array<(token: string) => void> = [];

if (needsRefresh) {
  if (!isRefreshing) {
    isRefreshing = true;
    const token = await doRefresh();
    subscribers.forEach(cb => cb(token));
    subscribers = [];
    isRefreshing = false;
  } else {
    await new Promise(resolve => subscribers.push(resolve));
  }
}
```

One request refreshes. Others wait. Everyone's happy.

![Roll Safe](https://i.imgflip.com/1h7in3.jpg)
*Can't have race conditions if you add a mutex*

---

## Lesson 4: Frameworks Have Defaults (And They Will Hurt You)

**The Bug:** All PUT and PATCH requests returned 307 redirects with empty bodies.

**The Lesson:** FastAPI's default `redirect_slashes=True` redirects `/users/123` to `/users/123/`. For PUT requests, browsers drop the body during the redirect.

```python
# Disable the sneaky default
app = FastAPI(
    redirect_slashes=False  # This one line took 3 hours to find
)
```

Always read the docs for default behaviors. ALWAYS.

---

## Lesson 5: Migrations Need Coordination

**The Bug:** `alembic upgrade head` failed with "Multiple heads detected."

**The Lesson:** When multiple developers work on database migrations:

1. **Communicate** before creating migrations
2. Make migrations **idempotent** (check before modify)
3. Add **CI checks** for multiple heads
4. **Merge** migration branches before they diverge too far

```bash
# Add to CI
heads=$(alembic heads | wc -l)
if [ "$heads" -gt 1 ]; then
    echo "ERROR: Multiple migration heads!"
    exit 1
fi
```

![Git Merge](https://i.imgflip.com/4acd7j.png)
*Database migrations when two developers work in parallel*

---

## Lesson 6: Time is Relative (Store UTC, Display Local)

**The Bug:** Korean users saw licenses expiring 9 hours early.

**The Lesson:**
- **Store** all times as UTC in the database
- **Attach** timezone info when serializing (`+00:00`)
- **Convert** to local time only in the frontend

```typescript
// Frontend: Always convert UTC to user's timezone
formatInTimeZone(date, 'Asia/Seoul', 'yyyy년 MM월 dd일 HH:mm:ss');
```

Never do timezone math manually. Use a library. Trust me.

---

## Lesson 7: Sometimes You Need to Redesign

**The Bug:** Users couldn't use multiple devices. Single HWID per user wasn't enough.

**The Lesson:** When feature requests pile up against your architecture, it's time to redesign, not patch.

Going from single HWID to multi-device required:
- 3 new database tables
- 5 altered tables
- 47 modified files
- 23 commits across 4 repos

**Was it worth it?**
- HWID reset requests: 47/week → 3/week
- User satisfaction: 3.2/5 → 4.7/5

Sometimes you have to tear it down to build it right.

![Flex Tape](https://i.imgflip.com/2reqtg.png)
*That's a lot of damage... to my codebase*

---

## Lesson 8: Event Loops Are Territorial

**The Bug:** Background cleanup jobs failed with "Connection pool exhausted."

**The Lesson:** When using async Python:
- Don't mix `BackgroundScheduler` (sync) with `AsyncSession` (async)
- Use `AsyncIOScheduler` instead
- Share the event loop, don't create new ones

```python
# BAD: Sync scheduler trying to run async code
scheduler = BackgroundScheduler()  # Creates new thread
asyncio.run(cleanup())  # Creates new event loop ← WRONG

# GOOD: Async scheduler sharing FastAPI's event loop
scheduler = AsyncIOScheduler()  # Same thread as FastAPI
scheduler.add_job(cleanup, 'interval', minutes=10)
```

![Drake Hotline Bling](/assets/img/posts/shieldmod/post12-creatingneweventloop.jpg)
*Creating new event loops vs. sharing the main one*

---

## Lesson 9: Crypto Formats Vary

**The Bug:** Python signed data correctly, but C++ couldn't verify it. Ever.

**The Lesson:** Cryptographic libraries output different formats:
- Python `cryptography`: DER-encoded (ASN.1)
- OpenSSL raw: Concatenated r||s

```python
# Convert DER to raw for cross-platform compatibility
r, s = decode_dss_signature(der_signature)
raw_signature = r.to_bytes(32, 'big') + s.to_bytes(32, 'big')
# Now it's 64 bytes exactly, not 70-72 variable
```

When signing in one language and verifying in another, **explicitly document the format**.

![Choosing Between DER and Raw](/assets/img/posts/shieldmod/post12-mechoosingbetweender.jpg)
*Me choosing between DER and raw signature format*

---

## Lesson 10: React Has Rules (Follow Them or Die)

**The Bug:** "Rendered more hooks than during the previous render"

**The Lesson:** React hooks must be called in the same order every render. This means:

```tsx
// BAD: Hook after conditional return
if (isLoading) return <Spinner />;
const data = useMemo(() => ...); // ← Sometimes called, sometimes not

// GOOD: All hooks before any returns
const data = useMemo(() => ...);
if (isLoading) return <Spinner />;
```

Use the ESLint plugin. It exists for a reason.

```json
{
  "rules": {
    "react-hooks/rules-of-hooks": "error"
  }
}
```

![You Had One Job](https://i.imgflip.com/2wifvo.jpg)
*React when you call hooks conditionally*

---

## Lesson 11: Networks Are Unreliable (Build Grace Periods)

**The Bug:** Brief WiFi drops terminated users immediately, losing their work.

**The Lesson:** Distinguish between temporary and permanent failures:

| Error Type | Action | Grace Period |
|------------|--------|--------------|
| Network timeout | Retry | 30 minutes |
| Server 500 | Retry | 30 minutes |
| License expired | Terminate | None |
| Account banned | Terminate | None |

```cpp
if (isTemporaryError(error)) {
    if (minutesSinceLastSuccess < GRACE_PERIOD) {
        showWarning("Reconnecting...");
        scheduleRetry();
    } else {
        terminateGracefully("Unable to verify for 30 minutes");
    }
} else {
    terminateImmediately(error.message);
}
```

False terminations dropped from 15% to 0.5% of sessions.

![User Network Issues](/assets/img/posts/shieldmod/post12-User_experiencing_network_issues.png)
*User experiencing network issues: "First time?"*

---

## Lesson 12: CORS is a Multi-Point Check

**The Bug:** Login worked, but subsequent requests weren't authenticated.

**The Lesson:** Cross-origin cookies require ALL of these:

1. **Frontend:** `withCredentials: true`
2. **Backend CORS:** `allow_credentials=True`
3. **Backend CORS:** Explicit origins (NOT `*`)
4. **Cookie:** `SameSite=Lax` (or `None` with `Secure`)
5. **All middlewares:** Must pass through `OPTIONS`

Miss ONE and the whole thing breaks silently.

```python
# The complete CORS setup
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],  # NOT "*"!
    allow_credentials=True,  # Required for cookies
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## The Summary

| Lesson | Category | Time Lost |
|--------|----------|-----------|
| Distributed auth state | Architecture | 3 days |
| Fragile HWIDs | Platform | 2 days |
| Async mutex pattern | Concurrency | 1 week |
| Framework defaults | Configuration | 3 hours |
| Migration coordination | Teamwork | 4 hours |
| Timezone handling | Data | Ongoing |
| Major redesign | Architecture | 2 weeks |
| Event loop isolation | Python | 4 hours |
| Crypto format mismatch | Cross-platform | 6 hours |
| React hook rules | Frontend | 30 minutes |
| Grace period handling | UX | 3 days |
| CORS configuration | Security | 2 days |

## Final Thoughts

The 1,060 fix commits in this project weren't failures. Each one was a lesson. Each bug taught me something about the technology, about users, or about myself.

If I could go back and give myself one piece of advice before starting:

> **Log everything. Document assumptions. Test logout as thoroughly as login. And never, ever trust MAC addresses.**

---

*Thanks for following along with the "Building ShieldMod" series. If you're building something similar, I hope these lessons save you some of the pain I went through.*

*Now if you'll excuse me, I have some new support tickets to read.*
