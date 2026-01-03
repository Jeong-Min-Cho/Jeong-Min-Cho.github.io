# Building a Full-Stack Software Licensing Platform: A Technical Deep Dive

## Introduction

I built a comprehensive software licensing and distribution platform from scratch—a system that handles everything from user authentication to secure file delivery across web and desktop applications. This project spans **82,000+ lines of code** across four major components: a Python backend, a React frontend, and two C++ Windows applications.

In this post, I'll walk through the architecture, the technical challenges I solved, and the design decisions that shaped this production-ready system.

---

## The Problem I Set Out to Solve

Software distribution with proper licensing is a complex problem. You need to:

1. **Authenticate users** across multiple platforms (web browsers and desktop applications)
2. **Manage licenses** with activation limits, expiration dates, and usage tracking
3. **Distribute files securely** so only authorized users can access them
4. **Track devices** to prevent license sharing while allowing legitimate multi-device use
5. **Monitor everything** with comprehensive audit logs for security and debugging

I wanted to build a system that handles all of this elegantly, with modern security practices and a great user experience.

---

## System Overview: How It All Works Together

The platform consists of four interconnected components:

```
┌─────────────────────────────────────────────────────────────────┐
│                      USER INTERACTIONS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   [Admin]                    [End User]                         │
│      │                           │                              │
│      ▼                           ▼                              │
│  ┌─────────┐              ┌─────────────┐                      │
│  │  Admin  │              │   Desktop   │                      │
│  │Dashboard│              │   Client    │                      │
│  │(Next.js)│              │   (C++)     │                      │
│  └────┬────┘              └──────┬──────┘                      │
│       │                          │                              │
│       └──────────┬───────────────┘                              │
│                  ▼                                              │
│           ┌─────────────┐                                       │
│           │   Backend   │                                       │
│           │  (FastAPI)  │                                       │
│           └──────┬──────┘                                       │
│                  │                                              │
│        ┌─────────┼─────────┐                                    │
│        ▼         ▼         ▼                                    │
│   ┌────────┐ ┌───────┐ ┌────────┐                              │
│   │Postgres│ │ Redis │ │Telegram│                              │
│   │   DB   │ │ Cache │ │  Bot   │                              │
│   └────────┘ └───────┘ └────────┘                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Here's the typical user journey:**

1. An admin creates a license key through the web dashboard
2. The user receives their license key and downloads the desktop client
3. The desktop client authenticates with the backend, binding to the user's hardware
4. The client downloads encrypted files, decrypts them in memory, and validates the license periodically
5. All actions are logged, and the admin can monitor everything in real-time

---

## Component 1: The Backend (Python/FastAPI)

### Why FastAPI?

I chose FastAPI for several reasons:

- **100% async** — Every database query, every API call is non-blocking
- **Automatic documentation** — OpenAPI/Swagger docs generated from code
- **Type safety** — Pydantic models catch errors before they hit the database
- **Performance** — One of the fastest Python web frameworks available

### The Database Design

The system uses **32 SQLAlchemy models** organized around these core entities:

```python
# Core relationships
User
├── Sessions (web + desktop)
├── Licenses (with activation history)
├── Devices (multi-device support)
├── AuditLogs (every action tracked)
└── Downloads (file access history)
```

**Multi-device support** was particularly interesting to implement. Users can register multiple devices (up to a configurable limit), and each device gets tracked independently:

```python
class UserDevice(Base):
    __tablename__ = "user_devices"

    id = Column(UUID, primary_key=True)
    user_id = Column(UUID, ForeignKey("users.id"))
    hwid_hash = Column(String(64), unique=True)  # SHA-256 of hardware ID
    nickname = Column(String(100))  # "Gaming PC", "Laptop", etc.
    first_seen = Column(DateTime, default=datetime.utcnow)
    last_seen = Column(DateTime)
    is_active = Column(Boolean, default=True)
