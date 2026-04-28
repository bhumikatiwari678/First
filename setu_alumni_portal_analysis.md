# Setu Alumni Portal — Deep Analysis Report
**Analyst:** Antigravity (Senior Software Analyst)  
**Date:** 28 April 2026  
**Repository:** `c:\mini-project`

---

## 1. Project Summary

**Setu** (Sanskrit: सेतु — "bridge") is a full-stack alumni management and communication portal built for **L.N.D. College**. The system allows graduated alumni to register, log in, view their profile, browse a searchable alumni directory, participate in events, and communicate peer-to-peer via real-time chat with file attachments and WebRTC audio/video calls. A separate admin panel (protected by its own credential store) lets administrators manage user records and create/delete college events.

The project is composed of two self-contained Git repositories:
- **`alumni/alumni-backend/`** — Node.js 22 / Express 5 REST API server with Socket.io and MongoDB Atlas.
- **`frontend/`** — Vanilla HTML + CSS + JavaScript static single-page-style frontend, served by the same Express server.

It is deployed (or intended to be deployed) on an **AWS EC2 instance** (`13.51.109.51`) with NGINX as a reverse proxy, using HTTPS for WebRTC support.

The application is clearly an **academic mini-project** and not yet a production-grade system: JWT secrets are hardcoded, admin password comparison is plain-text, there are no automated tests beyond a single e2e script, and several advertised landing-page features (job board, mentorship, scholarships, city chapters) have **zero backend implementation**.

---

## 2. Current System Understanding

### 2.1 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENT (Browser)                         │
│  Landing Page  →  Login/Signup Modal  →  Dashboard          │
│  Dashboard  →  Alumni Directory / Events / Profile Edit      │
│  Dashboard  →  Chat (Socket.io client)                      │
│  Chat  →  WebRTC call (peer-to-peer via STUN/TURN)          │
│  Admin Login  →  Admin Panel (users + events CRUD)          │
└──────────────────────┬──────────────────────────────────────┘
                       │  HTTP / WebSocket (port 5000 / NGINX proxy)
┌──────────────────────▼──────────────────────────────────────┐
│            EXPRESS 5 SERVER  (app.js)                       │
│                                                             │
│  REST API Routes:                                           │
│    /api/auth        →  authController (register, login)     │
│    /api/alumni      →  alumniController (list all)          │
│    /api/users       →  users route (list for chat)          │
│    /api/messages    →  messages route (history)             │
│    /api/events      →  eventController (CRUD)               │
│    /api/admin       →  adminAuthController (admin login)    │
│    /api/upload      →  multer (file uploads)                │
│    /api/contacts    →  inline aggregate (recent contacts)   │
│                                                             │
│  WebSocket (Socket.io):                                     │
│    socket/chatSocket.js  →  join, message, read, typing,    │
│                             react-message, reply-message,   │
│                             call-user, offer/answer/ICE     │
│                                                             │
│  Static Files:  /uploads, /js, / (frontend)                 │
└──────────────────────┬──────────────────────────────────────┘
                       │  Mongoose ODM
┌──────────────────────▼──────────────────────────────────────┐
│          MongoDB Atlas (multi-region replica set)           │
│  Collections:  users, messages, events, admins              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Module Inventory

| Layer | Files | Responsibility |
|---|---|---|
| Models | `user.js`, `Message.js`, `event.js`, `Admin.js` | MongoDB schemas |
| Controllers | `authController.js`, `adminAuthController.js`, `alumniController.js` | Business logic |
| Routes | `authRoutes.js`, `alumni.js`, `users.js`, `messages.js`, `events.js`, `adminAuth.js`, `upload.js`, `adminUsers.js` | HTTP endpoints |
| Middleware | `auth.js` (JWT verify), `verifyToken.js` | Request guards |
| Socket | `chatSocket.js` (863 lines) | Real-time chat + WebRTC signalling |
| Frontend Pages | `index.html`, `dashboard.html`, `admin.html`, `chating.html`, `login.html`, `team.html`, `call-ui.html`, `test-incoming-call.html` | UI views |
| Frontend JS | `main.js`, `dashboard.js`, `admin.js`, `js/chat.js`, `pages/chating.js`, `js/webrtc-call.js`, `js/cursor.js`, `pages/utils.js` | Client-side logic |
| Tests | `tests/e2e.js` | Single upload/chat E2E script |
| Config | `.env` | Environment variables |

