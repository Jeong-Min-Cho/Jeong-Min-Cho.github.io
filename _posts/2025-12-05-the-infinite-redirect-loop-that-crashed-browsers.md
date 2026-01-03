---
layout: post
title: "The Infinite Redirect Loop That Crashed Browsers"
date: 2025-12-05 10:00 +0900
categories: [Programming, Debugging]
tags: [ShieldMod, React, Next.js, Authentication, Debugging, JavaScript]
---

## When Logout Becomes an Infinite Loop

Picture this: You've just deployed your beautiful authentication system. Users can log in, browse around, do their thing. Life is good. Then someone clicks the logout button...

**200+ API requests in under 5 seconds. Browser tab crashes. CPU goes to 100%.**

![Panik Kalm Panik](/assets/img/posts/shieldmod/post2-1.png)
*Logout button clicked → Panik. Browser seems fine → Kalm. 200 requests in 5 seconds → PANIK*

## The Crime Scene

### Impact Report
- **Time to resolve:** 4 PRs over 3 days
- **User reports:** 12 support tickets
- **Related commits:** 6 fix commits

When users clicked logout, their browsers would rapidly cycle between `/login` and `/dashboard`. The network tab looked like a machine gun - request after request after request until Chrome decided it had enough and killed the tab.

```
Request #1:   GET /login       → 200
Request #2:   GET /dashboard   → 302 → /login
Request #3:   GET /login       → 302 → /dashboard
Request #4:   GET /dashboard   → 302 → /login
...
Request #247: Browser: "I'm out of here" 💀
```

## The Investigation

The commit history tells the story of my descent into madness:

```
fix: prevent infinite redirect loop on logout
fix: redirect to /login?expired=true on logout to clear cookies
fix: prevent infinite redirect loop on logout (yes, again)
fix: disable auth query on public routes to prevent redirect loop
fix: redirect immediately on logout without waiting for API
fix: clear cookies with correct domain for production
```

Yes, that's **two commits** with nearly the same message. That's what debugging does to you.

## The Root Cause

Here's the thing about modern auth - your user state exists in at least 4 places:

1. **HTTP-only Cookies** (server-set, browser-managed)
2. **React State** (component-level)
3. **React Query Cache** (global cache)
4. **LocalStorage** (refresh tokens, maybe)

When you click logout, you need to clear ALL of them. But here's what my original code did:

```typescript
// The innocent-looking logout function
async function logout() {
  // Clear React state
  setUser(null);

  // Redirect to login
  router.push('/login');

  // Call logout API (async, non-blocking)
  await api.post('/auth/logout');
}
```

Can you spot the bug? Let me draw it out:

```
┌─────────────────────────────────────────────────────────────────┐
│                    REDIRECT LOOP ANATOMY                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. User clicks logout                                         │
│   2. Frontend clears React state                                │
│   3. Frontend redirects to /login                               │
│   4. React Query useAuth hook fires on login page              │
│   5. Cookie still exists (not cleared yet!)                     │
│   6. Middleware sees cookie → redirect to /dashboard            │
│   7. Dashboard sees no React state → redirect to /login         │
│   8. GOTO step 4 (infinite loop)                                │
│                                                                 │
│   Time per cycle: ~25ms                                         │
│   Requests before crash: 200-400                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

The cookie was the traitor. The React state was cleared, but the cookie was still chilling in the browser, making the middleware think we were still logged in.

## The Fix

### Step 1: Nuclear Cookie Destruction

First, we need to absolutely, positively, with extreme prejudice, delete every possible cookie variation:

```typescript
// lib/auth.ts
export async function logout() {
  // Clear all possible cookie variations across domains
  const cookieNames = ['session_id', 'session', 'token', 'connect.sid'];
  const domains = ['', '.localhost', window.location.hostname];
  const paths = ['/', '/api', '/dashboard'];

  cookieNames.forEach(name => {
    domains.forEach(domain => {
      paths.forEach(path => {
        document.cookie = `${name}=; expires=Thu, 01 Jan 1970 00:00:00 GMT; path=${path}; domain=${domain}`;
      });
    });
  });

  // Clear React Query cache immediately
  queryClient.clear();

  // Clear localStorage tokens
  localStorage.removeItem('refresh_token');

  // Fire-and-forget logout API call (don't wait)
  api.post('/auth/logout').catch(() => {});

  // Redirect immediately with flag
  window.location.href = '/login?expired=true';
}
```

Notice we're not using `router.push()` - we're using `window.location.href`. This causes a full page reload, which ensures we're really starting fresh.

### Step 2: Don't Check Auth on Public Routes

The login page was triggering an auth check, which saw the cookie (before it was cleared) and redirected back. Stop that:

```typescript
// hooks/useAuth.ts
const PUBLIC_ROUTES = ['/login', '/register', '/forgot-password', '/reset-password'];

export function useAuth() {
  const pathname = usePathname();
  const isPublicRoute = PUBLIC_ROUTES.some(route => pathname?.startsWith(route));

  return useQuery({
    queryKey: ['auth', 'me'],
    queryFn: fetchCurrentUser,
    enabled: !isPublicRoute, // Critical: don't fetch on public routes
    retry: false,
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
}
```

### Step 3: Handle the Expired Flag

On the server-side middleware, we recognize the `?expired=true` flag and clean up any remaining cookies:

```typescript
// middleware.ts
export function middleware(request: NextRequest) {
  const { pathname, searchParams } = request.nextUrl;

  // If coming from logout, clear cookies server-side too
  if (searchParams.get('expired') === 'true') {
    const response = NextResponse.redirect(
      new URL('/login', request.url)
    );

    // Clear all auth cookies server-side
    ['session_id', 'session', 'token'].forEach(name => {
      response.cookies.delete(name);
    });

    return response;
  }

  // Normal auth check continues...
}
```

## The Results

| Metric | Before | After |
|--------|--------|-------|
| Logout time | ∞ (crash) | 150ms |
| Network requests | 200+ | 2 |
| CPU spike | 100% | <5% |
| User complaints | 12/week | 0 |

## The Meme That Summarizes It All

![Expanding Brain](https://i.imgflip.com/26k21l.jpg)

- **Small brain:** Clear React state on logout
- **Medium brain:** Clear React state AND cookies
- **Galaxy brain:** Clear React state, cookies, React Query cache, use full page reload, AND handle it server-side

![UNO Draw 25](/assets/img/posts/shieldmod/post2-2.jpg)
*"Test logout as thoroughly as login" or draw 25*

## Key Takeaways

1. **Authentication state is distributed** - You need to clear cookies, React state, React Query cache, and localStorage atomically.

2. **Order matters** - Clear everything BEFORE redirecting, not during or after.

3. **Skip auth checks on public routes** - Your login page doesn't need to check if you're logged in.

4. **Use a flag to prevent loops** - The `?expired=true` parameter lets the receiving page know this is a logout redirect.

5. **Test logout as thoroughly as login** - I tested login 50 times. I tested logout... once. Guess which one broke in production?

---

*Next up: The tale of how MAC addresses betrayed me and caused 47 support tickets in a single week. Stay tuned!*

*This is part 2 of my "Building ShieldMod" series.*