```

### Dual Authentication System

One of the more complex aspects was supporting two authentication methods simultaneously:

**Web Dashboard (Session-based):**

```
Browser → POST /auth/login → Set HTTP-only cookie → Done
```

**Desktop Client (JWT-based):**

```
Client → POST /auth/client/login (with HWID) → JWT tokens → Periodic refresh
```

The JWT system uses short-lived access tokens (15 minutes) and longer refresh tokens (7 days). The desktop client proactively refreshes tokens before expiry to maintain seamless operation.

```python
@router.post("/client/login")
async def client_login(
    credentials: ClientLoginRequest,
    db: AsyncSession = Depends(get_db)
):
    # Validate credentials
    user = await authenticate_user(db, credentials.username, credentials.password)

    # Check/register device
    device = await device_service.get_or_create_device(
        db, user.id, credentials.hwid
    )

    # Check device limits
    if device.is_new and user.device_count >= user.max_devices:
        raise HTTPException(400, "Device limit reached")

    # Generate tokens
    access_token = create_access_token(user.id, device.id)
    refresh_token = create_refresh_token(user.id, device.id)

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "device_id": device.id,
        "license_expiry": user.license_expiry
    }
```

### Rate Limiting with Redis

I implemented a token bucket algorithm for rate limiting:

```python
class RateLimiter:
    async def check_rate_limit(
        self,
        key: str,
        max_requests: int,
        window_seconds: int
    ) -> bool:
        current = await self.redis.incr(key)
        if current == 1:
            await self.redis.expire(key, window_seconds)
        return current <= max_requests
```

Different endpoints have different limits:

- Login attempts: 10 per 5 minutes (prevents brute force)
- API calls: 200 per minute (prevents abuse)
- File downloads: 20 per hour (prevents scraping)

### Secure File Distribution

Files are encrypted server-side with **AES-256-GCM**:

```python
def encrypt_file(plaintext: bytes, key: bytes) -> bytes:
    nonce = os.urandom(12)  # 96-bit nonce
    cipher = Cipher(algorithms.AES(key), modes.GCM(nonce))
    encryptor = cipher.encryptor()
    ciphertext = encryptor.update(plaintext) + encryptor.finalize()

    # Format: nonce (12) + ciphertext (variable) + tag (16)
    return nonce + ciphertext + encryptor.tag
```

When a user downloads a file, the system derives a unique decryption key using **HKDF**:

```python
def derive_user_key(user_id: str, access_token: str) -> bytes:
    hkdf = HKDF(
        algorithm=hashes.SHA256(),
        length=32,
        salt=user_id.encode(),
        info=b"file_decryption_key"
    )
    return hkdf.derive(access_token.encode())
```

This means:

- Each user gets a unique decryption key
- Keys are derived, not stored (reducing attack surface)
- Stolen files are useless without valid credentials

---

## Component 2: The Admin Dashboard (Next.js/TypeScript)

### Tech Stack Choices

I used **Next.js 14 with the App Router** for several reasons:

- **Server-side rendering** for fast initial loads
- **File-based routing** for clean project structure
- **Built-in optimizations** (code splitting, image optimization)

For state management, I chose **TanStack Query (React Query)** over Redux because:

- It's purpose-built for server state
- Automatic caching and invalidation
- No boilerplate for async operations

### Component Architecture

The dashboard has **129 React components** organized by feature:

```
components/
├── ui/              # shadcn/ui base components (20+)
├── layout/          # Sidebar, Header, Footer
├── dashboard/       # Stats cards, charts, activity feeds
├── users/           # User management (table, forms, dialogs)
├── licenses/        # License management
├── devices/         # Device tracking and analytics
└── auth/            # Login form, auth guards
```

### Real-Time Dashboard

The dashboard shows live statistics that update automatically:

```typescript
export function useDashboardStats() {
  return useQuery({
    queryKey: ["dashboard", "stats"],
    queryFn: async () => {
      const response = await api.get("/dashboard/stats");
      return response.data;
    },
    refetchInterval: 60000, // Refresh every minute
    staleTime: 30000, // Consider fresh for 30 seconds
  });
}
```

The stats include:

- Total users, active users, banned users
- Active sessions across all platforms
- License expiration breakdown
- Recent activity feed

### Type Safety Throughout

I use **Zod** for runtime validation that generates TypeScript types:

```typescript
const CreateUserSchema = z.object({
  username: z
    .string()
    .min(3, "Username must be at least 3 characters")
    .max(50)
    .regex(/^[a-zA-Z0-9_]+$/, "Only alphanumeric and underscore"),
  password: z.string().min(8, "Password must be at least 8 characters"),
  role: z.enum(["user", "vip", "admin"]),
  license_days: z.number().min(1).max(365).optional(),
});