### 2.3 Data Flow — Key Scenarios

**Alumni Registration & Login**
1. User fills Sign Up modal on `index.html` → POST `/api/auth/register` → `authController.register` → bcrypt hash → save to MongoDB → 201.
2. Login → POST `/api/auth/login` → find by `regNumber` → bcrypt compare → sign JWT (1 day) → set `HttpOnly` cookie + return token in body → redirect to `dashboard.html`.

**Real-time Chat**
1. Chat page loads → Socket.io client connects to `ws://server/socket.io/`.
2. Client emits `join { userId }` → server maps socketId→userId, marks user online, sends last 100 messages as history.
3. Client emits `message` → server persists to MongoDB, emits `message:sent` to sender and `message:received` to receiver; if receiver online → status `delivered`.
4. Features: typing indicators (3s timeout), emoji reactions (5 types), reply threading, file attachments (Multer → `/uploads/`).

**WebRTC Calls**
1. Caller emits `call-user { to, from, video, audio }` → server checks online status & busy state → emits `incoming-call` to receiver.
2. Receiver accepts → emits `call-accepted` → caller initiates WebRTC offer.
3. Server relays `webrtc-offer`, `webrtc-answer`, `webrtc-ice-candidate` purely as signalling; actual media is peer-to-peer.
4. STUN: Google public servers. TURN: Optional (currently unconfigured — reduces call success in NAT environments).

**Admin Panel**
1. Admin logs in via `/pages/admin_login.html` → POST `/api/admin/login` → plain-text password compare → JWT.
2. Admin panel fetches users list, events; can delete users and create/delete events.

---

## 3. Problem Statement

**Colleges and universities lack a unified, self-hosted digital platform that allows alumni to reconnect with their institution and with each other after graduation.** L.N.D. College alumni currently have no structured way to update their professional profiles, discover what their batch-mates are doing, receive information about campus events, or communicate directly — forcing them to rely on fragmented WhatsApp groups or manually maintained spreadsheets. This absence of institutional memory erodes the relationship between the college and its graduates, deprives students of mentorship and career opportunities from senior alumni, and prevents the institution from mobilising alumni goodwill for scholarships, events, and placements. **Setu** directly addresses this gap: a centralized alumni management portal with authenticated profiles, a searchable directory, event announcements, real-time peer-to-peer messaging, and audio/video calling — all under a single domain operated by the college. When fully functional and hardened, it would enable the college to build a sustainable, self-managed alumni network that drives engagement, career support, and institutional fundraising without depending on third-party social networks.

---

## 4. Objectives

| # | Objective | Type | Evidence / Source |
|---|---|---|---|
| **O1** | The system shall allow a new alumni user to register using their college registration number and receive a JWT-authenticated session within 2 seconds under normal network conditions, validated by the `/api/auth/register` + `/api/auth/login` pipeline. | **Functional** | `authController.js`, `authRoutes.js` |
| **O2** | The chat module shall deliver a real-time text or attachment message to an online recipient and reflect `delivered` status in MongoDB within 500 ms of the sender's `message` socket event, as verified by the existing `e2e.js` test. | **Functional** | `chatSocket.js`, `Message.js`, `tests/e2e.js` |
| **O3** | The admin panel shall allow a logged-in administrator to create, view, and delete alumni events, with changes visible on the alumni dashboard within one page refresh — all operations guarded by a valid admin JWT. | **Functional** | `admin.js`, `events.js`, `eventController.js` |
| **O4** | The WebRTC calling module shall establish an audio/video call between two online users within 5 seconds of the receiver accepting the call, succeeding for at least 80% of sessions on the same network (STUN only) and 95% across NATs when a TURN server is configured. | **Functional** | `chatSocket.js` (signalling), `.env` TURN config, `webrtc-call.js` |
| **O5** | All REST API endpoints that operate on user-specific data (`/api/auth/user/me`, `/api/alumni`, `/api/messages`) shall reject unauthenticated requests with HTTP 401, enforced via the `authenticateUser` JWT middleware, with zero routes accessible without a valid token. | **Non-Functional (Security)** | `middlewares/auth.js`, routes |
| **O6** | The server shall sustain concurrent connections from at least 50 simultaneous Socket.io clients on the EC2 `t2/t3` instance without event processing latency exceeding 200 ms per message, measured via load test (currently absent — requires tooling such as Artillery). | **Non-Functional (Performance)** | `app.js`, EC2 deployment context |
| **O7** | The admin panel password verification shall be migrated from plain-text comparison (`comparePassword` in `Admin.js`) to bcrypt hash comparison, eliminating the critical plaintext credential risk before any public deployment. | **Non-Functional (Security)** | `Admin.js` line 29, `adminAuthController.js` |
| **O8** | The system shall provide an operational NGINX + HTTPS configuration on the EC2 host such that 100% of WebRTC calls are initiated over secure contexts (`https://`), ensuring browser WebRTC APIs function in production without `localhost` exemptions. | **Business** | `.env` (NOTE comment), conversation history |

