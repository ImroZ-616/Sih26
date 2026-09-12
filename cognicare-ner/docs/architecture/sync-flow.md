# 🧠 CogniCare NER — Offline & Synchronization Flow

**Project:** CogniCare NER
**Hackathon:** Smart India Hackathon 2026
**Document Owner:** Product + System Architect
**Status:** Architecture Baseline — SIH 2026 MVP

---

# 1. Purpose

This document defines how CogniCare NER handles data when the patient device is:

* Offline
* Online
* Intermittently connected
* Reconnected after an offline period
* Experiencing synchronization failures

The synchronization architecture must ensure:

* No loss of completed patient activity
* Reliable retry behavior
* Duplicate prevention
* Eventually consistent server data
* Safe synchronization of offline-created records
* Clear ownership between mobile app and backend

---

# 2. Core Principle

> **The patient application must never depend on continuous internet connectivity for core patient functionality.**

The patient should be able to:

```text
Open App
   ↓
Play Game
   ↓
Complete Game
   ↓
Save Result
```

even when:

```text
Internet = OFF
```

Synchronization happens later.

---

# 3. Offline-First Architecture

```text
                  PATIENT APP
                       │
                       ▼
                ┌─────────────┐
                │  LOCAL DB   │
                └──────┬──────┘
                       │
                       ▼
                ┌─────────────┐
                │ SYNC QUEUE  │
                └──────┬──────┘
                       │
                 Connectivity
                       │
             ┌─────────┴─────────┐
             │                   │
          OFFLINE              ONLINE
             │                   │
             ▼                   ▼
       Stay in Queue         Sync Engine
                                 │
                                 ▼
                             BACKEND
                                 │
                                 ▼
                              SERVER DB
```

---

# 4. Data States

A locally generated record can move through the following states:

```text
LOCAL
  ↓
QUEUED
  ↓
SYNCING
  ↓
SYNCED
```

If synchronization fails:

```text
SYNCING
   ↓
FAILED
   ↓
QUEUED
   ↓
RETRY
```

The local record should remain available until successful synchronization.

---

# 5. Sync Queue

The patient application maintains a local synchronization queue.

Each queued operation represents an action that must eventually be reflected on the backend.

Conceptual structure:

```json
{
  "sync_id": "sync-abc123",
  "entity_type": "game_session",
  "entity_id": "session-123",
  "operation": "CREATE",
  "payload": {},
  "created_at": "2026-09-12T10:30:00Z",
  "attempt_count": 0,
  "status": "QUEUED",
  "last_attempt_at": null,
  "last_error": null
}
```

The exact implementation depends on the selected mobile technology and local database.

---

# 6. Unique Client IDs

Every offline-created entity should receive a client-generated unique identifier.

Example:

```text
session-8f3a91...
```

This identifier allows the backend to distinguish:

```text
New record
```

from:

```text
Same record being synchronized again
```

This is essential for duplicate prevention.

---

# 7. Why Client IDs Are Necessary

Consider:

```text
Patient completes game
        ↓
Local session created
        ↓
Sync request sent
        ↓
Server saves session
        ↓
Network fails BEFORE response reaches phone
```

The phone does not know whether the server saved the record.

It retries.

Without idempotency:

```text
Server:
Session A
Session A
```

With client-generated IDs:

```text
Server:
Session A

Retry:
"session_id already exists"

→ Return success / already processed
```

Therefore, synchronization must be **idempotent**.

---

# 8. Idempotency Principle

Repeated synchronization of the same logical operation must not create duplicate data.

Conceptually:

```text
Same client ID
      ↓
Backend checks existing record
      │
      ├── Exists → Return existing result
      │
      └── Does not exist → Create record
```

The backend should treat repeated delivery of the same operation safely.

---

# 9. Sync Trigger Conditions

Synchronization may be triggered when:

### Application starts

```text
App Launch
   ↓
Check Connectivity
   ↓
If Online → Attempt Sync
```

### Connectivity returns

```text
Offline
  ↓
Network Restored
  ↓
Trigger Sync
```

### Manual refresh

```text
User / App requests sync
        ↓
Sync Queue processed
```

### Background synchronization

If supported by the selected mobile framework/platform:

```text
Background Task
      ↓
Check Connectivity
      ↓
Sync Pending Data
```

Background behavior must respect operating-system restrictions.

---

# 10. Connectivity Detection

The mobile application should determine whether a network connection is available before attempting synchronization.

Conceptually:

```text
Connectivity Status
        │
        ├── OFFLINE
        │      ↓
        │   Do Nothing
        │
        └── ONLINE
               ↓
           Start Sync
```

Connectivity detection should not be treated as proof that the backend is reachable.

A request can still fail after the device reports connectivity.

---

# 11. Synchronization Algorithm