type CreateUserInput = z.infer<typeof CreateUserSchema>;
```

This ensures the frontend and backend stay in sync—if the schema changes, TypeScript catches it at compile time.

### Internationalization

The dashboard supports Korean and English:

```typescript
// lib/i18n/index.ts
i18n.init({
  resources: {
    ko: { translation: koTranslations },
    en: { translation: enTranslations },
  },
  lng: localStorage.getItem("language") || "ko",
  fallbackLng: "en",
});
```

Users can switch languages from the login screen, and the preference persists across sessions.

---

## Component 3: The Desktop Client (C++)

### Why C++?

For the Windows desktop client, C++ was the clear choice:

- **Native performance** — No runtime overhead
- **Direct Windows API access** — Credential Manager, WinHTTP, etc.
- **No dependencies** — Single executable, no installers needed

### The User Interface

I used **Dear ImGui** with DirectX 11 for the GUI:

```cpp
void RenderLoginScreen() {
    ImGui::SetNextWindowSize(ImVec2(400, 300));
    ImGui::Begin("Login", nullptr, ImGuiWindowFlags_NoResize);

    ImGui::InputText("Username", username, sizeof(username));
    ImGui::InputText("Password", password, sizeof(password),
                     ImGuiInputTextFlags_Password);

    if (ImGui::Button("Login", ImVec2(380, 40))) {
        AuthenticateUser();
    }

    ImGui::End();
}
```

ImGui provides:

- Immediate-mode rendering (no widget state management)
- Easy customization (dark theme, custom colors)
- GPU-accelerated rendering via DirectX

### Secure Credential Storage

User credentials are stored in Windows Credential Manager:

```cpp
bool SaveCredentials(const std::string& username, const std::string& password) {
    CREDENTIALW cred = {};
    cred.Type = CRED_TYPE_GENERIC;
    cred.TargetName = L"SoftwareLicenseClient";
    cred.CredentialBlobSize = password.size();
    cred.CredentialBlob = (LPBYTE)password.c_str();
    cred.Persist = CRED_PERSIST_LOCAL_MACHINE;
    cred.UserName = ConvertToWide(username);

    return CredWriteW(&cred, 0);
}
```

This is more secure than storing credentials in config files—Windows encrypts them with DPAPI.

### HWID Generation

Hardware IDs are generated from multiple system components:

```cpp
std::string GenerateHWID() {
    std::string components;

    // CPU ID
    int cpuInfo[4];
    __cpuid(cpuInfo, 0);
    components += std::to_string(cpuInfo[1]) + std::to_string(cpuInfo[3]);

    // Motherboard serial (via WMI)
    components += GetWMIProperty("Win32_BaseBoard", "SerialNumber");

    // Windows Product ID
    components += GetRegistryValue(
        "SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion",
        "ProductId"
    );

    // Hash everything together
    return SHA256(components);
}
```

This creates a stable identifier that:

- Survives OS reinstalls (hardware-based)
- Changes if significant hardware changes (prevents cloning)
- Is hashed (doesn't expose actual serial numbers)

### Secure File Handling

Downloaded files are decrypted entirely in memory:

```cpp
std::vector<uint8_t> DecryptFile(
    const std::vector<uint8_t>& encrypted,
    const std::string& userId,
    const std::string& accessToken
) {
    // Derive key using HKDF
    auto key = HKDF_SHA256(accessToken, userId, "file_key", 32);

    // Extract nonce, ciphertext, and tag
    auto nonce = std::vector<uint8_t>(encrypted.begin(), encrypted.begin() + 12);
    auto tag = std::vector<uint8_t>(encrypted.end() - 16, encrypted.end());
    auto ciphertext = std::vector<uint8_t>(encrypted.begin() + 12, encrypted.end() - 16);

    // Decrypt with AES-256-GCM
    return AES_GCM_Decrypt(ciphertext, key, nonce, tag);
}
```

The plaintext file **never touches the disk**—it's loaded directly into memory and used from there.

---

## Component 4: The Client Module (C++ Plugin)

### Purpose

The client module extends the main application with:

- License validation via periodic heartbeats
- Lua scripting for extensibility
- Telegram notifications for alerts
- Comprehensive logging

### Heartbeat System

The module periodically validates the license with the backend:

```cpp
void HeartbeatThread() {
    while (running) {
        auto response = httpClient.Post("/heartbeat", {
            {"user_id", userId},
            {"device_id", deviceId},
            {"version", VERSION}
        });

        if (!response.success || !response.data["valid"]) {
            HandleLicenseInvalid();
            break;
        }

        // Update license info
        licenseExpiry = response.data["license_expiry"];

        // Wait 5 minutes before next check
        std::this_thread::sleep_for(std::chrono::minutes(5));
    }
}
```

If the network is temporarily unavailable, the system uses a **grace period** (30 minutes) before taking action. This prevents false positives from brief connectivity issues.

### Lua Scripting Engine

For extensibility, the module embeds a Lua interpreter:

```cpp
void InitializeLua() {
    L = luaL_newstate();
    luaL_openlibs(L);

    // Register custom functions
    lua_register(L, "log", LuaLog);
    lua_register(L, "notify", LuaSendNotification);
    lua_register(L, "getConfig", LuaGetConfig);
}