---

## 5. Scope

### In Scope
- Alumni self-registration and JWT-authenticated login.
- Alumni profile view and update (company, position, bio, LinkedIn, email).
- Searchable and filterable alumni directory (by name, department, year, company).
- College event management (create / list / delete) — for both admin and alumni.
- Real-time one-to-one text chat with typing indicators, emoji reactions, reply threads, and file attachments.
- WebRTC peer-to-peer audio/video calls with server-side signalling.
- Admin panel (separate credential store): user list, event management.
- Server-side file upload via Multer, served as static files.
- Deployment on AWS EC2 behind NGINX reverse proxy.

### Out of Scope
- **Mentorship platform** — UI cards exist on landing page but no backend models or routes are implemented.
- **Jobs / Career Board** — Listed as a feature, no API or schema exists.
- **Giving & Scholarships** — Marketing copy only; no donation workflow.
- **City Chapters / Interest Groups / News Feed** — Marketing copy only.
- **Batch / group chat rooms** — Only 1:1 DMs are implemented.
- **Email notifications** (e.g., OTP, forgot password, event reminders) — No email service integrated.
- **Mobile app** — Web-only.
- **Analytics / reporting dashboard** — Admin sees raw counts only.
- **Payment integration** — Not present.
- **Content moderation** — No flagging, blocking, or reporting mechanism.

---

## 6. Key Risks and Constraints

| Risk | Severity | Detail |
|---|---|---|
| **Plain-text admin password** | 🔴 Critical | `Admin.js`'s `comparePassword` does a raw `===` string comparison. Any DB read exposes admin credentials. |
| **Hardcoded JWT secret** | 🔴 Critical | `.env` sets `JWT_SECRET=mysecretkey123` — a trivially guessable value exposed in version control. |
| **MongoDB credentials in `.env` committed to Git** | 🔴 Critical | Atlas username/password are in `.env` inside the versioned `alumni/` directory. |
| **`chating.html` contains JavaScript, not HTML** | 🟠 High | The file `pages/chating.html` has `.html` extension but its entire content is a JavaScript `ChatSecurity` class. It is likely misnamed and never loaded as HTML, causing the intended security/notification system to be silently skipped. |
| **No TURN server configured** | 🟠 High | `TURN_SERVER_URL` is blank; calls between users behind symmetric NAT (common on mobile/corporate networks) will fail ~20-30% of the time. |
| **CORS allows all origins in production** | 🟠 High | `app.js` lines 60-62 log unrecognised origins but then call `callback(null, true)` — effectively open CORS. |
| **Socket.io in-memory state** | 🟡 Medium | `userSockets` and `activeCallMap` are plain `Map` objects; any server restart loses all online presence and active call state. Incompatible with horizontal scaling. |
| **No refresh-token mechanism** | 🟡 Medium | JWT expires in 1 day. No refresh flow exists; users are silently logged out. |
| **`chatRoutes.js` is 0 bytes** | 🟡 Medium | The file exists in `/routes/` but contains no content; an import would fail silently or crash the server. |
| **Token stored in `localStorage`** | 🟡 Medium | Vulnerable to XSS. The cookie is also set but `secure` flag is conditional on `NODE_ENV === 'production'` — which is never set in `.env`. |
| **Single test — happy path only** | 🟡 Medium | `tests/e2e.js` covers only one scenario (attachment delivery). No unit tests, no auth tests, no failure path tests. |
| **`dashboard.js` hardcodes EC2 IP** | 🟡 Medium | `API_BASE` falls back to `http://13.51.109.51:5000` for localhost — a hardcoded external dependency that breaks offline development. |
| **Frontend README is empty** | 🟢 Low | `frontend/README.md` contains only `# frontend` — no setup instructions for contributors. |
| **`app.js.save` stale file in production path** | 🟢 Low | A backup copy of the server entry-point is committed; could cause confusion. |

