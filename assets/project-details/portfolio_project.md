# Software Licensing & Distribution System - Portfolio Overview

A comprehensive full-stack software licensing and distribution platform with hardware-based protection, featuring a web dashboard, user portal, and secure Windows desktop client for license validation and software delivery.

---

## Project Overview

| Aspect               | Details                                                       |
| -------------------- | ------------------------------------------------------------- |
| **Project Type**     | Full-Stack Software Licensing & Distribution Platform         |
| **Total Codebase**   | ~70,000+ lines of code across 4 major components              |
| **Development Time** | Production-ready system                                       |
| **Role**             | Full-Stack Developer (Backend, Frontend, Desktop Application) |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SYSTEM ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────────────────────┐   │
│  │   Frontend  │     │   Backend   │     │    Desktop Application      │   │
│  │  (Next.js)  │────▶│  (FastAPI)  │◀────│  (Windows Client)           │   │
│  └─────────────┘     └──────┬──────┘     └─────────────────────────────┘   │
│                             │                                               │
│                    ┌────────┴────────┐                                      │
│                    │                 │                                      │
│               ┌────▼────┐      ┌─────▼─────┐                               │
│               │PostgreSQL│      │   Redis   │                               │
│               │  (Data)  │      │  (Cache)  │                               │
│               └──────────┘      └───────────┘                               │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      External Integrations                           │   │
│  ├──────────────┬──────────────┬──────────────┬────────────────────────┤   │
│  │ Telegram Bot │   OCR API    │  Monitoring  │   Korean OCR Service   │   │
│  └──────────────┴──────────────┴──────────────┴────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Component 1: Backend API

### Tech Stack

| Technology            | Purpose                                 |
| --------------------- | --------------------------------------- |
| **Python 3.11+**      | Primary language                        |
| **FastAPI 0.109**     | Async web framework                     |
| **PostgreSQL 15**     | Primary database                        |
| **Redis 7**           | Session storage, caching, rate limiting |
| **SQLAlchemy 2.0**    | Async ORM                               |
| **Alembic**           | Database migrations                     |
| **JWT (python-jose)** | Token-based authentication              |
| **Argon2id / bcrypt** | Password hashing                        |
| **AES-256-GCM**       | File encryption                         |
| **Docker Compose**    | Containerization                        |

### Key Features

**Authentication System**

- Dual authentication: Web sessions (HTTP-only cookies) + JWT tokens (desktop client)
- Multi-device support with configurable limits per user
- Hardware ID (HWID) binding for license protection
- Role-based access control (super_admin, admin, vip, user)

**License Management**

- License key generation with customizable formats
- Activation tracking and expiry management
- Bulk license generation for administrators
- License history and audit trails

**Secure File Distribution**

- Encrypted file storage (AES-256-GCM)
- Version management with activation controls
- Download tracking and analytics
- Per-user encryption keys for secure delivery

**Monitoring & Analytics**

- Real-time dashboard with user statistics
- Session tracking and heartbeat monitoring
- Device analytics and comparison tools
- Comprehensive audit logging

**Security Features**

- Redis-based rate limiting (token bucket algorithm)
- IP blocklist middleware
- Request audit logging with geolocation
- Maintenance mode controls

### Database Schema (32 Models)

```
Users ──┬── Sessions
        ├── Licenses
        ├── Devices (Multi-device)
        ├── Audit Logs
        ├── Heartbeat Logs
        └── Download History
```

### API Endpoints (31+ Routers)

- Authentication (web + desktop client)
- User management (CRUD + ban/unban)
- License management (generate, activate, revoke)
- File management (upload, download, version control)
- Device management (register, remove, analytics)
- Dashboard analytics
- Audit logs with filtering and export
- Telegram integration
- OCR processing

### Code Metrics

| Metric          | Value                  |
| --------------- | ---------------------- |
| Source Files    | 80+ Python files       |
| Database Models | 32 SQLAlchemy models   |
| API Routers     | 31 endpoint modules    |
| Services        | 9 specialized services |
| Migrations      | 20+ Alembic versions   |

---

## Component 2: Admin Dashboard & User Portal

### Tech Stack

| Technology            | Purpose                               |
| --------------------- | ------------------------------------- |
| **Next.js 14**        | React framework (App Router)          |
| **TypeScript 5**      | Type-safe development                 |
| **TanStack Query v5** | Server state management               |
| **TanStack Table v8** | Data table handling                   |
| **Tailwind CSS 3.4**  | Utility-first styling                 |
| **shadcn/ui**         | Component library (Radix UI)          |
| **React Hook Form**   | Form management                       |
| **Zod**               | Schema validation                     |
| **Framer Motion**     | Animations                            |
| **i18next**           | Internationalization (Korean/English) |