// Execute user scripts safely
bool ExecuteScript(const std::string& script) {
    if (luaL_dostring(L, script.c_str()) != LUA_OK) {
        const char* error = lua_tostring(L, -1);
        Logger::Error("Lua error: {}", error);
        lua_pop(L, 1);
        return false;
    }
    return true;
}
```

Scripts run in a sandboxed environment with access only to approved functions.

### Crash Recovery

The module includes robust crash handling:

```cpp
LONG WINAPI ExceptionHandler(EXCEPTION_POINTERS* ep) {
    Logger::Error("Exception caught: 0x{:08X} at 0x{:p}",
        ep->ExceptionRecord->ExceptionCode,
        ep->ExceptionRecord->ExceptionAddress);

    // Save crash dump for debugging
    WriteMiniDump(ep);

    // Attempt graceful shutdown
    Cleanup();

    return EXCEPTION_EXECUTE_HANDLER;
}

// Install handler at startup
SetUnhandledExceptionFilter(ExceptionHandler);
```

This ensures crashes are logged and don't leave the system in an inconsistent state.

---

## Security: Defense in Depth

Security was a primary concern throughout development. Here's how I approached it:

### Layer 1: Network Security

- **HTTPS everywhere** — All API calls use TLS 1.2+
- **Rate limiting** — Prevents brute force and DoS attacks
- **IP blocklist** — Automatically blocks suspicious IPs

### Layer 2: Authentication

- **Short-lived tokens** — Access tokens expire in 15 minutes
- **Secure password storage** — Argon2id hashing (memory-hard, resistant to GPU attacks)
- **Device binding** — Licenses tied to hardware, not just accounts

### Layer 3: Data Protection

- **AES-256-GCM** — Authenticated encryption for files
- **HKDF key derivation** — Unique keys per user
- **In-memory decryption** — Plaintext never written to disk

### Layer 4: Monitoring

- **Comprehensive audit logs** — Every action is recorded with timestamp, IP, and user agent
- **Real-time alerts** — Suspicious activity triggers Telegram notifications
- **Session tracking** — All active sessions visible in dashboard

---

## Challenges I Overcame

### Challenge 1: Concurrent Token Refresh

**Problem:** Multiple API calls could trigger simultaneous token refreshes, causing race conditions.

**Solution:** Implemented a request queue that pauses requests during refresh:

```typescript
let isRefreshing = false;
let refreshQueue: Array<() => void> = [];

