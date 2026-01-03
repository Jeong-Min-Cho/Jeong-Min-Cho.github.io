---
layout: post
title: "Race Conditions: When 5 API Requests All Refresh the Token"
date: 2025-12-15 10:00 +0900
categories: [Programming, Debugging]
tags: [ShieldMod, JavaScript, TypeScript, Async, Authentication, Concurrency]
---

## The Heisenbug

Some bugs are easy. They happen every time, you debug them, you fix them. Done.

Then there are Heisenbugs - bugs that only appear when you're not looking. Bugs that work fine when you test manually but fail 5% of the time in production. Bugs that make you question your sanity.

This is a story about one of those bugs.

![Works On My Machine](https://i.imgflip.com/1ihzfe.jpg)
*"Works on my machine" - the battle cry of developers everywhere*

## The Symptoms

Users reported "random 401 errors" - intermittent authentication failures despite being logged in. The support tickets all had the same pattern:

- "I was browsing normally, then suddenly got logged out"
- "Some requests work, some fail, it's random"
- "The dashboard shows my data but the sidebar shows 'unauthorized'"

### Impact Report
- **Occurrence rate:** ~5% of sessions
- **Pattern:** Only happened when loading data-heavy pages
- **Time to resolve:** 1 week (tricky to reproduce)

## The Investigation

Let me show you the timeline of what was happening:

```
T+0ms:    Request A starts, token expired
T+2ms:    Request A calls /auth/refresh
T+5ms:    Request B starts, token still shows expired
T+7ms:    Request B calls /auth/refresh (DUPLICATE!)
T+10ms:   Request C starts, calls /auth/refresh (TRIPLICATE!)
T+15ms:   Request A gets new token, saves it
T+18ms:   Request B gets ERROR (refresh token already used!)
T+20ms:   Request C gets ERROR (refresh token already used!)
T+22ms:   Requests B and C fail with 401
T+25ms:   User sees partial data, some components show errors

Result: 2 of 3 requests failed randomly
```

The problem? My refresh token was **one-time use** (as it should be for security). But multiple requests were trying to use it simultaneously.

![Spider-Man Pointing](https://i.imgflip.com/1tkjq9.jpg)
*Request A, B, and C all trying to be the one that refreshes the token*

## The Buggy Code

Here's what my original implementation looked like:

```typescript
// BROKEN: Every request refreshes independently
api.interceptors.request.use(async (config) => {
  const token = getAccessToken();

  if (isTokenExpired(token)) {
    // Problem: Multiple requests hit this simultaneously!
    const newToken = await refreshToken();
    setAccessToken(newToken);
  }

  config.headers.Authorization = `Bearer ${getAccessToken()}`;
  return config;
});
```

The issue is subtle. When the dashboard loads, it might fire 5-10 API requests at once. If the token is expired (or about to expire), EACH request independently decides to refresh it. They all race to call the refresh endpoint, and only the first one wins.

## The Fix: Token Refresh Mutex

The solution is a classic concurrency pattern - a mutex-like approach where only one request does the refresh, and others wait for it:

```typescript
// lib/api.ts

// Mutex-like state
let isRefreshing = false;
let refreshSubscribers: Array<(token: string) => void> = [];

function subscribeToRefresh(callback: (token: string) => void) {
  refreshSubscribers.push(callback);
}

function notifySubscribers(token: string) {
  refreshSubscribers.forEach(callback => callback(token));
  refreshSubscribers = [];
}

api.interceptors.request.use(async (config) => {
  // Skip auth for auth endpoints themselves
  if (config.url?.includes('/auth/')) {
    return config;
  }

  const token = getAccessToken();
  const REFRESH_BUFFER_SECONDS = 120; // Refresh 2 min before expiry

  // Check if token needs refresh
  if (isTokenExpired(token) || willExpireSoon(token, REFRESH_BUFFER_SECONDS)) {

    if (!isRefreshing) {
      // This request wins the race - it does the refresh
      isRefreshing = true;

      try {
        const response = await axios.post('/auth/refresh', {
          refresh_token: getRefreshToken()
        });

        const newToken = response.data.access_token;
        setAccessToken(newToken);

        // Notify all waiting requests
        notifySubscribers(newToken);

        config.headers.Authorization = `Bearer ${newToken}`;
      } catch (error) {
        // Refresh failed - clear everything and redirect
        clearTokens();
        window.location.href = '/login?expired=true';
        throw error;
      } finally {
        isRefreshing = false;
      }
    } else {
      // Another request is already refreshing - wait for it
      return new Promise((resolve) => {
        subscribeToRefresh((newToken) => {
          config.headers.Authorization = `Bearer ${newToken}`;
          resolve(config);
        });
      });
    }
  } else {
    // Token is still valid
    config.headers.Authorization = `Bearer ${token}`;
  }

  return config;
});
```

## Visualizing the Fix

```
BEFORE (Race Condition):
─────────────────────────────────────────────────
Request A ──▶ refresh() ──▶ ✓
Request B ──▶ refresh() ──▶ ✗ (token invalidated)
Request C ──▶ refresh() ──▶ ✗ (token invalidated)


AFTER (Queue Pattern):
─────────────────────────────────────────────────
Request A ──▶ refresh() ──▶ ✓ ──▶ notify all
Request B ──▶ [waiting...] ────▶ ✓ (uses A's token)
Request C ──▶ [waiting...] ────▶ ✓ (uses A's token)
```


## The Subscriber Pattern Explained

The key insight is the subscriber pattern:

1. **First request** sets `isRefreshing = true` and makes the actual refresh call
2. **Subsequent requests** see `isRefreshing` is already true, so they add a callback to `refreshSubscribers` and return a Promise that won't resolve until notified
3. **When refresh completes**, `notifySubscribers()` calls all waiting callbacks with the new token
4. **All Promises resolve** with their configs now containing the fresh token

It's like a velvet rope at a club. The first person talks to the bouncer (makes the API call), everyone else waits in line (subscribes), and when the door opens (token received), everyone goes in together.

## Proactive Refresh

Notice the `REFRESH_BUFFER_SECONDS` constant:

```typescript
const REFRESH_BUFFER_SECONDS = 120; // Refresh 2 min before expiry

if (isTokenExpired(token) || willExpireSoon(token, REFRESH_BUFFER_SECONDS)) {
```

This refreshes the token **2 minutes before** it expires. Why? Because if the token expires during a request, it's too late. By refreshing proactively, we ensure the token is always valid when requests go out.

## The Results

| Metric | Before | After |
|--------|--------|-------|
| Duplicate refresh calls | 3-10 per session | 0 |
| 401 errors (false positive) | ~5% of sessions | 0% |
| Refresh API calls/hour | ~150 | ~20 |
| User-reported login issues | 8/week | 0 |

## Bonus: Server-Side Protection

Even with the client-side fix, I added server-side protection too. If somehow duplicate refresh requests come in, the server should handle it gracefully:

```python
@router.post("/auth/refresh")
async def refresh_token(request: RefreshRequest, db: AsyncSession = Depends(get_db)):
    # Find the refresh token
    token = await db.scalar(
        select(RefreshToken).where(
            RefreshToken.token == request.refresh_token,
            RefreshToken.is_used == False,  # Not already used
            RefreshToken.expires_at > datetime.utcnow()
        )
    )

    if not token:
        raise HTTPException(401, "Invalid or expired refresh token")

    # Mark as used IMMEDIATELY (before generating new tokens)
    token.is_used = True
    await db.commit()

    # Now generate new tokens...
```

The key is marking the token as used **before** doing anything else. This way, even if two requests somehow get through, only the first one succeeds.

![Bernie Sanders](https://i.imgflip.com/3oevdk.jpg)
*I am once again asking you to use a mutex for your shared resources*

## Key Takeaways

1. **Concurrent requests share state** - When your UI fires multiple API calls, they might all see the same "expired" state.

2. **Use a mutex for shared resources** - The subscriber pattern ensures only one refresh happens.

3. **Proactive refresh > Reactive refresh** - Refresh before expiry, not after.

4. **Defense in depth** - Client-side mutex + server-side protection = belt and suspenders.

5. **Log everything during debugging** - I only found this bug by adding detailed timing logs to every request.

---

*Next time: The tale of 12 engineering lessons learned from building a full-stack platform, complete with a meme for each one.*

*This is part 4 of my "Building ShieldMod" series.*
