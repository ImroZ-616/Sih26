# 🧠 CogniCare NER — API Contract

**Project:** CogniCare NER
**Hackathon:** Smart India Hackathon 2026
**Document Owner:** Product + System Architect
**Status:** Architecture Baseline — SIH 2026 MVP
**API Version:** v1

---

# 1. Purpose

This document defines the communication contract between the different CogniCare NER components.

The primary consumers are:

* Patient Mobile App
* Backend
* AI/Cognitive Engine
* Caregiver Dashboard
* Synchronization Layer

The purpose of this document is to ensure that all teams can develop their modules independently while maintaining compatible interfaces.

---

# 2. API Architecture

All backend APIs use the following conceptual base URL:

```text
/api/v1
```

Example:

```text
https://<server>/api/v1
```

The actual development URL may differ.

---

# 3. API Principles

The API should follow these principles:

1. REST-style HTTP endpoints
2. JSON request/response bodies
3. Token-based authentication
4. Role-based authorization
5. Consistent error responses
6. Idempotent synchronization
7. Versioned API paths
8. Input validation
9. Minimal data exposure
10. Secure HTTPS communication in deployed environments

---

# 4. Authentication

Protected endpoints require an authentication token.

Example:

```http
Authorization: Bearer <access_token>
```

The exact authentication provider may be selected during implementation.

---

# 5. User Roles

The MVP supports two primary roles:

```text
PATIENT
CAREGIVER
```

A user must only access resources permitted by their role and authorization state.

---

# 6. Standard Response Format

Successful responses should use a consistent structure where practical.

Example:

```json
{
  "success": true,
  "data": {}
}
```

Error responses:

```json
{
  "success": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "The request is invalid."
  }
}
```

The exact implementation may vary slightly, but the meaning of the fields should remain consistent.

---

# 7. Standard HTTP Status Codes

| Status | Meaning                                  |
| ------ | ---------------------------------------- |
| 200    | Successful request                       |
| 201    | Resource created                         |
| 204    | Successful request with no response body |
| 400    | Invalid request                          |
| 401    | Authentication required/invalid          |
| 403    | Authenticated but not authorized         |
| 404    | Resource not found                       |
| 409    | Conflict                                 |
| 422    | Validation failure                       |
| 429    | Rate limited                             |
| 500    | Internal server error                    |
| 503    | Service unavailable                      |

---

# 8. Authentication Endpoints

## 8.1 Register

```http
POST /api/v1/auth/register
```

### Request

```json
{
  "name": "Example User",
  "email": "user@example.com",
  "password": "password",
  "role": "PATIENT"
}
```

### Response

```json
{
  "success": true,
  "data": {
    "user": {
      "id": "user-001",
      "name": "Example User",
      "role": "PATIENT"
    },
    "access_token": "<token>"
  }
}
```

---

# 9. Login

```http
POST /api/v1/auth/login
```

### Request

```json
{
  "email": "user@example.com",
  "password": "password"
}
```

### Response

```json
{
  "success": true,
  "data": {
    "user": {
      "id": "user-001",
      "name": "Example User",
      "role": "PATIENT"
    },
    "access_token": "<token>"
  }
}
```

---

# 10. Current User

```http
GET /api/v1/auth/me
```

### Authentication

Required.

### Response

```json
{
  "success": true,
  "data": {
    "id": "user-001",
    "name": "Example User",
    "role": "PATIENT"
  }
}
```

---

# 11. Patient Profile

## Get Patient

```http
GET /api/v1/patients/{patient_id}
```

### Authentication

Required.

### Authorization

The requesting user must be:

* The patient
* An authorized caregiver

### Response

```json
{
  "success": true,
  "data": {
    "id": "patient-001",
    "name": "Example Patient",
    "language": "en",
    "created_at": "2026-09-12T10:00:00Z"
  }
}
```

Only permitted information should be returned.

---

# 12. Game Catalog

```http
GET /api/v1/games
```

### Response

