# Wapenmanda Voting Tracking System (WVTS) — Step-by-Step Technical Implementation Guide

This repository now contains the implementation guide for building the **Wapenmanda Voting Tracking System (WVTS)**.

## Access Model (Applies to All User Types)
- All users (general users, candidates, and national census team members) can access the platform through:
  - **QR code entry** to open login/register routes,
  - direct **URL links**, or
  - both methods.
- Authentication supports password login, with role-specific flows (including multi-factor for census team).

---

## Phase 1: Project Initialization and Environment Setup

1. **Set Up Version Control and Repository**
   - Create a central Git repository.
   - Use branches:
     - `main` for production
     - `develop` for integration
     - `feature/*` for component work

2. **Configure the Development Environment**
   - Provision local/cloud dev machine.
   - Install runtime (Node.js or Python) and package manager.

3. **Establish the Database Server**
   - Provision PostgreSQL (local/cloud).
   - Create WVTS database and store credentials securely.

4. **Create the Project Directory Structure**
   - Create:
     - `frontend/`
     - `backend/`
     - `database/`

5. **Initialize the Backend Project**
   - In `backend/`, initialize package/project metadata.
   - Set clear package name (e.g., `wvts-backend`).

6. **Install Core Backend Dependencies**
   - Install:
     - Web framework (API)
     - PostgreSQL driver/ORM
     - Password hashing library
     - JWT library
     - WebAuthn server library

7. **Initialize the Frontend Project**
   - In `frontend/`, scaffold React (or chosen framework) app.

8. **Install Frontend Dependencies**
   - Install routing, HTTP client, and state management (if needed).

9. **Configure Environment Variables**
   - Create backend `.env` with placeholders:
     - `DATABASE_URL`
     - `JWT_SECRET`
     - `SMS_API_KEY`
     - `WEBAUTHN_RP_ID`
     - `SENDER_EMAIL`
     - `SECURITY_ALERT_EMAIL`

10. **Set Up Environment Variable Loading**
    - Load environment variables at app startup.
    - Add `.env` to `.gitignore`.

---

## Phase 2: Database Implementation and Seeding

11. **Write the Schema Migration Script**
    - Add full DDL migration in `database/`.

12. **Create Roles Table**
    - `roles(id, role_name UNIQUE)`.
    - Seed: `user`, `candidate`, `census_team`, `government_authority`.

13. **Create Districts, LLGs, and Wards Tables**
    - `districts`
    - `llgs` → FK to `districts`
    - `wards` → FK to `llgs`

14. **Create Users Table**
    - Include identity, contact, ward FK, role FK, password hash, timestamps.

15. **Create Candidates Table**
    - PK/FK to `users`, candidate number, photo URL.

16. **Create Candidate-Wards Junction Table**
    - Candidate ↔ ward mapping with unique pair constraint.

17. **Create Followership Table**
    - Tracks user follow selection per candidate/ward, active flag, timestamps.

18. **Create Notifications Table**
    - Candidate recipient, optional triggering user, message body, type enum, read flag.

19. **Create Census Team Members Table**
    - PK/FK to `users`, employee ID, department, three-factor authentication (3FA) flags/timestamps.

20. **Create Census Audit Log Table**
    - Immutable record of ward population updates.

21. **Create Security Violations Log Table**
    - Stores denied attempts, endpoint, IP, timestamp.

22. **Execute Migration Script**
    - Run migration against PostgreSQL and verify table schemas.

23. **Seed Reference Data**
    - 1 district (Wapenmanda), 2 LLGs (Tsak, Wapenmanda), 130+ wards with population `0`.

24. **Create Census Team Administrator Account**
    - Seed a census-team user with hashed password and 3FA enabled.

---

## Phase 3: Authentication and Authorization Middleware

25. **Build Password Hashing Utility**
    - `hashPassword(plain)` and `verifyPassword(plain, hash)`.

26. **Build JWT Utility**
    - `signToken({ userId, roleId })` and `verifyToken(token)`.

27. **Create Authentication Middleware**
    - Validate Bearer token; attach decoded user to request; reject `401` if invalid.

28. **Create Role-Based Authorization Middleware Factory**
    - `authorize(...roles)` middleware returns `403` if role mismatch.

29. **Define Role Constants**
    - Central numeric role map to avoid magic numbers.

---

## Phase 4: Core API Implementation

30. **Set Up the Express Router**
    - Mount resource routers: auth, geography, user, candidate, census.

31. **Implement Public Registration Endpoint**
    - Validate uniqueness, hash password, create default user role.

32. **Implement Standard Login Endpoint**
    - Validate credentials and return JWT + profile.

33. **Implement Geography Endpoints**
    - District summary, LLG list, wards by LLG, candidates by ward.

