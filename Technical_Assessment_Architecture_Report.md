# Setu Alumni Portal — Technical Assessment & Architecture Report
**Analyst:** Antigravity (Senior Software Analyst & Architect)  
**Date:** 28 April 2026  
**Repository:** `c:\mini-project`

---

## 1. Executive Summary

**Setu** is an alumni networking and communication platform designed for L.N.D. College, built upon a Node.js/Express and MongoDB backend with a Vanilla HTML/JS frontend. It enables alumni to register via their college IDs, discover peers through a searchable directory, and communicate via real-time Socket.io text chat and WebRTC audio/video calls. While the core CRUD APIs and messaging infrastructure are functional, the codebase remains in a prototype state. Critical security vulnerabilities (plain-text admin credentials, hardcoded production secrets), a stateful real-time architecture, and heavily advertised but missing backend features (job boards, mentorship) require immediate remediation before the system is viable for its target AWS EC2 deployment.

---

## 2. System Understanding

The application serves two primary actors: **Alumni** (who seek to network, find/offer mentorship, and communicate peer-to-peer) and **Administrators** (who manage user access and broadcast college events). The platform eschews modern frontend frameworks in favor of multi-page HTML connected by modular JS files. The backend utilizes Express 5 to expose RESTful APIs for state management, while relying on a monolithic Socket.io instance to mediate all transient state—such as user online status, chat delivery, and WebRTC peer connection signaling.

---

## 3. Problem Statement

Graduates of L.N.D. College currently lack a dedicated, institution-backed digital platform to maintain professional connections, mentor students, and receive campus updates. This forces reliance on fragmented social media channels, leading to a loss of institutional memory, weak alumni engagement for college initiatives, and missed career networking opportunities. By providing a secure, centralized directory intertwined with real-time communication tools, Setu bridges this gap, enabling the institution to re-engage its diaspora and unlock a self-sustaining network for professional advancement.

---

## 4. Objectives

| # | Objective | Type | Measurable Success Criteria |
|---|---|---|---|
| **O1** | Enable secure alumni authentication and session management. | Functional | The `/api/auth/login` endpoint issues a valid JWT; protected REST routes return HTTP 401 when a token is absent. |
| **O2** | Facilitate real-time 1:1 text and media communication. | Functional | `chatSocket.js` routes a message from sender to receiver and persists it in MongoDB within 500ms. |
| **O3** | Support peer-to-peer audio/video calling across strict networks. | Functional | WebRTC connection state reaches `connected` for >95% of attempts, leveraging STUN and properly configured TURN servers. |
| **O4** | Secure all administrative actions and credentials. | Security | The Admin database schema utilizes bcrypt for hashing; all mutating event APIs enforce admin JWT validation. |
| **O5** | Ensure safe handling of file attachments. | Security | The `/api/upload` endpoint restricts uploads to predefined MIME types and enforces a 5MB file size limit. |
| **O6** | Eliminate scaling blockers within the real-time communication tier. | Architecture | `userSockets` and `activeCallMap` are migrated from in-memory JavaScript objects to a distributed store (e.g., Redis). |
| **O7** | Establish environmental isolation between local dev and production. | Operational | Frontend network requests utilize dynamic relative paths (`/api`) rather than hardcoded EC2 IPs (`13.51.109.51`). |

---

## 5. Architecture & Data Flow Summary

**Architecture Topology:**
- **Frontend Layer:** Vanilla HTML/CSS/JS deployed alongside the backend. Data is fetched asynchronously via `fetch`.
- **API Layer:** Node.js (v22) + Express 5 providing JSON REST endpoints.
- **Real-Time Layer:** A central `socket.io` server (`chatSocket.js`) handling WebSockets for chat, typing indicators, reactions, and WebRTC signaling.
- **Data Layer:** MongoDB Atlas accessed via Mongoose.
- **Deployment:** AWS EC2 instance running NGINX as a reverse proxy (to terminate HTTPS for WebRTC compliance).

**Key Workflows:**
- **Authentication Flow:** User submits registration number → `authController` hashes password (bcrypt) and saves to DB. On login → compares hash, generates JWT (1-day expiry) → returns in response body and HTTP-only cookie.
- **Chat Data Flow:** Sender emits `message` to socket → server persists message to `Message` collection → server emits `message:received` to recipient's socket if active (tracking via in-memory `userSockets` map).
- **WebRTC Call Flow:** Caller emits `call-user` to socket → server relays `incoming-call` to recipient. On acceptance, caller creates WebRTC offer → server relays offer/answer/ICE candidates. Media flows peer-to-peer (or via TURN if NAT is symmetric).

