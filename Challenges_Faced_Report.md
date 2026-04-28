# Setu Alumni Portal — Challenges Faced Report
**Reviewer:** Antigravity (Technical Project Reviewer)  
**Date:** 28 April 2026  
**Repository:** `c:\mini-project`

---

## 1. Challenge Summary
During the development of the Setu Alumni Portal, the team encountered significant hurdles primarily revolving around transitioning from a local academic prototype to a cloud-ready production environment. Key challenges involved managing complex real-time connection reliability (WebRTC NAT traversal), securing sensitive data without proper infrastructure orchestration (hardcoded secrets and plain-text admin credentials), and managing scope creep, where frontend UI commitments (Jobs, Mentorship) far outpaced backend delivery capacity. These issues directly impacted the system's operational readiness and horizontal scalability.

---

## 2. Challenge Log

| Challenge | Category | Evidence | Impact | Resolution Status |
|---|---|---|---|---|
| **WebRTC Connection Failures across NATs** | Infrastructure | `.env` file leaves `TURN_SERVER_URL` empty, relying solely on STUN. | **High** (Peer-to-peer calls fail for users on strict corporate/mobile networks). | **Open** |
| **Insecure Admin Credential Storage** | Technical | `Admin.js` model utilizes plain-text string comparison (`===`) for authentication instead of hashing. | **High** (Critical security vulnerability if the database is read). | **Open** |
| **Exposure of Production Secrets** | Process | `JWT_SECRET` and MongoDB Atlas connection string are hardcoded in the committed `.env` file. | **High** (Immediate vector for unauthorized access). | **Open** |
| **Hardcoded Environment IPs** | Technical | Frontend scripts (`dashboard.js`, `admin.js`) hardcode the EC2 IP (`13.51.109.51`) for `API_BASE` resolution. | **Medium** (Breaks offline local development and complicates CI/CD). | **Open** |
| **Stateful Socket Architecture** | Technical | `chatSocket.js` utilizes in-memory JavaScript `Map` objects (`userSockets`, `activeCallMap`). | **High** (Prevents horizontal scaling of Node instances; server restarts drop all active calls). | **Open** |
| **Unimplemented Scope Commitments** | Scope | Frontend landing page features Job Board and Mentorship Hub UI, but no corresponding backend routes or MongoDB schemas exist. | **Medium** (Creates a disjointed UX and sets false expectations). | **Open** |
| **Misnamed/Misplaced Code Assets** | Technical | `pages/chating.html` contains the `ChatSecurity` JavaScript class rather than HTML markup. | **Medium** (Results in silently failing security/notification logic). | **Open** |
| **File Upload Resource Exhaustion** | Infrastructure | `/api/upload` (Multer) lacks `limits.fileSize` and MIME-type validation. | **High** (Risk of EC2 storage exhaustion or malicious file execution). | **Open** |

---

## 3. Root Cause Analysis

**1. WebRTC Connection Failures across NATs**
- **Root Cause:** WebRTC relies on ICE candidates to traverse networks. While STUN servers discover public IPs, symmetric NATs (common in corporate/mobile networks) block direct peer connections. A TURN server (which relays media) is required as a fallback, but was not provisioned due to cost/complexity, leaving the `TURN_SERVER_URL` empty.

**2. Insecure Admin Credential Storage & Exposed Secrets**
- **Root Cause:** A lack of standardized security review prior to merging backend code. Development prioritized functional "happy-paths" (getting the Admin panel working quickly) over implementing standard cryptographic safeguards (bcrypt) and `.gitignore` environment variable isolation.

**3. Stateful Socket Architecture (In-Memory Maps)**
- **Root Cause:** The real-time chat implementation was designed for a single-node academic project rather than cloud deployment. Using simple `Map` objects is the easiest way to track socket IDs locally, but ignores the necessity of a distributed publish/subscribe message broker (like Redis) for multi-instance deployments.

**4. Unimplemented Scope Commitments**
- **Root Cause:** Asynchronous frontend and backend development lifecycles. The frontend UI was designed with aspirational features (Jobs, Mentorship) that were de-prioritized or abandoned during backend API development, resulting in "phantom" UI elements.

---

## 4. Mitigation Actions Taken

*Note: Based on repository evidence, the vast majority of critical challenges currently remain unmitigated. The application remains in a prototype state.*
- **CORS Relaxation:** To bypass local development origin errors, CORS was configured to accept all origins (`callback(null, true)`). *Note: While this mitigated local errors, it introduced a severe production security risk.*

---

## 5. Pending Risks from Unresolved Challenges

1. **System Compromise:** The exposed MongoDB URI in the `.env` file allows immediate external read/write access to the database. Coupled with plain-text admin passwords, the entire alumni dataset is at critical risk of exfiltration.
2. **Server Crashing / Storage Exhaustion:** Unrestricted Multer file uploads allow malicious actors to flood the EC2 instance's EBS volume, leading to an application crash.
3. **Loss of Service during Scaling:** If the EC2 instance load balances to a second Node process, connected users will experience dropped chats and failed calls because socket state is not shared between processes.

---

## 6. Lessons Learned

- **Security must be "Shift-Left":** Features like password hashing and `.gitignore` configuration for secrets cannot be treated as post-launch refactors. They must be established at project inception.
- **WebRTC requires infrastructure:** Peer-to-peer networking is rarely purely peer-to-peer in the real world. Real-time video projects must budget for and configure TURN servers from day one.
- **Frontend vs. Backend Alignment:** UI mockups should not dictate the `main` branch codebase until the backend data layer is ready to support them, preventing "phantom" features.

---

## 7. Preventive Recommendations for Next Iteration

1. **Implement Secret Scanning:** Add tools like `git-secrets` or GitHub Advanced Security to the repository to prevent `.env` files from being committed.
2. **Standardize Environment Variables:** Adopt a configuration library (e.g., `dotenv` combined with `config`) and use relative API paths in the frontend (`/api`) rather than hardcoded IPs, allowing seamless switching between local, staging, and production environments.
3. **Adopt Redis for Socket.io:** Utilize the `@socket.io/redis-adapter` to decouple real-time state from the Node.js process memory.
4. **Enforce Upload Limits:** Immediately patch `upload.js` to enforce a 5MB size limit and whitelist specific MIME types (e.g., `image/jpeg`, `application/pdf`).

---

## 8. Escalation Items

The following items require immediate decision from the Lead Developer/Stakeholders before the application can go live on the EC2 instance:

1. **TURN Server Budget:** We must either allocate budget for a managed TURN service (e.g., Metered/Twilio) or authorize DevOps time to self-host `coturn` on the EC2 instance. Without this, WebRTC calls will consistently fail for a subset of users.
2. **Secret Rotation Authorization:** Approval is required to immediately invalidate the current MongoDB Atlas cluster credentials and issue new ones, as the current keys are compromised in the git history.
3. **Scope Reduction Approval:** Should we temporarily remove the "Job Board" and "Mentorship" sections from the UI to avoid user confusion, or delay the launch to build the required backend logic?
