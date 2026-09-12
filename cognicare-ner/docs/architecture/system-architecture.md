# 🧠 CogniCare NER — System Architecture

**Project:** CogniCare NER
**Hackathon:** Smart India Hackathon 2026
**Problem Domain:** AI Cognitive Gaming & Memory Assistance for Elderly People
**Organization:** Ministry of Development of North Eastern Region (MDoNER)

---

## 1. Purpose

This document defines the high-level technical architecture of CogniCare NER.

It establishes:

* Major system components
* Responsibilities of each component
* Communication between components
* Data ownership
* Offline-first behavior
* AI integration
* Voice/language integration
* Caregiver monitoring
* Security boundaries
* Module boundaries for parallel development

This document is the primary architectural reference for the development team.

Any major architectural change should be discussed with the System Architect before implementation.

---

# 2. Product Overview

CogniCare NER is an **offline-first, multilingual cognitive stimulation and memory-assistance platform** designed for elderly users, particularly in remote regions of India's North Eastern Region.

The system consists of two primary user experiences:

### Patient

The patient uses the mobile application to:

* Play cognitive games
* Receive adaptive difficulty
* Complete memory-oriented activities
* Receive medicine/hydration/appointment reminders
* Interact through voice
* Use the application without continuous internet connectivity

### Caregiver

The caregiver uses a web dashboard to:

* View linked patients
* Review activity
* View cognitive-performance trends
* Monitor reminders
* Receive plain-language performance alerts

---

# 3. Important Medical Boundary

CogniCare NER is a **non-diagnostic system**.

The system must not claim to:

* Diagnose Alzheimer's disease
* Diagnose dementia
* Predict dementia
* Replace a doctor
* Replace a neurologist
* Replace a cognitive therapist
* Provide a clinical cognitive assessment

The AI engine evaluates **interaction and game-performance trends only**.

Example:

```text
Allowed:

"Performance has decreased over the last 5 sessions."

Not allowed:

"Patient is showing signs of Alzheimer's."
```

The caregiver dashboard must communicate observations rather than medical diagnoses.

---

# 4. High-Level Architecture

```text
                         ┌─────────────────────────┐
                         │       PATIENT            │
                         │                          │
                         │  Elderly User            │
                         └────────────┬────────────┘
                                      │
                                      ▼
                    ┌────────────────────────────────┐
                    │        PATIENT MOBILE APP      │
                    │                                │
                    │ Games                          │
                    │ Reminders                      │
                    │ Voice Interaction              │
                    │ Accessibility / UX             │
                    └───────────────┬────────────────┘
                                    │
                                    ▼
                    ┌────────────────────────────────┐
                    │         LOCAL DATA LAYER        │
                    │                                │
                    │ SQLite / Local DB              │
                    │ Session Data                   │
                    │ Reminder Data                  │
                    │ Sync Queue                     │
                    └───────────────┬────────────────┘
                                    │
                         Internet Available?
                              /           \
                            NO             YES
                            │               │
                            │               ▼
                            │       ┌───────────────┐
                            │       │ SYNC ENGINE   │
                            │       └───────┬───────┘
                            │               │
                            └───────────────┤
                                            ▼
                               ┌─────────────────────┐
                               │      BACKEND        │
                               │                     │
                               │ Authentication      │
                               │ Patient Data        │
                               │ Game Sessions       │
                               │ Reminders           │
                               │ Consent             │
                               │ Synchronization     │
                               └──────────┬──────────┘
                                          │
                         ┌────────────────┼────────────────┐
                         │                │                │
                         ▼                ▼                ▼
                 ┌──────────────┐ ┌──────────────┐ ┌─────────────────┐
                 │ AI /         │ │ DATABASE     │ │ CAREGIVER       │
                 │ COGNITIVE    │ │              │ │ DASHBOARD       │
                 │ ENGINE       │ │ PostgreSQL   │ │                 │
                 │              │ │ / chosen DB  │ │ Trends          │
                 │ Adaptation   │ │              │ │ Alerts          │
                 │ Trends       │ │              │ │ Reminders       │
                 └──────────────┘ └──────────────┘ └─────────────────┘
                                          │
                                          │
                               ┌──────────▼──────────┐
                               │ VOICE / LANGUAGE    │
                               │                     │
                               │ Bhashini            │
                               │ ASR / TTS /         │
                               │ Translation         │
                               └─────────────────────┘
```