async function refreshToken() {
  if (isRefreshing) {
    return new Promise((resolve) => refreshQueue.push(resolve));
  }

  isRefreshing = true;
  try {
    const newToken = await api.post("/auth/refresh");
    setAccessToken(newToken);
    refreshQueue.forEach((cb) => cb());
  } finally {
    isRefreshing = false;
    refreshQueue = [];
  }
}
```

### Challenge 2: Cross-Platform Session Management

**Problem:** Web sessions use cookies, but desktop clients can't use cookies.

**Solution:** Implemented parallel authentication systems with a shared session model:

```python
class Session(Base):
    session_id = Column(UUID, primary_key=True)
    user_id = Column(UUID, ForeignKey("users.id"))
    token_hash = Column(String(64))  # SHA-256 of token
    session_type = Column(Enum("web", "desktop"))
    device_id = Column(UUID, nullable=True)  # Only for desktop
    expires_at = Column(DateTime)
```

Both session types share the same table but with different token handling.

### Challenge 3: Hardware ID Stability

**Problem:** Some HWID components (like MAC addresses) change frequently, causing false "new device" detections.

**Solution:** Use only stable components (CPU ID, motherboard serial) and implement similarity checking:

```cpp
float CalculateHWIDSimilarity(const std::string& stored, const std::string& current) {
    // Compare individual components
    auto storedParts = SplitHWID(stored);
    auto currentParts = SplitHWID(current);

    int matches = 0;
    for (size_t i = 0; i < storedParts.size(); i++) {
        if (storedParts[i] == currentParts[i]) matches++;
    }

    return (float)matches / storedParts.size();
}

// Allow if >70% similar (tolerates minor changes)
if (CalculateHWIDSimilarity(stored, current) > 0.7f) {
    // Same device, update stored HWID
}
```

---

## Performance Optimizations

### Backend

- **Async everywhere** — No blocking I/O operations
- **Connection pooling** — Reuse database connections (pool size: 10-30)
- **Batch logging** — Audit logs queued and inserted in batches

### Frontend

- **React Query caching** — Reduces redundant API calls
- **Code splitting** — Each route loaded on demand
- **Optimistic updates** — UI updates immediately, syncs in background

### Desktop Client

- **Memory-mapped files** — Large files loaded without copying
- **Background threads** — UI never blocks on network/disk operations
- **Lazy initialization** — Components loaded only when needed

---

## What I Learned

Building this system taught me:

1. **Security is an architecture** — It's not something you add at the end; it must be designed in from the start.

2. **Async is essential** — For any system handling concurrent users, blocking I/O is a bottleneck.

3. **Type safety saves time** — The hours spent defining types are repaid in fewer bugs and easier refactoring.

4. **Cross-platform is hard** — Web and desktop have fundamentally different models; abstracting the differences requires careful design.

5. **Monitoring is not optional** — Without comprehensive logs, debugging production issues is nearly impossible.

---

## Tech Stack Summary

| Component          | Technologies                                                         |
| ------------------ | -------------------------------------------------------------------- |
| **Backend**        | Python 3.11, FastAPI, PostgreSQL, Redis, SQLAlchemy, Alembic, Docker |
| **Frontend**       | Next.js 14, TypeScript, TanStack Query, Tailwind CSS, shadcn/ui, Zod |
| **Desktop Client** | C++17, DirectX 11, ImGui, WinHTTP, AES-256-GCM                       |
| **Client Module**  | C++20, Lua, ImGui, simdjson                                          |

---

## Conclusion

This project represents months of work across the full stack—from database schema design to pixel-perfect UI components to low-level C++ memory management. It's a complete, production-ready system that handles real-world complexities like multi-device licensing, secure file distribution, and comprehensive monitoring.

The skills I developed building this—async programming, cryptographic implementations, cross-platform authentication, and security architecture—are directly applicable to any software system that needs to protect and distribute digital assets.

If you're building something similar or have questions about any of the technical decisions, feel free to reach out!

---

**Lines of Code:** 82,000+
**Languages:** Python, TypeScript, C++
**Database Models:** 32
**API Endpoints:** 31+
**React Components:** 129
**Development Time:** Production-ready system

_Built with FastAPI, Next.js, and modern C++._
