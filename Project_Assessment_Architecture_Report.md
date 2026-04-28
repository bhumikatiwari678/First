# Setu Alumni Portal — Project Assessment & Architecture Report
**Analyst:** Antigravity (Senior Software Analyst & Solution Architect)  
**Date:** 28 April 2026  
**Repository:** `c:\mini-project`

---

## 1. Executive Summary

**Setu** is a single-tenant alumni management and communication platform designed for L.N.D. College. Built upon a MERN-stack foundation (Express.js, MongoDB) combined with a Vanilla HTML/JS frontend, it enables alumni to register, maintain professional profiles, discover peers, and communicate via real-time socket-based messaging and WebRTC calls. While the foundational CRUD operations and real-time signaling are operational, the codebase heavily exhibits characteristics of an academic prototype. Critical security flaws (plain-text admin credentials, hardcoded production secrets), stateful scaling limitations, and partially implemented features must be resolved to elevate the system to a production-ready state for its AWS EC2 deployment target.

---

## 2. Project Understanding

**Architecture & Stack:**
- **Frontend Layer:** Vanilla HTML, CSS, and JS. Client-side routing is manual (multi-page). Dynamic data is fetched via REST endpoints, and real-time features utilize `socket.io-client`.
- **API & Application Layer:** Node.js v22 running Express 5. REST APIs manage state (Auth, Profiles, Events). A monolithic Socket.io implementation (`chatSocket.js`) handles real-time chat, reactions, replies, and WebRTC signaling (offers/answers/ICE candidates).
- **Data Layer:** MongoDB Atlas (accessed via Mongoose ODM).
- **Deployment Target:** AWS EC2 instance (`13.51.109.51`) behind an NGINX reverse proxy providing HTTPS (necessary for WebRTC functionality).

**Core Workflows & Data Flow:**
1. **User Onboarding:** Registration maps directly to a student's college registration number, storing hashed passwords. JWTs (1-day expiration) are issued on login and stored in cookies/localStorage.
2. **Directory & Profile:** Alumni data is queried to populate a searchable directory and individual profile management dashboards.
3. **Peer Communication:** Users engage in 1:1 messaging (text and Multer-handled file attachments) routed through Socket.io and persisted in MongoDB. WebRTC calls utilize server-mediated signaling to establish peer-to-peer media streams.
4. **Administration:** A separate admin login controls a dashboard for moderating user access and creating college-wide event bulletins.

---

## 3. Problem Statement

Graduates of L.N.D. College currently lack an institution-backed digital platform to maintain professional connections, mentor students, and receive campus updates. This forces reliance on fragmented, unverified social media channels, leading to a loss of institutional memory and weak alumni engagement for college initiatives. By providing a secure, centralized directory intertwined with real-time, peer-to-peer communication tools, Setu bridges this gap, enabling the institution to re-engage its diaspora and unlock a self-sustaining network for mentorship and career advancement.

---

## 4. Objectives

| # | Objective | Type | Measurable Criteria |
|---|---|---|---|
| **O1** | Enable secure alumni authentication and session management. | Functional | The `/api/auth/login` endpoint issues a valid JWT, and all protected REST routes return HTTP 401 when a token is absent or invalid. |
| **O2** | Facilitate real-time 1:1 text and media communication. | Functional | `chatSocket.js` successfully routes a message from sender to receiver and persists it in MongoDB within 500ms under normal network conditions. |
| **O3** | Support peer-to-peer audio/video calling across diverse network topologies. | Functional | WebRTC connection state reaches `connected` for >95% of call attempts, leveraging configured STUN/TURN servers. |
| **O4** | Secure all administrative actions and credentials. | Security | The Admin database schema utilizes bcrypt for password hashing, and all mutating event APIs enforce admin JWT validation. |
| **O5** | Ensure safe handling and storage of file attachments. | Security | The `/api/upload` endpoint restricts uploads to a predefined list of safe MIME types and enforces a maximum file size limit (e.g., 5MB). |
| **O6** | Eliminate scaling blockers within the real-time communication tier. | Architecture | `userSockets` and `activeCallMap` are migrated from in-memory JavaScript objects to a distributed datastore (e.g., Redis). |
| **O7** | Establish environmental isolation between local development and production. | Operational | Frontend network requests utilize dynamic relative paths (`/api`) rather than hardcoded EC2 IPs (`13.51.109.51`). |

---

## 5. Current Progress Assessment

### Completed
- Alumni registration and JWT-based login flows (frontend UI and backend controllers).
- Core REST APIs for fetching alumni lists and managing profiles.
- 1:1 Real-time messaging infrastructure via Socket.io (typing indicators, reactions, basic persistence).
- Basic WebRTC signaling implementation for audio/video calling.
- Administrator authentication (albeit insecure) and event CRUD operations.