---

# 5. Architectural Principles

## 5.1 Offline First

The patient application must remain usable without an internet connection.

Core patient functionality must not depend on a live server.

Offline functionality includes:

* Starting games
* Completing games
* Storing game results
* Viewing local progress
* Viewing scheduled reminders
* Completing reminders
* Basic voice/content functionality where locally available

When connectivity returns, locally stored data is synchronized with the backend.

---

## 5.2 Modular Architecture

Each major subsystem has a clear responsibility.

```text
Patient App
Backend
AI Engine
Caregiver Dashboard
Offline/Sync
Voice/Language
```

Modules should communicate through clearly defined interfaces.

Avoid direct coupling between unrelated modules.

---

## 5.3 API-First Integration

The backend API contract is the communication boundary between:

```text
Patient App ↔ Backend
Caregiver Dashboard ↔ Backend
AI Engine ↔ Backend
```

API definitions are maintained in:

```text
/docs/api/api-contract.md
```

Changes to API contracts should be coordinated before implementation.

---

## 5.4 Explainable AI

The MVP AI engine should primarily use transparent rules.

Example:

```text
High accuracy + low response time
        ↓
Increase difficulty

Low accuracy + many mistakes
        ↓
Decrease difficulty

Stable performance
        ↓
Maintain difficulty
```

The system should be able to explain why a difficulty recommendation was produced.

---

## 5.5 Privacy by Design

Patient data must be treated as sensitive.

The system should implement:

* Authentication
* Role-based authorization
* Consent-based caregiver linking
* Minimal data collection
* Secure communication
* No secrets in source control
* Appropriate local storage protection

---

# 6. Major Components

## 6.1 Patient Mobile App

Location:

```text
/patient-app/
```

### Responsibilities

The patient app is responsible for:

* Patient UI
* Cognitive games
* Game interaction
* Reminder interface
* Voice interaction
* Local storage
* Offline operation
* Sync queue interaction

### It should NOT own

* Server database
* Caregiver authorization
* Global patient analytics
* Medical diagnosis
* Backend authentication logic

---

# 7. Cognitive Game Layer

The MVP contains three games.

## 7.1 Memory Match

The patient matches related cards/items.

Example:

```text
Apple ↔ Apple
Book  ↔ Book
Cup   ↔ Cup
```

Potential metrics:

* Accuracy
* Attempts
* Mistakes
* Completion time
* Difficulty level

---

## 7.2 Daily Routine Recall

The patient recalls or arranges common daily activities.

Example:

```text
Wake up
   ↓
Brush teeth
   ↓
Breakfast
   ↓
Medicine
   ↓
Lunch
```

Potential metrics:

* Correct sequence
* Number of attempts
* Completion time
* Hints used

---

## 7.3 Pattern Recognition

The patient identifies the next item in a visual or numerical pattern.

Example:

```text
○ → △ → ○ → △ → ?
```

Expected:

```text
○
```

Potential metrics:

* Accuracy
* Response time
* Attempts
* Difficulty

---

# 8. Game Result Model

Every completed game should generate a standardized result.

Conceptually:

```json
{
  "session_id": "session_123",
  "patient_id": "patient_001",
  "game_id": "memory_match",
  "difficulty": 2,
  "score": 80,
  "accuracy": 0.8,
  "mistakes": 2,
  "completion_time_seconds": 42,
  "timestamp": "2026-09-12T10:30:00Z"
}
```

The exact API format is defined in:

```text
/docs/api/api-contract.md
```

The model may evolve during implementation.

---

# 9. Local Data Layer