### Key Features

**Admin Dashboard**

- Real-time statistics (users, sessions, licenses)
- Registration trend charts (30-day)
- License expiration analytics
- Recent activity feed
- Quick action widgets

**User Management**

- Paginated user list with search/filter
- User detail view with activity timeline
- Ban/unban with reason tracking
- HWID reset management
- Role assignment (with permission checks)

**License Management**

- License key generation interface
- Activation status tracking
- Expiry management
- Batch operations

**Device Management**

- Multi-device tracking per user
- Device comparison analytics
- Timeline view of device changes
- Removal quota management

**User Portal**

- Personal dashboard with usage stats
- Device management (add/remove)
- License information display
- Download history
- Session management
- Telegram integration setup

**Security & UX**

- JWT token refresh (proactive, 2 min before expiry)
- Session cookie authentication
- Role-based UI visibility
- Mobile-responsive design
- Dark/light theme support

### Code Metrics

| Metric              | Value                |
| ------------------- | -------------------- |
| Components          | 129 React components |
| Custom Hooks        | 30 hooks             |
| Type Definitions    | 22 type files        |
| Pages (Dashboard)   | 22 pages             |
| Pages (User Portal) | 20 pages             |
| Lines of Code       | ~15,000+             |

---

## Component 3: Windows Desktop Client

### Tech Stack

| Technology             | Purpose                |
| ---------------------- | ---------------------- |
| **C++17**              | Primary language       |
| **Visual Studio 2022** | Build system           |
| **DirectX 11**         | GUI rendering          |
| **ImGui**              | Immediate mode GUI     |
| **WinHTTP**            | HTTPS communication    |
| **AES-256-GCM**        | File decryption        |
| **ECDSA P-256**        | Signature verification |
| **HKDF-SHA256**        | Key derivation         |

### Key Features

**Authentication & Licensing**

- JWT-based authentication with backend API
- Multi-device support with device registration
- License expiration checking
- Automatic token refresh
- Secure credential storage via Windows Credential Manager

**Secure File Delivery**

- Encrypted file download from server
- In-memory decryption (no plaintext on disk)
- HKDF-derived keys unique per user
- Integrity verification via authentication tags

**Security Features**

- HWID-based device binding
- ECDSA signature verification for updates
- Secure communication over HTTPS
- Token-based session management

**User Interface**

- Modern dark-themed ImGui interface
- Login screen with credential storage
- Download progress display
- Status and log viewer
- Connection status indicators

### Code Metrics

| Metric               | Value                              |
| -------------------- | ---------------------------------- |
| Source Files         | 40+ C++ files                      |
| Lines of Code        | ~12,000                            |
| External Libraries   | 2 (ImGui, cryptographic libraries) |
| Build Configurations | 4 (Debug, Release, etc.)           |

---

## Component 4: Client Module (Plugin/Extension)

### Tech Stack

| Technology             | Purpose           |
| ---------------------- | ----------------- |
| **C++20**              | Primary language  |
| **Visual Studio 2022** | Build system      |
| **Dear ImGui**         | Overlay UI        |
| **Lua C API**          | Script execution  |
| **simdjson**           | JSON parsing      |
| **DirectX 11**         | Overlay rendering |

### Key Features

**License Validation**

- Encrypted authentication via shared memory
- Heartbeat validation (periodic license checks)
- Grace period handling for network errors
- Multi-device tracking

**Scripting Engine**

- Lua scripting support for extensibility
- Safe script execution with error handling
- Queue-based script management
- State validation and recovery

**Telegram Integration**

- Real-time notifications
- Command polling for remote control
- Status updates and alerts

**Monitoring & Logging**

- Comprehensive event logging
- Crash recovery with graceful degradation
- Session tracking
- Error reporting

### Code Metrics

| Metric                | Value                               |
| --------------------- | ----------------------------------- |
| Source Files          | 80+ C++ files                       |
| Lines of Code         | ~40,000                             |
| External Libraries    | 4 (ImGui, simdjson, Lua, stb_image) |
| Lua Functions Wrapped | 40+                                 |

---

## Data Flow Diagrams

### Authentication Flow (Web Dashboard)

```
┌──────────┐    POST /auth/login    ┌──────────┐
│  Browser │───────────────────────▶│  Backend │
│          │◀───────────────────────│  (JWT)   │
└──────────┘   Set-Cookie: session  └──────────┘
     │                                    │
     │         GET /auth/me               │
     │───────────────────────────────────▶│
     │◀───────────────────────────────────│
     │          User Data                 │
```