Conceptual process:

```text
START SYNC
    │
    ▼
Check Network
    │
    ├── Offline → STOP
    │
    ▼
Load Pending Queue
    │
    ▼
Select Next Item
    │
    ▼
Send to Backend
    │
    ├── Success
    │      ↓
    │   Mark Synced
    │
    ├── Duplicate
    │      ↓
    │   Mark Synced
    │
    ├── Temporary Error
    │      ↓
    │   Retry Later
    │
    └── Permanent Error
           ↓
       Record Error
           ↓
       Require Handling
```

---

# 12. Queue Processing Order

For the MVP, queued operations should normally be processed in creation order.

Example:

```text
Queue

1. Game Session A
2. Game Session B
3. Reminder Completion A
4. Game Session C
```

Process:

```text
A → B → Reminder → C
```

This makes behavior predictable.

However, independent operations may later be processed in parallel if required.

---

# 13. Retry Strategy

Temporary failures should be retried.

Examples of temporary failures:

* Network unavailable
* Request timeout
* Server temporarily unavailable
* Connection reset

Conceptual retry schedule:

```text
Attempt 1
   ↓
Failure
   ↓
Wait
   ↓
Attempt 2
   ↓
Failure
   ↓
Wait longer
   ↓
Attempt 3
```

An exponential backoff strategy is preferred.

Example:

```text
1st retry → short delay
2nd retry → longer delay
3rd retry → longer delay
```

The exact delay values can be selected during implementation.

---

# 14. Maximum Retry Attempts

The system should avoid continuously retrying a permanently invalid request.

Conceptually:

```text
Attempt Count
     │
     ├── Within limit → Retry
     │
     └── Exceeds limit
              ↓
          Mark Failed
              ↓
       Keep diagnostic info
```

The underlying patient data should not be deleted merely because synchronization failed.

A failed operation may require a future retry or manual recovery.

---

# 15. Temporary vs Permanent Errors

### Temporary

Examples:

```text
Network timeout
HTTP 408
HTTP 429
HTTP 500
HTTP 502
HTTP 503
```

Action:

```text
Retry later
```

### Permanent

Examples:

```text
Invalid data
Unauthorized request
Malformed request
Unsupported operation
```

Action:

```text
Do not blindly retry forever.
Record error.
Handle according to application logic.
```

The exact HTTP status handling will be finalized with the backend implementation.

---

# 16. Server Acknowledgement

A local record should only be considered synchronized after the backend acknowledges successful processing.

Example:

```text
Local DB
   │
   ▼
Sync Request
   │
   ▼
Backend
   │
   ▼
Validate
   │
   ▼
Save
   │
   ▼
ACK
   │
   ▼
Local DB
   │
   ▼
Mark SYNCED
```

If the acknowledgement is not received:

```text
Do NOT assume failure.
```

The request may have reached the server.

The next synchronization attempt must therefore be idempotent.

---

# 17. Example: Network Failure After Server Save

This is a critical case.

```text
Phone
  │
  │ POST session
  ▼
Server
  │
  │ Saves session
  ▼
Database
  │
  X
Network failure
  │
  X
ACK never reaches phone
```

The phone still considers the item pending.

Later:

```text
Phone
  │
  │ POST same session
  ▼
Server
  │
  ▼
Check client/session ID
  │
  ▼
Already exists
  │
  ▼
Return success / already processed
  │
  ▼
Phone marks synced
```

This prevents duplicate sessions.

---

# 18. Conflict Handling

Most MVP data generated by the patient is append-only.

Examples:

```text
Game Session
Game Result
Reminder Completion
Activity Event
```

These generally do not require complex merge logic.

The architecture should prefer:

```text
Append-only event creation
```

where possible.

---

# 19. Reminder Conflict

Reminders may require special handling because they can be modified from multiple locations.

Example:

```text
Caregiver changes reminder
        ↓
Server has new reminder
        ↓
Patient device has old cached reminder
```

For MVP, the backend should be treated as the authoritative source for synchronized reminder configuration.

The patient app should refresh reminder configuration when connectivity is available.

---

# 20. Last-Known-Server-State

The patient application may cache the latest server state.

Example:

```text
Server
   ↓
Reminder configuration
   ↓
Patient Local DB
```

When offline:

```text
Use latest known configuration
```

When online:

```text
Fetch updated configuration
```

---

# 21. Server as Source of Truth

The system follows:

```text
Offline:
Local DB = current local source of truth

Online / synchronized:
Server DB = shared source of truth
```

The local database remains necessary for offline functionality.

---

# 22. Sync Direction

Synchronization is not always one-way.

### Patient → Server

Examples:

```text
Game sessions
Game results
Reminder completions
Patient activity
```

### Server → Patient

Examples:

```text
Reminder configuration
Difficulty recommendation
Updated patient configuration
Caregiver-approved settings
Language/content configuration
```

Therefore:

```text
             ┌───────────────┐
             │   PATIENT APP │
             └───────┬───────┘
                     │
              BIDIRECTIONAL
                     │
                     ▼
             ┌───────────────┐
             │    BACKEND    │
             └───────────────┘
```

---

# 23. Bidirectional Synchronization

Example:

```text
Patient App
     │
     │ Upload local game sessions
     ▼
Backend
     │
     │ Return recommendations
     ▼
Patient App
```

The application should process both:

```text
UPLOAD
```

and:

```text
DOWNLOAD
```

operations.

---

# 24. Game Difficulty Synchronization

The AI may recommend a new difficulty.

Flow:

```text
Game Session
     ↓
Backend
     ↓
AI Engine
     ↓
Difficulty Recommendation
     ↓
Backend
     ↓
Patient App
     ↓
Local Difficulty State
```

If the patient is offline, the app may use the last known difficulty recommendation.

After reconnecting, it can obtain the latest recommendation.

---

# 25. Sync Metadata

The local system should maintain enough metadata to understand synchronization state.

Possible fields:

```text
sync_id
entity_id
entity_type
operation
status
attempt_count
created_at
updated_at
last_attempt_at
last_error
```

This is implementation guidance, not a strict database schema.

---

# 26. Sync Queue Example

Example local queue:

```text
┌────────┬─────────────┬──────────┬──────────┐
│ ID     │ Entity      │ Operation│ Status   │
├────────┼─────────────┼──────────┼──────────┤
│ 001    │ Session A   │ CREATE   │ QUEUED   │
│ 002    │ Session B   │ CREATE   │ QUEUED   │
│ 003    │ Reminder A  │ UPDATE   │ QUEUED   │
└────────┴─────────────┴──────────┴──────────┘
```

After successful synchronization:

```text
┌────────┬─────────────┬──────────┬──────────┐
│ ID     │ Entity      │ Operation│ Status   │
├────────┼─────────────┼──────────┼──────────┤
│ 001    │ Session A   │ CREATE   │ SYNCED   │
│ 002    │ Session B   │ CREATE   │ SYNCED   │
│ 003    │ Reminder A  │ UPDATE   │ SYNCED   │
└────────┴─────────────┴──────────┴──────────┘
```

---

# 27. Complete Offline-to-Online Example

Consider a patient playing three games while offline.

```text
             INTERNET OFF
                  │
                  ▼
        ┌──────────────────┐
        │ Patient App      │
        └────────┬─────────┘
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
   Game A     Game B     Game C
       │         │         │
       ▼         ▼         ▼
    Local DB  Local DB  Local DB
       │         │         │
       └─────────┼─────────┘
                 ▼
            Sync Queue
```

When internet returns:

```text
             INTERNET ON
                  │
                  ▼
             Sync Engine
                  │
          ┌───────┼───────┐
          ▼       ▼       ▼
       Session A B       C
          │       │       │
          └───────┼───────┘
                  ▼
               Backend
                  │
                  ▼
              Database
```

After acknowledgement:

```text
Local Queue
    ↓
A = SYNCED
B = SYNCED
C = SYNCED
```

---

# 28. Synchronization and AI

AI processing should normally occur after the relevant performance data reaches the backend.

```text
Offline Game
     ↓
Local DB
     ↓
Sync
     ↓
Backend
     ↓
Database
     ↓
AI Engine
     ↓
Recommendation
```

However, simple local difficulty adaptation may be performed offline if the mobile implementation requires immediate adaptation.

If local AI is used, its behavior must remain consistent with the defined cognitive-engine rules.

---

# 29. Synchronization and Caregiver Dashboard

The caregiver dashboard reads synchronized server data.

```text
Patient
   ↓
Offline Activity
   ↓
Local DB
   ↓
Sync
   ↓
Backend
   ↓
Server DB
   ↓
Caregiver Dashboard
```

Therefore:

> A caregiver should not be expected to see activity that has not yet synchronized from the patient's device.

The dashboard may display the last synchronization timestamp.

---

# 30. Last Synchronization Timestamp

The caregiver dashboard should ideally show:

```text
Last synchronized:
12 September 2026, 10:42 AM
```

This helps the caregiver understand whether the displayed data is current.

Example:

```text
Last Sync: 2 hours ago
```

This is particularly important for an offline-first application.

---

# 31. Sync Status for Patient

The patient application may display a simple status:

```text
✓ Synced
```

or:

```text
⟳ Syncing...
```

or:

```text
Offline — saved locally
```

Avoid technical error messages that may confuse elderly users.

---

# 32. Sync Status for Developers

Developer/debug mode may expose:

```text
Pending: 3
Syncing: 1
Failed: 0
Last Sync: 10:42
```

This should not necessarily be visible to normal patients.

---

# 33. Data Integrity Rules

The synchronization system must ensure:

### Rule 1

Never delete unsynchronized patient activity automatically.

### Rule 2

Never mark a record synchronized without server acknowledgement or a confirmed idempotent server response.

### Rule 3

Repeated synchronization must not create duplicates.

### Rule 4

Temporary network failures must be recoverable.

### Rule 5

Permanent validation failures must be recorded and handled explicitly.

### Rule 6

The patient must remain able to use core functionality while synchronization is unavailable.

---

# 34. Security During Synchronization

Synchronization must use secure network communication.

```text
Patient App
     │
     │ HTTPS
     ▼
Backend API
```

Authentication must be applied to protected endpoints.

The sync queue must not expose sensitive information unnecessarily.

API keys and authentication secrets must never be stored in source control.

---

# 35. Sync Failure Scenarios

The implementation should consider:

### Scenario A — No internet

```text
Result → Local DB → Queue
```

No error should be shown to the patient.

### Scenario B — Internet returns

```text
Queue → Backend → Success
```

Record becomes synced.

### Scenario C — Server unavailable

```text
Queue → Backend → Error
              ↓
          Retry Later
```

### Scenario D — Request timeout after server processing

```text
Server saves
     ↓
ACK lost
     ↓
Client retries
     ↓
Server detects duplicate
     ↓
Client marks synced
```

### Scenario E — Invalid request

```text
Backend rejects
     ↓
Do not retry forever
     ↓
Record error
```

---

# 36. MVP Simplification

For the SIH MVP, synchronization should focus on:

```text
1. Game Sessions
2. Game Results
3. Reminder Completions
4. Reminder Configuration
5. Difficulty Recommendation
```

Complex multi-device conflict resolution is not required unless introduced by the team's implementation.

---

# 37. Recommended MVP Sync Architecture

```text
             ┌──────────────────┐
             │   PATIENT APP    │
             └────────┬─────────┘
                      │
                      ▼
              ┌───────────────┐
              │    LOCAL DB   │
              └───────┬───────┘
                      │
                      ▼
              ┌───────────────┐
              │  SYNC QUEUE   │
              └───────┬───────┘
                      │
              Internet Available
                      │
                      ▼
              ┌───────────────┐
              │  SYNC SERVICE │
              └───────┬───────┘
                      │ HTTPS
                      ▼
              ┌───────────────┐
              │   BACKEND API │
              └───────┬───────┘
                      │
                      ▼
              ┌───────────────┐
              │ SERVER DATABASE│
              └───────────────┘
```

---

# 38. Team Responsibilities

| Component                    | Owner                 |
| ---------------------------- | --------------------- |
| Local DB integration         | Patient App + Offline |
| Sync Queue                   | Offline/Sync          |
| Connectivity detection       | Patient App + Offline |
| Sync API                     | Backend               |
| Idempotency                  | Backend               |
| Server persistence           | Backend               |
| AI synchronization           | AI + Backend          |
| Dashboard refresh            | Caregiver             |
| Voice/content offline assets | Voice                 |
| Overall contract             | System Architect      |

---

# 39. Integration Contract

The following must be agreed between teams before implementation:

```text
1. Local game-session schema
2. Client-generated ID format
3. Sync queue structure
4. Sync API endpoint
5. Request/response format
6. Authentication method
7. Idempotency behavior
8. Retry behavior
9. Error format
10. Server acknowledgement format
```

These details belong in:

```text
/docs/api/api-contract.md
```

---

# 40. Definition of Done

The synchronization system is considered complete when the team can demonstrate:

```text
1. Turn internet OFF.

2. Patient plays a game.

3. Game result is saved locally.

4. Sync queue contains the result.

5. Close/reopen application.

6. Result remains available.

7. Turn internet ON.

8. Synchronization begins.

9. Backend receives result.

10. Backend stores result.

11. Client receives acknowledgement.

12. Queue marks result as synced.

13. Repeating synchronization does not create duplicates.

14. Caregiver dashboard eventually displays the result.
```

---

# 41. Final Principle

> **Offline is a normal operating state, not an error state.**

The patient application must continue functioning normally when the network is unavailable.

Synchronization should happen transparently whenever connectivity returns.

---

# 42. Related Documents

System architecture:

```text
/docs/architecture/system-architecture.md
```

Data flow:

```text
/docs/architecture/data-flow.md
```

API contracts:

```text
/docs/api/api-contract.md
```

Product requirements:

```text
/docs/product/requirements.md
```

---

**Document Owner:** Product + System Architect
**Project:** CogniCare NER
**Status:** Architecture Baseline — SIH 2026 MVP