The patient application uses a local database.

Candidate technologies:

* SQLite
* Realm
* WatermelonDB

The final choice should be made based on the selected mobile framework.

The local database stores information required for offline operation.

Possible local entities:

```text
PatientProfile
GameSession
GameResult
Reminder
SyncQueue
DifficultyState
VoiceContent
```

---

# 10. Sync Engine

The sync engine is responsible for transferring locally created data to the backend when connectivity becomes available.

Basic flow:

```text
Game completed
      ↓
Save to Local DB
      ↓
Create Sync Queue Entry
      ↓
Internet unavailable?
      │
      ├── YES → Keep queued
      │
      └── NO
            ↓
       Send to Backend
            ↓
       Server acknowledges
            ↓
       Mark as synchronized
```

The synchronization system should support retries.

---

# 11. Duplicate Prevention

Network failures can cause the same request to be sent multiple times.

Therefore, important offline-created records should have a unique identifier such as:

```text
client_generated_id
```

Example:

```text
game-session-8f29...
```

The backend should recognize duplicate synchronization attempts and avoid creating duplicate records.

---

# 12. Backend

Location:

```text
/backend/
```

The backend provides centralized services for the platform.

### Responsibilities

* Authentication
* Authorization
* Patient management
* Caregiver management
* Consent
* Game sessions
* Performance data
* Reminders
* Synchronization
* AI data exchange
* Dashboard data

---

# 13. Database

The server-side database stores synchronized application data.

The architecture should support entities such as:

```text
User
Patient
Caregiver
Consent
Game
GameSession
PerformanceMetric
DifficultyState
Reminder
SyncEvent
```

Relationships must enforce appropriate authorization boundaries.

A caregiver should only access patients who have been explicitly linked/approved.

---

# 14. AI / Cognitive Engine

Location:

```text
/ai-engine/
```

The cognitive engine analyzes game-performance information.

### Inputs

Potential inputs:

```text
Game ID
Difficulty
Score
Accuracy
Mistakes
Completion Time
Previous Sessions
```

### Outputs

Potential outputs:

```text
Recommended Difficulty
Performance Trend
Confidence / Reason
Caregiver-Friendly Message
```

Example:

```text
Input:

accuracy = 0.91
mistakes = 1
difficulty = 2

Output:

recommended_difficulty = 3

reason =
"Performance was consistently strong at the current level."
```

---

# 15. Adaptive Difficulty

The MVP uses rule-based adaptation.

Example conceptual algorithm:

```text
IF performance is consistently high
    increase difficulty

ELSE IF performance is consistently low
    decrease difficulty

ELSE
    maintain difficulty
```

Difficulty should change gradually rather than jumping multiple levels after one session.

The AI engine must consider multiple sessions where appropriate.

---

# 16. Trend Detection

The system may analyze recent sessions to identify:

* Improving performance
* Stable performance
* Declining performance
* Irregular performance
* Reduced participation

Example:

```text
Session 1 → 82%
Session 2 → 79%
Session 3 → 76%
Session 4 → 71%
Session 5 → 68%

Trend:
Declining performance
```

This is an observation, not a diagnosis.

---

# 17. Caregiver Dashboard

Location:

```text
/caregiver-dashboard/
```

The caregiver dashboard provides a summarized view of linked patients.

### Responsibilities

* Authentication
* Patient selection
* Activity overview
* Performance trends
* Reminder status
* Alerts
* Basic historical information

### Example dashboard

```text
Patient: Patient 001

Recent Activity
────────────────────────
Memory Match       ✓
Pattern Recognition ✓
Routine Recall     ✓

Performance
────────────────────────
Accuracy            ↓
Completion Time     ↑
Game Frequency      →

Awareness
────────────────────────
"Performance has decreased
over recent sessions."
```

---

# 18. Consent-Based Caregiver Linking

Caregiver access must not be automatic.

Conceptual flow:

```text
Patient
   ↓
Approve caregiver
   ↓
Consent Record
   ↓
Backend verifies relationship
   ↓
Caregiver can access permitted data
```

A caregiver must not be able to access arbitrary patient records.

---

# 19. Reminder System

The reminder system supports:

* Medicine
* Hydration
* Appointments

A reminder contains conceptually:

```text
Reminder
├── ID
├── Patient
├── Type
├── Title
├── Scheduled Time
├── Repeat Rule
├── Completion Status
└── Voice/Text Content
```

Reminders should work locally even when the device is offline.

---

# 20. Voice and Language Layer

The voice/language layer provides:

```text
Speech → ASR
Text → TTS
Language A → Translation → Language B
```

The architecture allows integration with Bhashini.

The patient application should not tightly couple its UI logic to a specific external voice provider.

A provider abstraction should be preferred.

Conceptually:

```text
Patient App
     ↓
Voice Service Interface
     ↓
┌─────────────────┐
│ Bhashini        │
│ Local Audio     │
│ Future Provider │
└─────────────────┘
```

---

# 21. Language Content Packs

Language-specific content is stored separately.

Example:

```text
/voice-content-packs/

├── en/
├── as/
├── bo/
└── mni/
```

Content may include:

* Game instructions
* Reminder messages
* Navigation prompts
* Feedback messages
* Accessibility text
* Audio references

The MVP does not require complete coverage of all Indian languages.

---

# 22. Data Flow

The standard game flow is:

```text
Patient
   ↓
Starts Game
   ↓
Game Engine
   ↓
Game Completed
   ↓
Generate Result
   ↓
Local Database
   ↓
Sync Queue
   ↓
Backend
   ↓
Database
   ↓
AI Engine
   ↓
Difficulty Recommendation
   ↓
Patient App
```

The detailed data flow is documented in:

```text
/docs/architecture/data-flow.md
```

---

# 23. Offline Flow

```text
                    GAME START
                        │
                        ▼
                  Local Game Data
                        │
                        ▼
                   Play Game
                        │
                        ▼
                  Game Result
                        │
                        ▼
                    Local DB
                        │
                        ▼
                   Sync Queue
                        │
                ┌───────┴───────┐
                │               │
             Offline          Online
                │               │
                ▼               ▼
          Wait in Queue      Send API
                                │
                                ▼
                           Server ACK
                                │
                                ▼
                       Mark Synchronized
```

---

# 24. Online/Offline Responsibility

| Function                         |    Offline | Online |
| -------------------------------- | ---------: | -----: |
| Play games                       |          ✅ |      ✅ |
| Save game result                 |          ✅ |      ✅ |
| View local progress              |          ✅ |      ✅ |
| View reminders                   |          ✅ |      ✅ |
| Complete reminders               |          ✅ |      ✅ |
| Voice using cached/local content |          ✅ |      ✅ |
| Synchronize data                 |          ❌ |      ✅ |
| Caregiver dashboard              | Limited/No |      ✅ |
| Server analytics                 |          ❌ |      ✅ |

---

# 25. Security Architecture

Security is divided across layers.

```text
Mobile
  ↓
Secure Authentication
  ↓
Encrypted Transport
  ↓
Backend Authorization
  ↓
Consent Verification
  ↓
Database
```

### Requirements

* HTTPS for network communication
* Secure authentication
* Role-based authorization
* Consent verification
* Input validation
* Secure token handling
* No API keys in Git
* No unnecessary personal data
* Appropriate local storage protection

---

# 26. Environment Configuration

Secrets must never be committed.

Use:

```text
.env
```

locally.

Document required variables using:

```text
.env.example
```

Examples:

```text
DATABASE_URL=
API_BASE_URL=
BHASHINI_API_KEY=
AUTH_SECRET=
```

Values must remain empty or placeholder values in `.env.example`.

---

# 27. Module Ownership

| Module                         | Owner  |
| ------------------------------ | ------ |
| Patient App                    | Role 2 |
| AI Engine                      | Role 3 |
| Backend + Database             | Role 4 |
| Offline + Voice + Security     | Role 5 |
| Caregiver Dashboard + Research | Role 6 |
| Overall Architecture           | Role 1 |