### Authentication Flow (Desktop Client)

```
┌──────────┐  POST /auth/client/login  ┌──────────┐
│  Desktop │────────────────────────▶  │  Backend │
│  Client  │   {username, password,    │          │
│          │    hwid}                  │          │
│          │◀────────────────────────  │          │
│          │  {access_token,           │          │
│          │   refresh_token,          │          │
│          │   device_id}              │          │
└──────────┘                           └──────────┘
     │                                       │
     │      GET /files/download (encrypted)  │
     │──────────────────────────────────────▶│
     │◀──────────────────────────────────────│
     │                                       │
     │        (Decrypt with HKDF key)        │
     ▼                                       │
┌──────────┐    POST /heartbeat         ┌────┴─────┐
│  Client  │────────────────────────────│  Backend │
│  Module  │◀───────────────────────────│          │
└──────────┘    {valid, license_info}   └──────────┘
```

### File Encryption Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    FILE ENCRYPTION FLOW                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [Admin Upload]                                                 │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────┐                                           │
│  │   Raw File      │                                           │
│  └────────┬────────┘                                           │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │  AES-256-GCM    │◀────│  Master Key     │                   │
│  │  Encryption     │     │  (Server-side)  │                   │
│  └────────┬────────┘     └─────────────────┘                   │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────────────────────────────────────────┐       │
│  │  Encrypted File = [12-byte nonce][ciphertext][tag]  │       │
│  └─────────────────────────────────────────────────────┘       │
│           │                                                     │
│           │  (Stored in database)                               │
│           ▼                                                     │
│  [User Download Request]                                        │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐     ┌─────────────────────────────────┐   │
│  │  HKDF-SHA256    │◀────│  User ID + Access Token         │   │
│  │  Key Derivation │     │  (Unique per user)              │   │
│  └────────┬────────┘     └─────────────────────────────────┘   │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                           │
│  │  Decrypt in     │                                           │
│  │  Client Memory  │                                           │
│  │  (Never on disk)│                                           │
│  └─────────────────┘                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Security Architecture

### Defense in Depth