---

## 6. Implementation Status

### Completed
- Alumni registration and JWT-based login controllers.
- Core REST APIs for fetching alumni lists, managing profiles, and admin event CRUD operations.
- 1:1 Real-time messaging infrastructure (Socket.io) with typing indicators, reactions, and database persistence.
- Basic WebRTC signaling implementation for audio/video calling.

### In Progress
- **WebRTC Network Reliability:** The signaling logic is built, but the `.env` file lacks a `TURN_SERVER_URL` value, meaning calls will fail across strict NAT environments.
- **Frontend Modularization:** Refactoring of chat and dashboard UI is underway, though files like `pages/chating.js` highlight ongoing structural adjustments.
- **File Uploads:** The Multer endpoint (`/api/upload`) functions but lacks security guardrails (size limits, file type validation).

### Missing
- **Job Board & Mentorship Hub:** Prominently featured on the frontend landing page, but entirely absent from backend models and routes.
- **Session Refresh Mechanism:** JWT sessions expire abruptly after 1 day with no silent refresh flow implemented.
- **Automated Testing & CI/CD:** Only a single happy-path E2E script (`tests/e2e.js`) exists. Unit tests and deployment pipelines are undefined.
- **Password Reset Flow:** No "Forgot Password" capability or email integration exists.

---

## 7. Technical Gaps, Risks, and Constraints

- **🔴 Critical Security Risk:** Admin passwords are authenticated using plain-text equality (`===` in `Admin.js`), exposing the system to severe compromise if the database is read.
- **🔴 Critical Security Risk:** Production secrets (MongoDB Atlas URI with credentials, `JWT_SECRET`) are hardcoded directly into the `.env` file committed to version control.
- **🟠 High Architecture Constraint:** `chatSocket.js` relies on in-memory `Map` objects to track connected users and active calls. Any server restart or horizontal scaling to multiple Node instances will drop all active calls and clear presence indicators.
- **🟠 High Security Risk:** Open CORS configuration (`callback(null, true)`) in the Express production setup leaves APIs vulnerable to cross-origin exploitation.
- **🟡 Medium Constraint:** Hardcoded EC2 IP addresses (`13.51.109.51`) in frontend files (like `dashboard.js`) break local development environments without an internet connection.

---

## 8. Open Questions

1. **Session Expiry Rules:** Should users be forced to log in daily, or is a long-lived refresh token mechanism desired for improved UX?
2. **File Storage Policy:** What is the exact retention policy for Multer-handled file uploads to prevent EBS volume exhaustion on the EC2 instance?
3. **Data Privacy Bounds:** Does the alumni directory require privacy toggles, or is it acceptable for all registered alumni to view each other's full profiles (including email and phone number)?
4. **Marketing Consistency:** Should the un-implemented Jobs and Mentorship sections be hidden from the landing page UI until their backend architectures are developed?

---

## 9. Prioritized Next Steps

### NOW (Critical Path to Production Viability)
1. **Remediate Admin Security:** Refactor `Admin.js` to hash and compare passwords using `bcrypt`. Implement a database migration script to secure existing admin accounts.
2. **Secret Rotation:** Revoke the exposed MongoDB URI, generate a new `JWT_SECRET`, and securely inject environment variables on the EC2 host (ensuring `.env` is removed from Git tracking).
3. **Fix Network Configurations:** Enforce explicit CORS origins in Express. Update frontend `API_BASE` definitions to utilize relative paths instead of hardcoded IPs.

### NEXT (Stabilization & Core Reliability)
4. **Configure TURN Server:** Integrate a managed TURN service (e.g., Twilio or Metered) and populate `.env` variables to ensure WebRTC reliability across varied NAT environments.
5. **Secure File Uploads:** Update the Multer configuration to enforce strict `limits.fileSize` (e.g., 5MB) and `fileFilter` MIME type validations.
6. **Expand Test Coverage:** Implement Jest/Supertest unit tests for authentication failure paths and socket connectivity limits.

### LATER (Feature Expansion & Scaling)
7. **Stateless Architecture:** Migrate `userSockets` and `activeCallMap` to a Redis datastore to enable deployment resilience and horizontal scaling.
8. **Build Missing Features:** Design the database schemas and REST controllers for the Jobs and Mentorship modules.
9. **Implement Refresh Tokens:** Upgrade the authentication architecture to support secure, long-lived sessions using HttpOnly refresh cookies.