```json
{
  "success": true,
  "data": [
    {
      "id": "memory_match",
      "name": "Memory Match",
      "type": "MEMORY",
      "available": true
    },
    {
      "id": "routine_recall",
      "name": "Daily Routine Recall",
      "type": "SEQUENCE",
      "available": true
    },
    {
      "id": "pattern_recognition",
      "name": "Pattern Recognition",
      "type": "PATTERN",
      "available": true
    }
  ]
}
```

The patient app may cache the game catalog for offline operation.

---

# 13. Game Session

A game session represents one completed or in-progress game interaction.

## Create Game Session

```http
POST /api/v1/game-sessions
```

### Authentication

Required.

### Request

```json
{
  "client_session_id": "session-abc123",
  "patient_id": "patient-001",
  "game_id": "memory_match",
  "difficulty": 2,
  "score": 80,
  "accuracy": 0.80,
  "mistakes": 2,
  "attempts": 10,
  "completion_time_seconds": 42,
  "hints_used": 1,
  "started_at": "2026-09-12T10:20:00Z",
  "completed_at": "2026-09-12T10:20:42Z"
}
```

---

# 14. Game Session Response

```json
{
  "success": true,
  "data": {
    "session_id": "server-session-001",
    "client_session_id": "session-abc123",
    "status": "SYNCED"
  }
}
```

---

# 15. Idempotency

`client_session_id` must uniquely identify a logical client-created game session.

If the same session is uploaded more than once:

```text
First request
    ↓
Create session

Second request
    ↓
Recognize existing client_session_id
    ↓
Do NOT create duplicate
    ↓
Return existing session
```

This behavior is required for offline synchronization.

---

# 16. Get Game Sessions

```http
GET /api/v1/patients/{patient_id}/game-sessions
```

### Optional query parameters

```text
game_id
from
to
limit
cursor
```

Example:

```http
GET /api/v1/patients/patient-001/game-sessions?game_id=memory_match
```

### Response

```json
{
  "success": true,
  "data": {
    "sessions": [
      {
        "session_id": "server-session-001",
        "game_id": "memory_match",
        "difficulty": 2,
        "score": 80,
        "accuracy": 0.80,
        "mistakes": 2,
        "completion_time_seconds": 42,
        "completed_at": "2026-09-12T10:20:42Z"
      }
    ]
  }
}
```

---

# 17. Performance Summary

```http
GET /api/v1/patients/{patient_id}/performance
```

### Query parameters

```text
game_id
period
```

Example:

```http
GET /api/v1/patients/patient-001/performance?period=30d
```

### Response

```json
{
  "success": true,
  "data": {
    "patient_id": "patient-001",
    "period": "30d",
    "sessions_completed": 12,
    "average_accuracy": 0.78,
    "average_score": 76,
    "average_completion_time_seconds": 45
  }
}
```

---

# 18. Performance Trend

```http
GET /api/v1/patients/{patient_id}/trends
```

### Response

```json
{
  "success": true,
  "data": {
    "patient_id": "patient-001",
    "trend": "STABLE",
    "period": "30d",
    "message": "Performance has remained relatively stable over recent sessions."
  }
}
```

Allowed trend values:

```text
IMPROVING
STABLE
DECLINING
IRREGULAR
INSUFFICIENT_DATA
```

---

# 19. Important Medical Boundary

Trend information is an observation about application performance.

It must NOT be interpreted as a medical diagnosis.

Valid:

```text
"Performance has decreased over recent sessions."
```

Invalid:

```text
"Patient is developing Alzheimer's."
```

The backend and dashboard must preserve this distinction.

---

# 20. AI Cognitive Engine

The AI engine evaluates performance and recommends game difficulty.

## Analyze Performance

Conceptual endpoint:

```http
POST /api/v1/ai/analyze
```

This endpoint may be internal rather than publicly accessible.

### Request

```json
{
  "patient_id": "patient-001",
  "game_id": "memory_match",
  "current_difficulty": 2,
  "current_session": {
    "score": 80,
    "accuracy": 0.80,
    "mistakes": 2,
    "completion_time_seconds": 42
  },
  "recent_sessions": [
    {
      "accuracy": 0.82,
      "score": 81,
      "mistakes": 2
    },
    {
      "accuracy": 0.85,
      "score": 84,
      "mistakes": 1
    }
  ]
}
```