```
┌─────────────────────────────────────────────────────────────────┐
│                    SECURITY LAYERS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Layer 1: Network Security                                      │
│  ├── HTTPS/TLS for all communications                          │
│  ├── Rate limiting (Redis token bucket)                        │
│  ├── IP blocklist middleware                                    │
│  └── CORS configuration                                         │
│                                                                 │
│  Layer 2: Authentication                                        │
│  ├── JWT tokens (15 min access, 7 day refresh)                 │
│  ├── HTTP-only session cookies                                  │
│  ├── HWID binding                                               │
│  └── Multi-device limits                                        │
│                                                                 │
│  Layer 3: Data Protection                                       │
│  ├── Argon2id password hashing                                  │
│  ├── AES-256-GCM file encryption                               │
│  ├── ECDSA signature verification                              │
│  └── HKDF key derivation                                        │
│                                                                 │
│  Layer 4: Application Security                                  │
│  ├── Input validation (Pydantic/Zod)                           │
│  ├── SQL injection prevention (parameterized queries)          │
│  ├── XSS prevention (React auto-escaping)                      │
│  └── CSRF protection                                            │
│                                                                 │
│  Layer 5: Client Security                                       │
│  ├── Secure credential storage (Windows Credential Manager)    │
│  ├── In-memory file decryption                                 │
│  ├── Token-based authentication                                │
│  └── Heartbeat license validation                              │
│                                                                 │
│  Layer 6: Monitoring & Audit                                    │
│  ├── Comprehensive audit logging                                │
│  ├── Heartbeat validation                                       │
│  ├── Session tracking                                           │
│  └── Security alert system                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Technical Highlights

### Backend Highlights

1. **100% Async Architecture**

   - AsyncIO throughout with non-blocking I/O
   - Async database sessions with SQLAlchemy 2.0
   - Background task scheduling with APScheduler
   - Batch audit log insertion for performance

2. **Dual Authentication System**

   - Session cookies for web dashboard
   - JWT tokens for desktop client
   - Automatic token refresh handling

3. **Multi-Device Management**
   - Per-user device limits
   - Monthly removal quotas
   - Device comparison analytics
   - Automatic device registration

### Frontend Highlights

1. **React Query State Management**

   - Automatic cache invalidation
   - Optimistic updates
   - Request deduplication
   - Background refetching

2. **Type-Safe Development**

   - Strict TypeScript configuration
   - Zod schema validation
   - 22 comprehensive type definition files

3. **Internationalization**
   - Korean and English support
   - Auto-detection from browser
   - Persistent language preference

### Desktop Client Highlights

1. **Secure File Delivery**

   - In-memory decryption only
   - HKDF-derived keys per user
   - No plaintext artifacts on disk

2. **Modern GUI**

   - ImGui-based interface
   - DirectX 11 rendering
   - Responsive design

3. **Robust Authentication**
   - Windows Credential Manager integration
   - Automatic token refresh
   - Multi-device support

### Client Module Highlights

1. **Scripting Engine**

   - Lua scripting for extensibility
   - Safe execution with error handling
   - Queue-based script management

2. **Crash Recovery**

   - Top-level exception handling
   - Graceful state recovery
   - Comprehensive error logging

3. **Telegram Integration**
   - Real-time notifications
   - Remote command support
   - Status monitoring

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    DOCKER COMPOSE STACK                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   PostgreSQL    │  │     Redis       │  │    Backend      │ │
│  │   Port: 5433    │  │   Port: 6379    │  │   Port: 8000    │ │
│  │   Primary DB    │  │   Sessions &    │  │   FastAPI       │ │
│  │                 │  │   Rate Limits   │  │   Uvicorn       │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│           │                    │                    │           │
│           └────────────────────┼────────────────────┘           │
│                                │                                │
│                    ┌───────────┴───────────┐                   │
│                    │     Docker Network    │                   │
│                    │     (Internal)        │                   │
│                    └───────────────────────┘                   │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐                     │
│  │    Frontend     │  │    pgAdmin      │                     │
│  │   Port: 3000    │  │   Port: 5050    │                     │
│  │   Next.js       │  │   (Optional)    │                     │
│  └─────────────────┘  └─────────────────┘                     │
│                                                                 │
│  Volumes:                                                       │
│  ├── postgres_data (persistent)                                │
│  ├── redis_data (persistent)                                   │
│  └── file_storage (local storage)                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Skills Demonstrated

### Backend Development

- Async Python programming
- RESTful API design
- Database modeling (relational)
- ORM usage (SQLAlchemy)
- Authentication & authorization
- Rate limiting & caching
- Background job processing
- Docker containerization

### Frontend Development

- React/Next.js development
- TypeScript type safety
- State management (React Query)
- Form handling & validation
- Responsive design
- Internationalization
- Component architecture
- Modern CSS (Tailwind)

### Desktop Application Development

- C++ application development
- Windows API programming
- GUI development (ImGui/DirectX)
- Secure communication (HTTPS)
- Cryptography implementation
- Multi-threading
- Memory management

### Security

- Cryptographic protocols (AES, ECDSA, HKDF)
- Authentication systems (JWT, sessions)
- Secure software distribution
- Hardware-based licensing
- Audit logging

### DevOps

- Docker & Docker Compose
- Database migrations
- Environment management
- Multi-service orchestration

---

## Lines of Code Summary

| Component      | Language         | LOC          |
| -------------- | ---------------- | ------------ |
| Backend        | Python           | ~15,000      |
| Frontend       | TypeScript/React | ~15,000      |
| Desktop Client | C++              | ~12,000      |
| Client Module  | C++              | ~40,000      |
| **Total**      | **Mixed**        | **~82,000+** |

---

## Key Technical Decisions

### Why FastAPI?

- Native async support for high concurrency
- Automatic OpenAPI documentation
- Pydantic validation integration
- Python ecosystem for rapid development

### Why Next.js 14?

- App Router for improved routing
- Server components capability
- Built-in optimizations
- Strong TypeScript support

### Why C++ for Desktop Components?

- Direct Windows API access
- Native performance
- No runtime dependencies
- Full control over memory and resources

### Why PostgreSQL + Redis?

- PostgreSQL: ACID compliance for critical data
- Redis: Low-latency session storage
- Both: Proven reliability at scale

---

## Future Enhancements

- Real-time WebSocket notifications
- Two-factor authentication
- API key management for third-party integrations
- Advanced analytics dashboard
- Automated testing pipeline
- Kubernetes deployment option
- Load balancing configuration

---

## Project Repository Structure

```
project/
├── backend/           # FastAPI backend (Python)
├── frontend/          # Next.js dashboard (TypeScript)
├── desktop-client/    # Windows client (C++)
├── client-module/     # Client module (C++)
└── docs/              # Documentation
```

---

_This project demonstrates full-stack development capabilities across web technologies and desktop application development, with a focus on security, performance, and user experience._