---

## 7. Open Questions

1. **Institution scope**: Is Setu intended solely for L.N.D. College (inferred from `dashboard.js` and department lists) or designed to be multi-tenant (multiple colleges)?
2. **Who creates admin accounts?** There is an `admincreate.js` script but no documentation. Is admin seeding a one-time manual CLI operation?
3. **Data privacy / GDPR / IT Act compliance**: Do alumni consent to their data (name, regNumber, employment, email) being visible to all other logged-in alumni via the directory API?
4. **TURN server budget**: Cloud TURN services (Twilio, Metered) cost money. Is there budget, or will `coturn` be self-hosted on the EC2 instance?
5. **File upload limits and cleanup**: Multer has no configured size limits; uploaded files persist indefinitely in `/uploads/`. What is the storage strategy?
6. **`chatRoutes.js` intent**: Is this an abandoned file or a planned feature stub for REST-based chat history (supplementing the Socket.io approach)?
7. **Landing page statistics** (28,000 alumni, 140 countries): Are these aspirational mock numbers or should they be dynamically fetched? Currently hardcoded in HTML.
8. **Logout behaviour**: `dashboard.html`'s logout button navigates to `login.html` without calling `/api/auth/logout` to clear the cookie. Is cookie-clearing required?

---

## 8. Recommended Next Steps (Prioritized)

### 🔴 Immediate — Security Hardening (Before Any Public Access)

1. **Hash admin passwords**: Replace `Admin.comparePassword` plain-text compare with `bcrypt.compare`. Run a migration script to re-hash existing admin records.
2. **Rotate all secrets**: Replace `JWT_SECRET`, regenerate MongoDB credentials, and remove `.env` from version control (add to `.gitignore`, use environment injection on EC2).
3. **Restrict CORS**: Define an explicit allowlist (college domain + localhost) and remove the catch-all `callback(null, true)` fallback.
4. **Fix cookie security flags**: Set `secure: true` and `sameSite: 'Strict'` unconditionally in production; enforce `NODE_ENV=production` in the systemd service file.

### 🟠 High — Stability & Core Functionality

5. **Fix `chating.html`**: Rename it to `chating.js` or `chat-security.js` and load it via `<script>` tag in the correct chat HTML page. Verify the `ChatSecurity` class is actually running.
6. **Configure TURN server**: Install `coturn` on the EC2 instance or subscribe to Metered TURN; populate `.env` TURN variables. This is a prerequisite for reliable WebRTC in any non-LAN scenario.
7. **Delete or implement `chatRoutes.js`**: The 0-byte file is a broken require risk. Either implement REST history endpoints or delete the file.
8. **Replace hardcoded EC2 IP** in `dashboard.js` `API_BASE` with a relative path (`''`) or a build-time environment variable so that local development doesn't depend on the live server.

### 🟡 Medium — Quality & Completeness

9. **Expand test suite**: Add unit tests for `authController` (register duplicate, login wrong password) and `chatSocket` (message persistence, read receipts). Use Jest + Supertest. Target ≥60% coverage before deployment.
10. **Add file upload validation**: Configure Multer `limits.fileSize` (e.g., 10 MB) and `fileFilter` to accept only safe MIME types; add a cron job or TTL index to clean orphaned uploads.
11. **Implement refresh token flow**: Issue a short-lived access token (15 min) and a long-lived refresh token (7 days) stored in `HttpOnly` cookie; add `POST /api/auth/refresh` endpoint.
12. **Add `.gitignore` and `README.md`**: Document setup steps, environment variable keys (not values), and deployment procedure for both the backend and frontend repositories.

### 🟢 Lower Priority — Feature Completion

13. **Connect landing-page feature sections to real backend**: Jobs board, mentorship, and news feed are either stubs or absent. Decide which to implement for v1.0 and create corresponding models + routes.
14. **Implement proper logout**: Call `/api/auth/logout` (which clears the cookie) on logout button click, then redirect.
15. **Real-time event updates**: Push new/deleted events to all connected dashboards via a Socket.io `events:updated` broadcast from the admin panel, eliminating the need for manual page refresh.
16. **Persistent socket state with Redis**: Replace in-memory `userSockets` Map with Redis Pub/Sub adapter (`socket.io-redis`) to support multi-process or multi-instance deployments.