34. **Implement Followership Logic**
    - Transactionally enforce one active follow per user and generate notifications.

35. **Implement Current Following Endpoint**
    - Return authenticated user’s active follow details.

36. **Implement Candidate Login Endpoint**
    - Allow only candidate role for candidate login path.

37. **Implement Candidate Dashboard Data Endpoint**
    - Aggregate follower/population metrics at ward, LLG, district levels.

38. **Implement Candidate Notifications Endpoint**
    - List notifications by candidate with optional read filter.

39. **Implement Notification Mark-As-Read Endpoint**
    - Verify ownership then set `is_read = true`.

---

## Phase 5: Census Portal and Three-Factor Authentication

40. **Set Up WebAuthn Server Configuration**
    - Configure RP ID/origin at startup.

41. **Build Temporary Token System for Multi-Step Login**
    - Use short-lived, single-use intermediate token/session.

42. **Implement Census Login Step 1: Password**
    - Validate employee ID + password, return temporary token.

43. **Implement SMS OTP Service**
    - Add service wrapper for Twilio (or equivalent).

44. **Implement Census Login Step 2: SMS OTP**
    - Generate 6-digit OTP, store hashed in Redis (short TTL), send SMS.

45. **Implement Census Login Step 3: SMS Verification**
    - Verify OTP hash/expiry; mark SMS factor complete.

46. **Implement WebAuthn Registration Options Endpoint**
    - Generate registration options/challenge for census official.

47. **Implement WebAuthn Registration Verification Endpoint**
    - Verify attestation; store credential ID/public key.

48. **Implement Census Login Step 4: Biometric Assertion Options**
    - Generate login assertion options for registered credential.

49. **Implement Biometric Assertion Verification**
    - Verify assertion; issue full JWT; update 3FA login timestamp.

50. **Implement Population Update Endpoint**
    - Require census role authorization.
    - Require valid recent 3FA within the configured window (for example via `CENSUS_3FA_VALIDITY_WINDOW_MINUTES`).
    - Write an audit log entry for each successful update.
    - Update ward population value and trigger recalculation/notifications.

51. **Implement Recalculation + Notification Trigger**
    - Recompute impacted candidate metrics and create census update notifications.

52. **Implement Census Audit Endpoint**
    - Read-only, filterable, paginated audit history.

---

## Phase 6: Security Enforcement and Logging

53. **Implement Security Violation Logger**
    - Log denied/invalid actions with user, endpoint, and IP.

54. **Integrate Violation Logging into Authorization Failures**
    - On `403`, write violation record before response.

55. **Implement Administrative Security Alert**
    - Send async security email for each logged violation.

56. **Implement IP-Based Rate Limiting**
    - Strict limits for login endpoints; very strict for OTP verification.

57. **Apply Temporary Blocks for Repeated Violations**
    - Block IP temporarily after threshold violations via Redis TTL key.

---

## Phase 7: Frontend Implementation

58. **Set Up User Authentication Context**
    - Store JWT/profile/role; expose login/logout; persist in session storage.

59. **Build Role-Based Route Protection**
    - Redirect unauthenticated users; block incorrect roles.

60. **Implement User Registration and Login Pages**
    - Build forms and connect to public auth APIs.

61. **Implement Geographical Hierarchy Browser**
    - Drill-down district → LLG → ward with authenticated calls.

62. **Implement Candidate List and Follow Interface**
    - Candidate cards + selection + follow action handling.

63. **Implement Candidate Login and Dashboard**
    - Candidate-only dashboard with hierarchy-level metrics.

64. **Implement Candidate Notifications Component**
    - Bell icon, unread badge, dropdown, mark-as-read behavior.

65. **Implement Census Multi-Step Login Flow**
    - Password → OTP → WebAuthn assertion state machine.

66. **Implement Census Population Update UI**
    - Ward selection, population update form, reason field, audit feed.

67. **Implement Census Biometric Registration UI**
    - Registration flow via `navigator.credentials.create` and verification endpoint.

---

## Phase 8: Integration, Testing, and Deployment

68. **Connect Frontend to Backend**
    - Centralize API base URL and auto-attach JWT on requests.

69. **Perform Unit Testing of Core Logic**
    - Test password hashing, JWT utility, and percentage calculations.

70. **Perform Integration Testing of APIs**
    - Test registration, login, follow/unfollow behavior, census 3FA flow, and violation logging paths.

---

## Recommended Implementation Notes
- Keep schema migrations idempotent and versioned.
- Use transactions for follow-switch and population-update critical flows.
- Store only hashed OTP values.
- Enforce least privilege through middleware composition.
- Keep all secrets out of source control.