---

# 21. AI Response

```json
{
  "success": true,
  "data": {
    "recommended_difficulty": 3,
    "trend": "IMPROVING",
    "reason": "Recent performance has remained strong at the current difficulty."
  }
}
```

---

# 22. AI Output Constraints

The AI engine must produce:

* Difficulty recommendation
* Performance trend
* Explainable reason

It must NOT produce:

* Medical diagnosis
* Alzheimer's probability
* Dementia probability
* Medical treatment recommendations
* Medication recommendations

---

# 23. Difficulty State

```http
GET /api/v1/patients/{patient_id}/difficulty/{game_id}
```

### Response

```json
{
  "success": true,
  "data": {
    "game_id": "memory_match",
    "difficulty": 3,
    "updated_at": "2026-09-12T10:30:00Z"
  }
}
```

---

# 24. Update Difficulty

This may be an internal AI/backend operation.

```http
PUT /api/v1/patients/{patient_id}/difficulty/{game_id}
```

### Request

```json
{
  "difficulty": 3,
  "reason": "Strong recent performance"
}
```

### Response

```json
{
  "success": true,
  "data": {
    "game_id": "memory_match",
    "difficulty": 3
  }
}
```

---

# 25. Reminders

## Create Reminder

```http
POST /api/v1/reminders
```

### Request

```json
{
  "patient_id": "patient-001",
  "type": "MEDICINE",
  "title": "Morning Medicine",
  "scheduled_time": "08:00",
  "repeat": "DAILY",
  "voice_enabled": true
}
```

### Allowed types

```text
MEDICINE
HYDRATION
APPOINTMENT
```

### Response

```json
{
  "success": true,
  "data": {
    "id": "reminder-001",
    "patient_id": "patient-001",
    "type": "MEDICINE",
    "title": "Morning Medicine",
    "scheduled_time": "08:00",
    "repeat": "DAILY",
    "voice_enabled": true
  }
}
```

---

# 26. Get Reminders

```http
GET /api/v1/patients/{patient_id}/reminders
```

### Response

```json
{
  "success": true,
  "data": {
    "reminders": [
      {
        "id": "reminder-001",
        "type": "MEDICINE",
        "title": "Morning Medicine",
        "scheduled_time": "08:00",
        "repeat": "DAILY",
        "voice_enabled": true,
        "active": true
      }
    ]
  }
}
```

---

# 27. Update Reminder

```http
PUT /api/v1/reminders/{reminder_id}
```

### Request

```json
{
  "title": "Morning Medicine",
  "scheduled_time": "08:30",
  "active": true
}
```

---

# 28. Delete Reminder

```http
DELETE /api/v1/reminders/{reminder_id}
```

### Response

```text
204 No Content
```

---

# 29. Reminder Completion

```http
POST /api/v1/reminders/{reminder_id}/completions
```

### Request

```json
{
  "client_completion_id": "completion-123",
  "completed_at": "2026-09-12T08:05:00Z"
}
```

### Response

```json
{
  "success": true,
  "data": {
    "completion_id": "completion-123",
    "status": "SYNCED"
  }
}
```

The `client_completion_id` prevents duplicate completion records during offline synchronization.

---

# 30. Caregiver Linking

Caregiver access is consent-based.

## Create Consent Request

```http
POST /api/v1/consents
```

### Request

```json
{
  "patient_id": "patient-001",
  "caregiver_id": "caregiver-001"
}
```

### Response

```json
{
  "success": true,
  "data": {
    "consent_id": "consent-001",
    "status": "PENDING"
  }
}
```

---

# 31. Approve Consent

```http
PUT /api/v1/consents/{consent_id}
```

### Request

```json
{
  "status": "APPROVED"
}
```

### Response

```json
{
  "success": true,
  "data": {
    "consent_id": "consent-001",
    "status": "APPROVED"
  }
}
```

Allowed states:

```text
PENDING
APPROVED
REJECTED
REVOKED
```

---

# 32. Get Caregiver Patients

```http
GET /api/v1/caregiver/patients
```

### Authentication

Required.

### Role

CAREGIVER.

### Response

