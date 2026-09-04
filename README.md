# SIH26003 — AI-Based Cognitive Gaming & Memory Assistance Platform

> **Offline-first · Event-driven · Explainable adaptive difficulty · Privacy-preserving · Open-source intent**

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Core Philosophy](#2-core-philosophy)
3. [Final Minimal Tech Stack](#3-final-minimal-tech-stack)
4. [System Architecture](#4-system-architecture)
5. [Module Structure](#5-module-structure)
6. [Hive Database Schema](#6-hive-database-schema)
7. [State Machines](#7-state-machines)
8. [Interface & API Boundaries](#8-interface--api-boundaries)
9. [Adaptive Difficulty — How It Works](#9-adaptive-difficulty--how-it-works)
10. [Cognitive Scoring Model](#10-cognitive-scoring-model)
11. [Memory Recall + MediaPipe Integration](#11-memory-recall--mediapipe-integration)
12. [Voice & Language Strategy](#12-voice--language-strategy)
13. [Caregiver Dashboard & Alerts](#13-caregiver-dashboard--alerts)
14. [Offline Sync Strategy](#14-offline-sync-strategy)
15. [Security & Privacy](#15-security--privacy)
16. [Elderly UX Constraints](#16-elderly-ux-constraints)
17. [MVP vs Future Roadmap](#17-mvp-vs-future-roadmap)
18. [Sprint Plan — 10 Days](#18-sprint-plan--10-days)
19. [Technical Risks & Fallbacks](#19-technical-risks--fallbacks)
20. [Demo Script](#20-demo-script)
21. [Pitch Anchors](#21-pitch-anchors)
22. [Development Order for Dart/Flutter Beginners](#22-development-order-for-dartflutter-beginners)

---

## 1. Problem Statement

**SIH26003** — Build an AI-based cognitive gaming and memory assistance platform for elderly patients, specifically targeting:

- Cognitive decline screening and monitoring
- Memory assistance via reminders and family recognition
- Multilingual support for North-East Indian (NER) languages
- Offline-first operation on low-end Android hardware
- Caregiver visibility via analytics dashboard and alerts

**Target hardware:** Low-end Android phones and tablets (2015–2020 era devices).  
**Target users:** Elderly patients + their caregivers.  
**Deployment:** Android-primary, tablet-optimised layout.

---

## 2. Core Philosophy

```
OFFLINE-FIRST
  + EVENT-DRIVEN
  + EXPLAINABLE ADAPTATION
  + PRIVACY-FIRST
  + MINIMAL STACK
  + ONE COHERENT VERTICAL SLICE
```

> "Do one thing, do it well, make it composable."  
> Linux-style: lean, modular, optimised for the actual hardware your users own.

**Priority order:**

```
RELIABILITY          > FEATURE COUNT
EXPLAINABILITY       > BLACK-BOX ML
OFFLINE FUNCTION     > CLOUD DEPENDENCY
ONE COHERENT STACK   > MANY FRAMEWORKS
ONE WORKING SLICE    > FOUR HALF-FINISHED FEATURES
```

---

## 3. Final Minimal Tech Stack

| Layer | Technology | Justification |
|---|---|---|
| Language | Dart | Only language needed — Flutter is written in Dart |
| UI Framework | Flutter | One codebase, native ARM binary, offline-capable |
| Local DB | Hive | Embedded, no setup, Dart-native, fast key-value |
| Cloud Backend | Supabase (Postgres) | Auth + DB + realtime in one — no separate backend |
| Voice / TTS | Bhashini API (via adapter) + pre-baked `.mp3` cache | Isolated behind adapter, offline fallback always present |
| On-device Vision | MediaPipe (face detection only) | Family album / Memory Recall stimuli — one-time on upload |
| Charts | fl_chart | One library, Flutter-native |
| Notifications | flutter_local_notifications | No backend needed |
| Auth | Supabase Auth | Already chosen backend — no extra service |
| State Management | Riverpod | Dart-native, testable, granular rebuilds |
| Audio Playback | just_audio | Pre-baked `.mp3` prompt playback |

### Removed — and why

| Removed | Reason |
|---|---|
| React Native | Team is not JS-heavy — adds a language, removes nothing |
| Node.js / Python backend | Supabase replaces the need entirely |
| Multiple databases | Hive local + Supabase cloud — that is sufficient |
| Redis / Kafka | No throughput requirement at this scale |
| Docker / Kubernetes | No deployment infra needed for MVP |
| ML training pipeline | Rule-based adaptive policy covers MVP — ML interface exists for future |
| LLM API | Explainability strings are deterministic, not generated |
| Vector database | Face embeddings stored locally in Hive — no cloud vector store |
| Multiple cloud providers | One Supabase project is enough |
| Multiple state-management frameworks | Riverpod only |

### Languages used

```
Dart       — everything
SQL        — only inside Supabase schema definitions
```

That is all.

---

## 4. System Architecture

```
┌─────────────────────────────────────┐
│        PATIENT / TABLET UI          │
│   (Flutter — Android-first)         │
└────────────────┬────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│       APPLICATION MODULES           │
│  cognitive_games  │  reminders      │
│  family_album     │  voice          │
│  auth             │  profile        │
└────────────────┬────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│    EVENT + INTELLIGENCE LAYER       │
│  GameEvent  │  AdaptivePolicy       │
│  DomainScorer │ AlertEngine         │
│  ExplainabilityBuilder              │
└────────────────┬────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      LOCAL-FIRST DATA LAYER         │
│  Hive boxes — source of truth       │
│  game_events │ reminders            │
│  difficulty_state │ sync_queue      │
└────────────────┬────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│        OFFLINE SYNC ENGINE          │
│  ConnectivityManager                │
│  SyncQueue → idempotent upserts     │
│  operation_id deduplication         │
└────────────────┬────────────────────┘
                 ↓ (only when online)
┌─────────────────────────────────────┐
│      CLOUD BACKEND — SUPABASE       │
│  Postgres │ Auth │ Realtime         │
│  delta sync only — never full push  │
└────────────────┬────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│       CAREGIVER DASHBOARD           │
│  fl_chart │ alert feed              │
│  evidence strings │ reminder log    │
└─────────────────────────────────────┘
```

---

## 5. Module Structure

```
lib/
├── core/
│   ├── models/
│   │   ├── game_event.dart          ← core event model
│   │   └── reminder_event.dart      ← reminder event model
│   ├── database/
│   │   └── hive_service.dart        ← box registry, init, access
│   └── sync/
│       ├── sync_queue.dart          ← local operation queue
│       └── connectivity_manager.dart← online/offline detection
│
├── features/
│   ├── auth/
│   │   └── auth_service.dart        ← Supabase Auth, 2 roles
│   ├── cognitive_games/
│   │   ├── game_engine.dart         ← reusable pipeline (all games share this)
│   │   └── memory_recall/
│   │       ├── memory_game.dart     ← stimulus presentation + response capture
│   │       └── face_stimulus.dart   ← loads MediaPipe face crops as stimuli
│   ├── reminders/
│   │   └── reminder_engine.dart     ← medicine reminders, local notifications
│   └── caregiver/
│       ├── dashboard_view.dart      ← fl_chart + alert status card
│       └── alert_engine.dart        ← GREEN / YELLOW / RED logic
│
├── intelligence/
│   ├── adaptive_policy.dart         ← abstract interface + RuleBasedPolicy
│   └── domain_scorer.dart           ← memory domain score calculation
│
└── integrations/
    ├── bhashini/
    │   ├── bhashini_adapter.dart    ← TTS/STT API calls
    │   └── audio_cache.dart         ← pre-baked .mp3 fallback loader
    └── mediapipe/
        └── face_detection_service.dart ← detect → crop → embed (upload only)
```

**~16 Dart files. Entire MVP.**

---

## 6. Hive Database Schema

### Box: `game_events` — APPEND-ONLY

```dart
{
  event_id:         String,   // UUID
  patient_id:       String,
  session_id:       String,
  game_id:          String,
  domain:           String,   // 'memory' | 'attention' | 'routine' | 'pattern'
  difficulty_before: int,     // 1 | 2 | 3
  stimulus_id:      String,
  response:         String,
  correctness:      bool,
  response_time_ms: int,
  hints_used:       int,
  difficulty_after: int,
  timestamp:        DateTime,
  synced:           bool,
}
```

### Box: `cognitive_profile` — VERSIONED UPSERT

```dart
{
  patient_id:      String,
  memory_score:    double,
  attention_score: double,
  routine_score:   double,
  pattern_score:   double,
  consistency:     double,
  engagement:      double,
  baseline:        Map<String, double>,
  rolling_avg:     Map<String, double>,
  last_updated:    DateTime,
  version:         int,
}
```

### Box: `difficulty_state` — MUTABLE

```dart
{
  key: patient_id + '_' + domain,
  current_level:        int,   // 1–3
  consecutive_correct:  int,
  consecutive_wrong:    int,
  last_updated:         DateTime,
}
```

### Box: `reminders` — VERSIONED

```dart
{
  reminder_id:          String,
  patient_id:           String,
  type:                 String,   // 'P1_medicine' | 'P2_appointment' | 'P3_hydration'
  title:                String,
  body_text:            String,
  scheduled_time:       DateTime,
  repeat_rule:          String,
  status:               String,   // 'pending' | 'ack' | 'snoozed' | 'dismissed' | 'escalated'
  ack_time:             DateTime?,
  escalate_after_mins:  int,
  synced:               bool,
  version:              int,
}
```

### Box: `reminder_events` — APPEND-ONLY

```dart
{
  event_id:    String,
  reminder_id: String,
  patient_id:  String,
  action:      String,   // 'fired' | 'ack' | 'snooze' | 'dismiss' | 'escalated'
  timestamp:   DateTime,
  synced:      bool,
}
```

### Box: `face_embeddings` — LOCAL ONLY, NEVER SYNCED

```dart
{
  face_id:     String,
  patient_id:  String,
  cluster_id:  String,
  photo_path:  String,   // local filesystem path only
  person_name: String,
  labelled_by: String,
  created_at:  DateTime,
}
```

### Box: `sync_queue` — OPERATIONAL

```dart
{
  operation_id:   String,   // UUID — idempotency key
  device_id:      String,
  entity_type:    String,   // 'game_event' | 'reminder' | 'profile'
  entity_id:      String,
  operation_type: String,   // 'append' | 'upsert'
  payload_json:   String,
  created_at:     DateTime,
  status:         String,   // 'pending' | 'synced' | 'failed'
  retry_count:    int,
}
```

### Box: `caregiver_alerts` — APPEND-ONLY

```dart
{
  alert_id:          String,
  patient_id:        String,
  domain:            String,
  level:             String,   // 'GREEN' | 'YELLOW' | 'RED'
  current_score:     double,
  baseline:          double,
  change_pct:        double,
  sessions_observed: int,
  trend:             String,   // 'declining' | 'stable' | 'improving'
  reason_text:       String,   // deterministic explainability string
  created_at:        DateTime,
  synced:            bool,
}
```

---

## 7. State Machines

### 7.1 Game Session

```
IDLE
  → [patient taps game]
LOADING
  → [profile + difficulty loaded from Hive]
PRESENTING_STIMULUS
  → [response captured OR 8s timeout]
EVALUATING
  → [correctness + time + hints calculated]
  → [GameEvent generated]
ADAPTING
  → [AdaptivePolicy.evaluate(performance_vector)]
  → [difficulty_delta applied]
  → [explainability string generated if delta ≠ 0]
PERSISTING
  → [Hive write — game_events box]
  → [enqueue to sync_queue]
  → [more stimuli?] → back to PRESENTING_STIMULUS
  → [session complete?] → SESSION_SUMMARY
SESSION_SUMMARY
  → [score displayed]
  → [voice prompt played — pre-baked .mp3]
  → IDLE
```

### 7.2 Adaptive Difficulty

```
INPUT: performance_vector (last 5–10 game_events)

EVALUATE:
  consecutive_wrong ≥ 2  AND  accuracy < 50%
    → DECREASE:  difficulty_delta = -1
    → hysteresis lock (skip next 3 events before re-evaluating)
    → explainability: "Repeated low accuracy detected. Reducing difficulty."

  consecutive_correct ≥ 3  AND  accuracy > 80%
    → INCREASE:  difficulty_delta = +1
    → hysteresis lock (skip next 3 events before re-evaluating)
    → explainability: "Sustained high performance. Increasing difficulty."

  otherwise
    → HOLD:  difficulty_delta = 0

CLAMP: level stays within [1, 3]

OUTPUT: new difficulty level + explainability string
```

### 7.3 Reminder

```
SCHEDULED
  → [scheduled_time fires]
NOTIFIED
  → push notification + voice prompt (.mp3) + visual card
  → [patient acknowledges within window]
      → ACKNOWLEDGED → ReminderEvent(action: 'ack') → Hive
  → [patient snoozes]
      → SNOOZED → re-schedule +15min → back to NOTIFIED
  → [patient dismisses]
      → DISMISSED → ReminderEvent(action: 'dismiss') → Hive
  → [no response within escalate_after_mins]
      → ESCALATED → caregiver push notification
      → ReminderEvent(action: 'escalated') → Hive
```

### 7.4 Voice / Bhashini Fallback

```
OUTPUT (TTS):
  application needs to speak
    → [Bhashini online + available]
        → BhashiniAdapter.synthesize(text, language)
        → playback
    → [Bhashini unavailable OR offline]
        → AudioCache.play(prompt_key, language)   ← pre-baked .mp3
    → [no cache hit]
        → text + visual display only

INPUT (STT) — future:
  mic input
    → [Bhashini STT available]
        → transcribe → intent router → action
    → [unavailable]
        → text input fallback
```

### 7.5 Offline Sync

```
ANY APP WRITE:
  → write to Hive box first (always, unconditionally)
  → enqueue SyncOperation to sync_queue box

CONNECTIVITY_MANAGER (polls every 30s):
  OFFLINE:
    → queue accumulates, no action

  ONLINE:
    → dequeue pending SyncOperations (FIFO)
    → POST to Supabase (idempotent upsert by operation_id)
    → 200 OK
        → mark synced: true in Hive
    → 4xx client error
        → log, discard (bad payload — don't retry)
    → 5xx / timeout
        → retry with exponential backoff
        → max 3 retries
        → after 3 failures → mark status: 'failed'
```

### 7.6 Caregiver Alert

```
NEW game_event arrives for patient + domain:

CHECK sessions_observed:
  < 10 sessions
    → NO ALERT (insufficient baseline)
    → display: "Collecting baseline data"

  ≥ 10 sessions:
    → compute rolling_avg  (last 7 sessions)
    → compute baseline     (first 10 sessions or all prior)
    → deviation = (baseline - rolling_avg) / baseline × 100

  deviation < 10%
    → GREEN — normal variation, no alert

  10% ≤ deviation < 25%,  persisted ≥ 3 sessions
    → YELLOW — possible decline
    → alert payload generated
    → caregiver notified

  deviation ≥ 25%,  persisted ≥ 3 sessions
    → RED — significant persistent deviation
    → alert payload generated
    → caregiver push notification

ALERT PAYLOAD:
  domain, current_score, baseline, change_pct,
  sessions_observed, trend, reason_text (deterministic string)

EXAMPLE reason_text:
  "Memory performance declined 31% relative to personal
   baseline across 5 sessions. Consistent downward trend detected."
```

---

## 8. Interface & API Boundaries

All external dependencies live behind abstract interfaces.  
The rest of the app never imports Bhashini, MediaPipe, or Supabase directly.

```dart
// Adaptive Policy — swappable without touching GameEngine
abstract class AdaptivePolicy {
  int evaluate(PerformanceVector v);     // returns -1 | 0 | +1
  String explainLastDecision();
}

class RuleBasedPolicy implements AdaptivePolicy { ... }  // MVP
// class MLPolicy implements AdaptivePolicy { ... }      // Future

// Voice — Bhashini isolated
abstract class TTSService {
  Future<void> speak(String text, Language lang);
}
class BhashiniAdapter implements TTSService { ... }
class CachedAudioFallback implements TTSService { ... }

// Face detection — MediaPipe isolated
abstract class FaceDetectionService {
  Future<List<FaceCrop>> detect(Image img);
}
class MediaPipeFaceService implements FaceDetectionService { ... }

// Sync — Supabase isolated
abstract class RemoteSync {
  Future<SyncResult> push(List<SyncOperation> ops);
}
class SupabaseSyncAdapter implements RemoteSync { ... }
```

---

## 9. Adaptive Difficulty — How It Works

No ML training. No external API. Pure Dart logic behind an interface.

```
PerformanceVector:
  accuracy              (0.0 – 1.0)
  avg_response_time_ms
  consecutive_correct
  consecutive_wrong
  hints_used
  recent_trend          (last N events slope)
  current_difficulty    (1 | 2 | 3)

RuleBasedPolicy logic:
  IF consecutive_wrong ≥ 2 AND accuracy < 0.5  → -1
  IF consecutive_correct ≥ 3 AND accuracy > 0.8 → +1
  ELSE → 0

Hysteresis:
  After any delta ≠ 0 → lock evaluation for next 3 events
  Prevents oscillation on single mistakes

Difficulty levels (Memory Recall example):
  Level 1: 3 photos shown, choose 1 correct from 2 options, no delay
  Level 2: 4 photos shown, choose 1 correct from 3 options, 3s delay
  Level 3: 5 photos shown, choose 1 correct from 4 options, 5s delay + interference
```

**Presentation wording:**
> "Explainable adaptive intelligence for MVP. ML-ready architecture for future validated personalisation."

---

## 10. Cognitive Scoring Model

**Two separate concerns — do not conflate them:**

### A. Game Performance (per session)

```
memory_score    = weighted(correctness, response_time, hints_used)
attention_score = future domain
routine_score   = future domain
pattern_score   = future domain
```

### B. Longitudinal Trend (across sessions)

```
baseline        = average of first 10 sessions (or all prior sessions)
rolling_avg     = average of last 7 sessions
deviation       = (baseline - rolling_avg) / baseline
persistence     = how many consecutive sessions deviation holds
trend_direction = 'improving' | 'stable' | 'declining'
```

### CognitiveProfile fields

```
memory_score, attention_score, routine_score, pattern_score
consistency  (variance across sessions)
engagement   (session completion rate)
```

### Critical framing

```
✅ Say:  "Performance monitoring and screening support"
✅ Say:  "Caregiver-facing cognitive trend indicator"
❌ Never: "Dementia diagnosis"
❌ Never: "Clinical MoCA score"
```

MoCA/ADAS-Cog used only as **domain/construct grounding** — not claimed as clinical output.

---

## 11. Memory Recall + MediaPipe Integration

One implementation. Two features ticked.

```
SETUP (caregiver, one-time):
  Caregiver uploads family photo
        ↓
  MediaPipe detects face → crops it
        ↓
  Crop stored locally (Hive: face_embeddings box)
        ↓
  Caregiver types person name → saved with crop
        ↓
  Raw photo never uploaded to cloud

GAME SESSION (patient):
  Memory Recall game loads face crops as stimuli
        ↓
  Shows face photo: "Who is this person?"
        ↓
  Patient taps from 4 name options
        ↓
  GameEvent generated → AdaptivePolicy evaluates
        ↓
  Difficulty adjusts → persisted to Hive
```

**MediaPipe runs once on upload only. Never real-time. Never on scroll.**  
Optimised for low-end hardware — no continuous inference cost.

---

## 12. Voice & Language Strategy

### Scope for MVP

| Language | Status |
|---|---|
| English | Full — all UI text, all prompts |
| Assamese | Pre-baked `.mp3` audio prompts (15 files) |
| Manipuri / Khasi / Mizo | Roadmap — BhashiniAdapter ready to extend |

### Pre-baked Assamese audio files (bundle in assets)

```
who_is_this.mp3         → "এইজন কোন?"
correct.mp3             → "সঠিক!"
wrong_try_again.mp3     → "ভুল। পুনৰ চেষ্টা কৰক।"
medicine_time.mp3       → "আপোনাৰ ঔষধ খোৱাৰ সময় হৈছে।"
excellent_work.mp3      → "অসাধাৰণ কাম!"
game_start.mp3          → game instruction prompt
level_up.mp3            → difficulty increase notification
level_down.mp3          → difficulty decrease notification
session_complete.mp3    → session summary prompt
+ 6 additional game prompts
```

**Total: ~15 `.mp3` files. One afternoon of generation via Bhashini TTS. Zero demo risk.**

### LanguageManager

Single point of language logic. No language hardcoding anywhere else in the app.

```dart
class LanguageManager {
  Language currentLanguage;
  Future<void> speak(String promptKey);
  String getText(String promptKey);
}
```

---

## 13. Caregiver Dashboard & Alerts

### Dashboard screens

```
Patient selector
      ↓
Domain scores (memory — MVP, others stubbed)
      ↓
7-day rolling trend (fl_chart line graph)
      ↓
Alert status (GREEN / YELLOW / RED card)
      ↓
Supporting evidence string
      ↓
Reminder adherence log
```

### Alert evidence example

```
Level:    YELLOW
Domain:   Memory
Current:  58% accuracy (7-day avg)
Baseline: 74% accuracy
Change:   -21.6%
Sessions: 8 observed since baseline
Trend:    Declining for 4 consecutive sessions
Reason:   "Memory performance declined 22% relative to personal
           baseline across 4 sessions. Consistent downward trend."
```

All strings are **deterministic and generated from data** — no LLM involved.

---

## 14. Offline Sync Strategy

```
Rule 1: Hive write ALWAYS happens first — before any network call.
Rule 2: Every write enqueues a SyncOperation with a UUID.
Rule 3: Supabase receives idempotent upserts keyed by operation_id.
Rule 4: Never push full patient state — only delta operations.
Rule 5: face_embeddings box NEVER syncs — local only always.

Conflict policy by data type:
  game_events     → append-only, no conflict possible
  reminders       → versioned, last-valid-write wins
  cognitive_profile → versioned upsert, server version checked
  caregiver_alerts → append-only
```

---

## 15. Security & Privacy

| Concern | Implementation |
|---|---|
| Auth | Supabase Auth — email/password, JWT |
| Roles | Patient / Caregiver — RBAC in Supabase RLS |
| Local sensitive data | Hive encrypted box (AES key in Flutter Secure Storage) |
| Network | HTTPS only — Supabase enforces this |
| Face data | Never leaves device — local Hive box, no sync |
| Raw photos | Never uploaded — face crops only, local |
| Cloud data | Minimal — scores, events, alerts only |

**Least-privilege access:** Caregiver sees only their linked patient's data.  
Patient cannot access caregiver management screens.

---

## 16. Elderly UX Constraints

These are **functional requirements**, not design preferences.

```
✅ Minimum 56px touch targets (all buttons, all options)
✅ High contrast — dark background, large light text
✅ Minimal on-screen text — icons + audio carry the meaning
✅ No countdown timers — timeout is silent, no pressure
✅ No swipe-dependent interactions — tap only
✅ Large, clear icons
✅ Simple flat navigation — no nested menus
✅ Voice + visual redundancy on every prompt
✅ Obvious correct/incorrect feedback (colour + sound + text)
✅ Recoverable accidental exits — confirm dialogs
✅ Test on low-end Android hardware before demo day
```

---

## 17. MVP vs Future Roadmap

| Feature | MVP (Pitch) | Future |
|---|---|---|
| Memory Recall game — full pipeline | ✅ | — |
| Attention / Routine / Pattern games | Stubbed UI | ✅ |
| Rule-based adaptive difficulty | ✅ | — |
| ML adaptive policy | Interface exists | ✅ swap in |
| Caregiver dashboard + alerts | ✅ | — |
| Medicine reminder (P1) | ✅ | — |
| Appointment / hydration reminders | Stubbed | ✅ |
| English UI | ✅ | — |
| Assamese pre-baked audio | ✅ | — |
| Manipuri / Khasi / Mizo | Adapter ready | ✅ |
| Live Bhashini TTS/STT | Adapter exists | ✅ |
| MediaPipe face → name quiz | ✅ | — |
| Real-time face recognition | ❌ | ✅ |
| Offline-first Hive persistence | ✅ | — |
| Supabase cloud sync | ✅ | — |
| Multi-device patient | ❌ | ✅ |
| Clinical MoCA scoring | ❌ never without validation | Clinical Phase 2 |
| Open-source community languages | Architecture ready | ✅ |

---

## 18. Sprint Plan — 10 Days

| Day | Deliverable | Critical note |
|---|---|---|
| **1** | Hive schema locked, box registry, GameEvent + ReminderEvent models, Supabase project + Auth | Lock schema Day 1 or rebuild twice |
| **2** | GameEngine pipeline, session state machine, AdaptivePolicy interface + RuleBasedPolicy | Architecture before UI |
| **3** | Memory Recall game — stimulus presentation, response capture, face crop loader | First working vertical slice |
| **4** | Difficulty changes live on screen, explainability strings visible, hysteresis working | This is your demo moment |
| **5** | SyncQueue, ConnectivityManager, offline write → online push proof | Core infra |
| **6** | ReminderEngine, flutter_local_notifications, medicine reminder, escalation | P1 feature |
| **7** | BhashiniAdapter, AudioCache, 15 pre-baked Assamese .mp3 files, LanguageManager | Isolate external risk early |
| **8** | MediaPipe face detection on upload, face crop → Hive store, name-matching game stimuli | One-time detection only |
| **9** | Caregiver dashboard, fl_chart line graph, AlertEngine GREEN/YELLOW/RED, Supabase sync | Close the loop |
| **10** | Elderly UX pass (56px targets, contrast, no timers), full demo rehearsal, offline→sync proof | Win the room |

### Most important early vertical slice

```
ONE GAME
  → SCORE
  → ADAPTIVE DIFFICULTY CHANGES LIVE
  → SAVES OFFLINE
  → SYNCS WHEN ONLINE
  → CAREGIVER SEES GRAPH
  → PERSISTENT DECLINE → EXPLAINABLE ALERT
```

Get this working before touching any other feature.

---

## 19. Technical Risks & Fallbacks

| Risk | Probability | Fallback |
|---|---|---|
| Bhashini Khasi/Mizo not in live catalogue | High | Pre-bake all prompts, adapter returns cached file |
| Bhashini latency >2s during demo | Medium | AudioCache fires if no response within 1.5s |
| MediaPipe slow on low-end Android | Medium | Detect once on upload only — never real-time |
| Hive box corruption on crash | Low | Enqueue to sync_queue before Hive write |
| Supabase free tier rate limit | Low | Batch sync ops — not per-event push |
| Judge asks "is this clinical diagnosis?" | Certain | "Performance screening support — not diagnosis. Clinical validation is Phase 2." |
| Team can't finish all 4 games | High | 1 complete + 3 stubbed with placeholder UI — planned, not forgotten |
| Demo network unreliable | Medium | Full offline demo path prepared — airplane mode ON by default |

---

## 20. Demo Script

**2 minutes. In this exact order.**

```
1. Open app as patient
   → tap Memory Recall

2. Answer 3 questions wrong deliberately
   → show difficulty drop on screen
   → show explainability string: "Repeated low accuracy detected."

3. Answer 3 questions correct
   → show difficulty increase on screen
   → show explainability string: "Sustained high performance."

4. Turn airplane mode ON
   → play another round
   → show data saves fine (Hive — offline)

5. Turn airplane mode OFF
   → show sync indicator
   → open caregiver tab
   → show line graph update
   → show alert status

6. Say:
   "Every decision this system makes — why difficulty changed,
    why a caregiver alert fires — is a deterministic, readable string.
    No black box. No cloud dependency. Runs on any Android device."
```

---

## 21. Pitch Anchors

| Judge question | Your answer |
|---|---|
| "Why not use an existing app?" | "Existing apps don't work offline, don't support NER languages, and aren't designed for elderly Indian users on budget hardware." |
| "Is this clinical diagnosis?" | "No. Performance screening and caregiver support. Clinical validation is a Phase 2 milestone with domain experts." |
| "Why Flutter?" | "One codebase, one language, compiles to native ARM binary, runs smoothly on 2015 Android hardware." |
| "What's the AI?" | "Explainable rule-based adaptive difficulty — ML-ready interface exists for future validated personalisation." |
| "Why open source?" | "NER communities need to extend language and content support. A closed product can't do that." |
| "What if Bhashini is down?" | "Pre-baked audio fallback — demo works fully offline. External dependency isolated behind adapter." |

---

## 22. Development Order for Dart/Flutter Beginners

### Step 1 — Dart only (3 days, no Flutter yet)

Read: [dart.dev/language](https://dart.dev/language)

Focus on:
- Types + null safety
- async / await (Hive and Supabase are async)
- Abstract classes and interfaces (your entire architecture)
- Collections + functional ops (scoring logic)

**Exercise:** Write `GameEvent`, `ReminderEvent`, `AdaptivePolicy`, and `RuleBasedPolicy` as plain `.dart` files. Run with `dart run`. No Flutter, no UI.

### Step 2 — Flutter in this order

```
1. Widget tree — Stateless vs Stateful
2. Riverpod — state management
3. Navigation between screens
4. Hive integration
5. fl_chart — dashboard graph
6. flutter_local_notifications — reminders
7. Supabase client — auth + sync
8. MediaPipe plugin — face detection
9. just_audio — pre-baked .mp3 playback
```

One at a time. Not all at once.

### One rule

> Draw the flow on paper before writing any code.  
> If you can't draw it clearly, the code will be confused too.

---

## Final Architecture Statement

> "An offline-first, event-driven cognitive assistance platform with explainable adaptive difficulty, English and Assamese voice interaction, privacy-preserving on-device family recognition, longitudinal cognitive-performance monitoring, and caregiver alerts — built on a minimal, modular, open-source Flutter stack optimised for low-end Android hardware."

---

*SIH26003 — Built lean. Built to last. Built for the people who actually need it.*