---

# 28. Repository Boundaries

Each team member should primarily work within their assigned directory.

```text
Role 2 → /patient-app/
Role 3 → /ai-engine/
Role 4 → /backend/
Role 5 → /voice-content-packs/
Role 6 → /caregiver-dashboard/
Role 1 → /docs/ + architecture
```

Cross-module changes require communication with the affected owner.

---

# 29. Integration Strategy

Development follows:

```text
feature branch
      ↓
Pull Request
      ↓
dev
      ↓
Integration Testing
      ↓
Demo Testing
      ↓
main
```

`main` must remain demo-ready.

---

# 30. MVP Integration Scenario

The complete MVP should demonstrate the following scenario:

```text
1. Patient opens application.

2. Patient starts Memory Match.

3. Patient completes the game.

4. Application calculates performance.

5. Result is saved locally.

6. If offline:
       result remains in sync queue.

7. Internet becomes available.

8. Result synchronizes with backend.

9. Backend stores the session.

10. AI engine evaluates recent performance.

11. AI recommends next difficulty.

12. Patient receives the new difficulty.

13. Caregiver opens dashboard.

14. Caregiver sees recent activity.

15. Caregiver sees performance trend.

16. If a meaningful trend exists,
    caregiver receives a plain-language awareness alert.
```

This end-to-end flow is the primary integration target for the hackathon.

---

# 31. Architecture Decision Principles

When choosing between technical approaches, prioritize:

```text
Reliability
    ↓
Offline capability
    ↓
Simple integration
    ↓
Explainability
    ↓
Security
    ↓
Maintainability
    ↓
Scalability
```

For the hackathon, a simple working solution is preferred over an unnecessarily complex production architecture.

---

# 32. Out of Scope

The MVP does not include:

* Clinical diagnosis
* Medical prediction
* Clinical validation
* Full 22-language implementation
* Production-grade hospital integration
* Advanced medical AI
* Automated medical decisions
* Emergency medical response
* Replacement of healthcare professionals

---

# 33. Future Extensions

Possible future features include:

* Additional NER languages
* More cognitive games
* Personalized activity recommendations
* Advanced longitudinal analytics
* Healthcare-provider integration
* More sophisticated ML models
* Wearable/device integration
* Telehealth integration
* Larger-scale deployment

These are not required for the SIH MVP.

---

# 34. Definition of Done

The architecture is considered successfully implemented when:

```text
Patient App
      │
      ▼
Cognitive Games
      │
      ▼
Offline Storage
      │
      ▼
Synchronization
      │
      ▼
Backend
      │
      ├──────────────┐
      ▼              ▼
 AI Engine     Caregiver Dashboard
      │              │
      ▼              ▼
Adaptive       Performance
Difficulty       Trends
```

works as one integrated demonstration.

The system must remain usable by the patient during temporary loss of connectivity.

---

# 35. Architecture Change Rule

Any change affecting the following must be discussed with the System Architect:

* API contracts
* Database schema
* Authentication
* Consent model
* Sync protocol
* Module boundaries
* Data models shared between modules
* AI input/output contract
* Major technology choices

Small implementation details inside an owned module do not require architectural approval.

---

## 36. Current Architecture Status

### Defined

* System components
* Module ownership
* Offline-first principle
* Patient/caregiver separation
* AI responsibilities
* Voice/language responsibilities
* Security boundaries
* MVP integration flow

### To be finalized

* Mobile framework
* Backend framework
* Database technology
* Local database technology
* Authentication implementation
* Exact API schemas
* Exact database schema
* Sync conflict-resolution strategy
* Bhashini integration details

These decisions should be finalized before dependent modules are deeply integrated.

---

**Document Owner:** Product + System Architect
**Repository:** CogniCare NER
**Status:** Architecture Baseline — SIH 2026 MVP