```json
{
  "success": true,
  "data": {
    "patients": [
      {
        "id": "patient-001",
        "name": "Example Patient",
        "consent_status": "APPROVED"
      }
    ]
  }
}
```

---

# 33. Sync API

Synchronization is one of the most important APIs in the platform.

```http
POST /api/v1/sync
```

The endpoint accepts locally generated operations.

---

# 34. Sync Request

```json
{
  "operations": [
    {
      "sync_id": "sync-001",
      "entity_type": "GAME_SESSION",
      "entity_id": "session-abc123",
      "operation": "CREATE",
      "payload": {
        "client_session_id": "session-abc123",
        "patient_id": "patient-001",
        "game_id": "memory_match",
        "difficulty": 2,
        "score": 80,
        "accuracy": 0.80,
        "mistakes": 2,
        "completion_time_seconds": 42
      },
      "created_at": "2026-09-12T10:30:00Z"
    }
  ]
}
```

---

# 35. Sync Response

```json
{
  "success": true,
  "data": {
    "results": [
      {
        "sync_id": "sync-001",
        "entity_id": "session-abc123",
        "status": "SYNCED"
      }
    ]
  }
}
```

Possible statuses:

```text
SYNCED
DUPLICATE
FAILED
REJECTED
```

---

# 36. Sync Idempotency

The backend must safely handle repeated operations.

Example:

```text
Request A
sync_id = sync-001
entity_id = session-abc123

Request A repeated

→ Do not create duplicate
→ Return existing result
```

The server may use:

```text
sync_id
```

and/or:

```text
client entity ID
```

to implement idempotency.

---

# 37. Sync Download

The patient application may also need to retrieve changes from the server.

Conceptual endpoint:

```http
GET /api/v1/sync?since=<cursor>
```

### Response

```json
{
  "success": true,
  "data": {
    "changes": [
      {
        "entity_type": "DIFFICULTY_STATE",
        "entity_id": "memory_match",
        "operation": "UPSERT",
        "updated_at": "2026-09-12T10:30:00Z",
        "payload": {
          "difficulty": 3
        }
      }
    ],
    "next_cursor": "cursor-002"
  }
}
```

---

# 38. Sync Cursor

The client may store a synchronization cursor.

Example:

```text
cursor-001
```

Next synchronization:

```text
GET /sync?since=cursor-001
```

The server returns changes after that point.

This avoids downloading the complete dataset repeatedly.

---

# 39. Last Sync Information

The backend may expose:

```http
GET /api/v1/sync/status
```

### Response

```json
{
  "success": true,
  "data": {
    "last_server_sync": "2026-09-12T10:42:00Z"
  }
}
```

The caregiver dashboard may use this information to show when the patient device last synchronized.

---

# 40. Caregiver Dashboard Data

The caregiver dashboard primarily consumes:

```text
GET /patients
GET /patients/{id}
GET /patients/{id}/game-sessions
GET /patients/{id}/performance
GET /patients/{id}/trends
GET /patients/{id}/reminders
GET /sync/status
```

The dashboard should not directly access the database.

---

# 41. Patient App Data Requirements

The patient app primarily consumes:

```text
POST /auth/login
GET /auth/me

GET /games

GET /patients/{id}
GET /patients/{id}/difficulty/{game}

POST /game-sessions

GET /patients/{id}/reminders

POST /reminders/{id}/completions

POST /sync
GET /sync
```

---

# 42. AI Engine Data Requirements

The AI engine requires:

```text
Game session data
Current difficulty
Recent performance
Historical performance where necessary
```

It returns:

```text
Recommended difficulty
Performance trend
Reason/explanation
```

---

# 43. Error Contract

All API errors should provide machine-readable and human-readable information.

Example:

```json
{
  "success": false,
  "error": {
    "code": "PATIENT_NOT_FOUND",
    "message": "The requested patient could not be found."
  }
}
```

Possible error codes:

```text
INVALID_REQUEST
VALIDATION_ERROR
UNAUTHORIZED
FORBIDDEN
PATIENT_NOT_FOUND
GAME_NOT_FOUND
REMINDER_NOT_FOUND
CONSENT_REQUIRED
DUPLICATE_OPERATION
SYNC_FAILED
INTERNAL_ERROR
```

