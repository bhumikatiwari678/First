# Setu Alumni Portal — Deep Analysis Report & Solution Architecture
**Analyst:** Antigravity (Senior Software Analyst & Solution Architect)  
**Date:** 28 April 2026  
**Repository:** `c:\mini-project`

---

## 1. Executive Summary

**Setu** is a full-stack, single-tenant alumni management and communication platform designed for L.N.D. College. Built with an Express.js backend, MongoDB database, and a Vanilla HTML/JS frontend, it enables alumni to register, update professional profiles, discover peers via a directory, and communicate through real-time chat and WebRTC audio/video calls. While the core CRUD and Socket.io messaging infrastructure is implemented, the repository exhibits the characteristics of an academic prototype. Critical security vulnerabilities (plain-text admin credentials, hardcoded JWT secrets, exposed database URIs), in-memory state constraints, and incomplete advertised features (jobs, mentorship) must be addressed before the system is viable for production deployment on AWS EC2.

---

## 2. System Understanding

### Purpose
To provide a self-hosted, centralized digital network that bridges the gap between an educational institution and its graduated alumni, fostering professional networking, direct peer-to-peer communication, and institutional engagement.

### Target Users
1. **Alumni (Primary):** Graduates who need to network, seek/offer professional opportunities, and stay informed about college events.
2. **Administrators (Secondary):** Institutional staff who manage user access and broadcast campus events.

### Core Features
- **Authentication:** Alumni signup (registration number validation) and login with JWT session tokens. Separate Admin login flow.
- **Profile & Directory:** CRUD operations for alumni profiles; searchable and filterable directory (by name, department, year).
- **Real-Time Communication:** 1:1 messaging via Socket.io with typing indicators, emoji reactions, reply threading, and file attachments (via Multer).
- **Audio/Video Calls:** Peer-to-peer WebRTC calling with server-mediated signaling (offers, answers, ICE candidates).
- **Event Management:** Admin-driven event creation/deletion, visible on the alumni dashboard.

### Current Architecture Snapshot
- **Frontend Layer:** Vanilla HTML, CSS, and JavaScript. Uses `fetch` for REST API calls and `socket.io-client` for real-time events. No framework (e.g., React/Vue) is used.
- **API & Application Layer:** Node.js v22 running Express 5. REST APIs handle state transfers (Auth, Profiles, Events), while a centralized Socket.io server (`chatSocket.js`) handles transient messaging and WebRTC signaling.
- **Data Layer:** MongoDB Atlas (multi-region replica set) accessed via Mongoose ODM.
- **Deployment Topology:** Target environment is an AWS EC2 instance (`13.51.109.51`), utilizing NGINX as a reverse proxy for HTTPS termination (required for WebRTC).

---

## 3. Problem Statement

Graduates of L.N.D. College currently lack a dedicated, institution-backed platform to maintain professional connections, mentor juniors, and receive campus updates, forcing them to rely on fragmented social media groups (e.g., WhatsApp, LinkedIn) or outdated spreadsheets. This fragmentation leads to a loss of institutional memory, missed career networking opportunities, and low alumni engagement for college initiatives. By providing a secure, centralized directory and real-time communication portal, the institution can re-engage its diaspora, unlocking a self-sustaining network of mentorship and professional advancement.

---

## 4. Objectives

| # | Objective | Type | Success Criteria |
|---|---|---|---|
| **O1** | Enable secure alumni onboarding and session management using college registration numbers. | Functional | `authController.register` successfully hashes passwords via bcrypt, and `authController.login` issues a valid 1-day JWT. |
| **O2** | Facilitate real-time 1:1 text and media communication between connected peers. | Functional | `chatSocket.js` processes and delivers a `message` payload to an online recipient's socket within 500ms, storing it in MongoDB. |
| **O3** | Support peer-to-peer audio/video calling across diverse network topologies. | Functional | WebRTC connection state reaches `connected` for >95% of attempts, leveraging STUN and configured TURN servers. |
| **O4** | Secure all sensitive administrative actions and administrative credential storage. | Security (Non-Functional) | `Admin.comparePassword` utilizes bcrypt rather than strict equality (`===`), and all `/api/events` mutating routes enforce admin JWT validation. |
| **O5** | Eliminate horizontal scaling blockers within the real-time communication tier. | Architecture (Non-Functional) | `userSockets` and `activeCallMap` are migrated from in-memory JavaScript `Map` objects to a distributed store (e.g., Redis). |
| **O6** | Ensure environmental configuration parity between local development and cloud production. | Operational (Business) | `API_BASE` in frontend scripts uses dynamic origin resolution (`/api`) rather than hardcoded EC2 IPs (`13.51.109.51`). |

---

## 5. Scope Definition