### In Progress
- **WebRTC Network Reliability:** The signaling logic exists, but the `.env` `TURN_SERVER_URL` is empty, meaning calls will fail across strict NATs.
- **Frontend Modularization:** Chat UI and dashboard functionality are being actively refined, but files like `pages/chating.html` (which contains pure JavaScript) reflect structural anomalies.
- **File Uploads:** Multer endpoint exists, but lacks size limitations, MIME-type validation, and cleanup strategies.

### Not Started / Missing
- **Job Board & Mentorship Hub:** Prominently featured on the landing page, but entirely absent from backend models and routes.
- **Refresh Token Mechanism:** Sessions expire abruptly after 1 day with no silent refresh flow.
- **Automated Testing & CI/CD:** Only a single happy-path E2E script (`tests/e2e.js`) exists; no unit tests or deployment pipelines are defined.
- **Password Reset Flow:** No email integration or "Forgot Password" capability exists.

---

## 6. Technical Gaps and Risks

- **Critical Security:** Admin passwords are compared using plain-text equality (`===` in `Admin.js`).
- **Critical Security:** Production secrets (MongoDB Atlas URI, `JWT_SECRET`) are hardcoded directly into the committed `.env` file.
- **Security:** Open CORS configuration (`callback(null, true)`) in production leaves APIs vulnerable to cross-origin exploitation.
- **Stateful Sockets:** `chatSocket.js` relies on in-memory `Map` objects. Any server restart or horizontal scaling event will drop all active calls and clear presence indicators.
- **Frontend Bug:** `pages/chating.html` is misnamed and contains the `ChatSecurity` JavaScript class, meaning it will fail to render as a view or execute correctly in its current location.

---

## 7. Scope

### In Scope
- Alumni self-registration, authentication, and profile management.
- Searchable directory of registered alumni.
- One-to-one real-time text chat with media attachments.
- WebRTC-based audio/video calling.
- Admin dashboard for user moderation and college event announcements.

### Out of Scope
- **Job Board & Mentorship Platform:** (UI exists, but backend implementation is deferred).
- **Fundraising / Donations / Scholarships:** No payment gateways or tracking schemas.
- **Group Chat / Public Rooms:** Communication is strictly 1:1.
- **Automated Email/SMS Notifications:** No integration with external notification providers.

---

## 8. Open Questions

1. **Authentication Expiry:** Should users be forced to log in daily, or is a long-lived refresh token mechanism required for a better user experience?
2. **File Storage Policy:** What is the retention policy for Multer-handled file uploads to prevent disk exhaustion on the EC2 instance?
3. **Data Privacy:** Does the directory require privacy toggles, or is it acceptable for all registered alumni to view each other's full profiles (including email and phone number)?
4. **Landing Page Consistency:** Should the un-implemented Jobs and Mentorship sections be hidden from the UI until their backends are developed?

---

## 9. Prioritized Next Steps

### NOW (Critical Path to Production Viability)
1. **Remediate Admin Security:** Refactor `Admin.js` to hash and compare passwords using `bcrypt`. Implement a database migration to secure existing admin accounts.
2. **Secret Rotation:** Revoke the exposed MongoDB URI, generate a new `JWT_SECRET`, and securely inject environment variables on the EC2 host (removing `.env` from Git tracking).
3. **Fix Network Configuration:** Enforce strict CORS origins. Update frontend `API_BASE` variables to utilize relative paths instead of the hardcoded IP `13.51.109.51`.
4. **Resolve `chating.html`:** Rename the file to `.js` and verify the `ChatSecurity` class is correctly imported into the active chat UI.

### NEXT (Stabilization & Core Reliability)
5. **Configure TURN Server:** Integrate a managed TURN service (or provision `coturn`) and populate `.env` variables to ensure WebRTC reliability across varied NAT environments.
6. **Secure File Uploads:** Configure Multer with strict `limits.fileSize` and `fileFilter` validations.
7. **Expand Test Coverage:** Implement basic unit tests for authentication failure paths and socket connectivity limits.

### LATER (Feature Expansion & Scaling)
8. **Stateless Architecture:** Migrate `userSockets` and `activeCallMap` to a Redis instance to enable horizontal scaling and deployment resilience.
9. **Build Missing Features:** Design the database schemas and REST controllers for the Jobs and Mentorship modules.
10. **Implement Refresh Tokens:** Upgrade the authentication architecture to support secure, long-lived sessions using HttpOnly refresh cookies.