---

# 44. Validation

Backend APIs must validate:

* Required fields
* Data types
* Allowed enum values
* Numeric ranges
* Date/time formats
* Patient ownership/authorization
* Consent status
* Authentication state

The backend must not trust validation performed only by the client.

---

# 45. Authorization Rules

## Patient

A patient can:

* Access their own profile
* Play games
* Submit their own sessions
* View their own progress
* View their reminders
* Complete reminders
* Manage permitted personal settings

## Caregiver

An authorized caregiver can:

* View explicitly linked patients
* View permitted performance data
* View trends
* View reminder status
* Manage permitted reminders

A caregiver cannot access arbitrary patient data.

---

# 46. API Security Rules

All protected endpoints require authentication.

Authorization must be checked server-side.

Never rely on:

```text
patient_id
```

provided by the client alone to determine access.

The backend must verify that the authenticated user is allowed to access the specified patient.

---

# 47. Versioning

The API uses:

```text
/api/v1/
```

Future breaking changes may use:

```text
/api/v2/
```

Existing clients should not unexpectedly break because of a minor backend change.

---

# 48. API Ownership

| API Area       | Primary Owner     | Consumers           |
| -------------- | ----------------- | ------------------- |
| Authentication | Backend           | Patient + Caregiver |
| Patient        | Backend           | Patient + Caregiver |
| Games          | Backend           | Patient             |
| Game Sessions  | Backend           | Patient + AI        |
| Performance    | Backend           | AI + Caregiver      |
| AI             | AI + Backend      | Patient + Caregiver |
| Reminders      | Backend           | Patient + Caregiver |
| Consent        | Backend           | Patient + Caregiver |
| Sync           | Backend + Offline | Patient             |
| Dashboard data | Backend           | Caregiver           |

---

# 49. Development Contract

Before changing an API:

1. Update this document.
2. Inform affected team members.
3. Update backend implementation.
4. Update consuming clients.
5. Add/update tests.
6. Verify integration.

No team should silently change a shared API contract.

---

# 50. MVP API Priority

The following endpoints have the highest priority:

### Priority 1

```text
POST /auth/login
GET /auth/me
POST /game-sessions
GET /patients/{id}/game-sessions
POST /sync
GET /sync
```

### Priority 2

```text
GET /patients/{id}/performance
GET /patients/{id}/trends
GET /patients/{id}/difficulty/{game}
POST /ai/analyze
```

### Priority 3

```text
POST /reminders
GET /patients/{id}/reminders
POST /reminders/{id}/completions
POST /consents
PUT /consents/{id}
GET /caregiver/patients
```

---

# 51. MVP Simplification

The API is intentionally designed for the hackathon MVP.

The team does not need to implement:

* Microservices
* GraphQL
* Complex event streaming
* Real-time WebSockets
* Advanced distributed synchronization
* Complex healthcare integrations

unless required later.

A simple, reliable REST API is preferred.

---

# 52. Definition of Done

The API contract is considered ready for implementation when:

```text
Patient App
     │
     ├── Authentication
     ├── Games
     ├── Sessions
     ├── Reminders
     └── Sync
             │
             ▼
          Backend
             │
        ┌────┼─────┐
        ▼    ▼     ▼
       DB    AI   Auth
        │
        ▼
 Caregiver Dashboard
```

can be implemented without teams having to guess:

* Endpoint names
* HTTP methods
* Required fields
* Response fields
* Authentication requirements
* Authorization rules
* Sync behavior
* Error behavior

---

# 53. Related Documents

System architecture:

```text
/docs/architecture/system-architecture.md
```

Data flow:

```text
/docs/architecture/data-flow.md
```

Synchronization:

```text
/docs/architecture/sync-flow.md
```

Requirements:

```text
/docs/product/requirements.md
```

User flows:

```text
/docs/product/user-flows.md
```

---

**Document Owner:** Product + System Architect
**Project:** CogniCare NER
**API Version:** v1
**Status:** Architecture Baseline — SIH 2026 MVP