### In Scope
- Alumni self-registration and JWT-based authentication.
- Alumni profile management and searchable directory display.
- One-to-one real-time chat (text, reactions, replies, file uploads).
- WebRTC-based audio and video calling (signaling via Socket.io).
- Admin dashboard for user moderation and event bulletin management.
- Static file serving for user uploads (`/uploads`).

### Out of Scope
- **Job Board & Mentorship Hub:** Present in frontend landing page mockups but completely absent from backend models and routes.
- **Fundraising & Scholarships:** No payment gateways or donation tracking schemas exist.
- **Group Chat / Rooms:** Communication is strictly bounded to 1:1 direct messaging.
- **Email/SMS Notifications:** No integration with external notification providers (e.g., SendGrid, Twilio).
- **Automated Password Reset:** No "Forgot Password" email flow is implemented.

---

## 6. Risks, Constraints, and Dependencies

- **Critical Security Vulnerabilities:** 
  - Admin passwords are compared in plain-text (`candidatePassword === this.password` in `Admin.js`).
  - Production secrets (`JWT_SECRET`, MongoDB Atlas URI with credentials) are hardcoded and committed to version control in `.env`.
- **Architectural Constraint (Stateful Sockets):** `chatSocket.js` stores active sockets and call states in memory. If the Node.js process restarts or scales to multiple instances, all active calls drop and presence indicators break.
- **WebRTC NAT Traversal Risk:** The `.env` file indicates `TURN_SERVER_URL` is empty. Without a TURN server, WebRTC calls will fail symmetrically when both peers are behind strict NATs or corporate firewalls (~20-30% failure rate).
- **CORS Misconfiguration:** `app.js` logs invalid origins but ultimately permits them (`callback(null, true)`), negating CORS protection in production.
- **Silent Frontend Bug:** The file `pages/chating.html` actually contains a JavaScript class (`ChatSecurity`). This will not render as a UI and indicates a structural misplacement of code.

---

## 7. Gaps and Open Questions

1. **Authentication Expiry:** JWTs are issued for 1 day without a refresh token mechanism. *Question: Should users be abruptly logged out daily, or is a silent refresh flow required?*
2. **File Storage Cleanup:** The `/api/upload` endpoint uses Multer to write to local disk indefinitely. *Question: What is the retention policy for chat attachments, and how will disk exhaustion on the EC2 instance be prevented?*
3. **Data Privacy:** All registered alumni can view the full directory. *Question: Is explicit user consent required to list personal emails and phone numbers, or should privacy toggles be introduced?*
4. **Unimplemented Marketing Features:** The landing page heavily advertises Jobs, Mentorship, and City Chapters. *Question: Should these UI elements be hidden until backend support is built, or are they immediate roadmap priorities?*
5. **Testing Coverage:** Only a single happy-path E2E script (`tests/e2e.js`) exists. *Assumption: The project currently lacks automated CI/CD pipelines to enforce unit and integration test passing.*

---

## 8. Prioritized Next Actions

### NOW (Critical Path to Production Viability)
1. **Remediate Admin Security:** Rewrite `Admin.js` to hash and compare passwords using `bcrypt`. Create a migration script to update existing admin records in the database.
2. **Secret Rotation & Environment Isolation:** Revoke the exposed MongoDB URI, generate new database credentials, replace `JWT_SECRET`, and remove `.env` from Git tracking immediately.
3. **Fix CORS & Hardcoded IPs:** Restrict the Express CORS configuration to explicit origins. Update frontend scripts (`dashboard.js`, `admin.js`, `chat.js`) to use relative paths or dynamic origins instead of `http://13.51.109.51:5000`.
4. **Resolve `chating.html` Anomaly:** Rename the file to `.js` and ensure the `ChatSecurity` class is correctly imported into the actual chat UI.

### NEXT (Stabilization & Core Reliability)
5. **Configure TURN Server:** Provision a self-hosted `coturn` instance or integrate a managed service (e.g., Twilio/Metered) to ensure WebRTC reliability across varied network topologies.
6. **Implement Upload Limits:** Add file size limits (e.g., 5MB) and MIME-type validation to the Multer configuration in `upload.js` to prevent malicious payloads or disk exhaustion.
7. **Expand Test Suite:** Write Jest/Supertest unit tests covering authentication failures, missing fields, and basic Socket.io connection limits.

### LATER (Feature Expansion & Scaling)
8. **Stateless Sockets:** Migrate Socket.io state management to a Redis adapter to allow horizontal scaling of the Node.js backend.
9. **Build Missing Features:** Design schemas and controllers for the highly advertised Job Board and Mentorship features.
10. **Implement Refresh Tokens:** Upgrade the authentication flow to support long-lived sessions via secure HttpOnly refresh cookies paired with short-lived access tokens.
