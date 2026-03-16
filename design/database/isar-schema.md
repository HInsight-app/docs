# HInsight — Isar Local Cache Schema

## Overview

Isar serves as an **offline-capable local cache** with PostgreSQL as the primary source of truth.
The app reads from and writes to Isar first, syncing to PostgreSQL when connectivity is restored.
This architecture guarantees eventual consistency via last-write-wins conflict resolution using `updatedAt`.

### Key differences from PostgreSQL schema

| Concept | PostgreSQL | Isar |
|---|---|---|
| Primary key | `uuid` | `localId` (auto-increment int) |
| Server reference | — | `serverId` (nullable UUID, set after sync) |
| Relationships | Join tables (e.g. `DECISION_TAGS`) | Embedded objects or `List<String>` of serverIds |
| Sync tracking | — | `syncStatus` on every collection |
| Conflict resolution | — | `updatedAt` on every collection |

### `syncStatus` values

| Value | Meaning |
|---|---|
| `pending_create` | Created offline, not yet pushed to server |
| `pending_update` | Edited offline, server not yet updated |
| `pending_delete` | Deleted offline, server not yet notified |
| `synced` | In sync with PostgreSQL |

---

## Embedded Objects

### `IsarOption` (embedded inside `IsarDecision`)

Options are never loaded independently — always with their parent decision.
No separate collection needed.

```dart
@embedded
class IsarOption {
  String? serverId;        // PostgreSQL UUID, null until synced
  late String title;
  String? pros;
  String? cons;
  bool isChosen = false;
  late DateTime updatedAt;
}
```

---

## Collections

### `IsarDecision`

```dart
@collection
class IsarDecision {
  Id localId = Isar.autoIncrement;

  @Index(unique: true)
  String? serverId;                  // PostgreSQL UUID, null until synced

  late String userId;
  late String title;
  String? context;
  late int initialConfidence;        // 1–100
  late String status;                // pending | decided | evaluated
  String? commitmentNote;            // Nullable, filled on decide
  DateTime? decidedAt;               // Nullable, set on decide
  DateTime? scheduledReviewDate;     // Nullable
  late DateTime createdAt;
  late DateTime updatedAt;           // Used for last-write-wins conflict resolution

  // Embedded — loaded with the decision automatically
  late List<IsarOption> options;

  // Replaces DECISION_TAGS join table
  late List<String> tagServerIds;

  @Index()
  late String syncStatus;
}
```

### `IsarEvaluation`

```dart
@collection
class IsarEvaluation {
  Id localId = Isar.autoIncrement;

  @Index(unique: true)
  String? serverId;

  @Index()
  late String decisionServerId;      // References IsarDecision.serverId

  late int satisfactionScore;        // 1–10
  late String hindsightNotes;
  late DateTime createdAt;
  late DateTime updatedAt;

  @Index()
  late String syncStatus;
}
```

### `IsarAiInsight`

```dart
@collection
class IsarAiInsight {
  Id localId = Isar.autoIncrement;

  @Index(unique: true)
  String? serverId;

  @Index()
  late String decisionServerId;      // References IsarDecision.serverId

  late String triggerContext;        // pre_decision | post_evaluation
  late String insightText;
  late String primaryBias;           // e.g. Sunk Cost Fallacy
  late String emotionalTone;         // e.g. Anxious
  late String riskLevel;             // low | medium | high
  late int actionabilityScore;       // 1–10
  String? suggestedAlternative;      // Nullable for post_evaluation
  late String status;                // processing | complete | failed
  late DateTime generatedAt;
  late DateTime updatedAt;

  @Index()
  late String syncStatus;
}
```

### `IsarTag`

```dart
@collection
class IsarTag {
  Id localId = Isar.autoIncrement;

  @Index(unique: true)
  String? serverId;

  late String userId;
  late String name;
  late String colorHex;
  late DateTime updatedAt;

  @Index()
  late String syncStatus;
}
```

---

## Design Notes

### `localId` vs `serverId`
- Flutter UI navigates and references records by `serverId` once it exists
- Falls back to `localId` only for `pending_create` records not yet synced
- Prevents broken ID references when a local record gets its server UUID assigned

### `tagServerIds` on `IsarDecision`
- Replaces the entire `DECISION_TAGS` join table from PostgreSQL
- On sync, reconcile this list against the server
- Simpler and more appropriate for a document model

### `decisionServerId` as FK
- Plain string reference instead of Isar's `links` API
- More explicit and easier to reconcile against PostgreSQL UUIDs during sync

### `@Index()` on `syncStatus`
- Makes the reconnect sync query performant
- On reconnect, Repository runs: `SELECT all WHERE syncStatus != synced`
- Most frequent query in the entire app — indexing is non-negotiable

### AI Insights offline behavior
- `status = processing` is set immediately when the user taps generate
- Prevents double-tap race condition (idempotency guard)
- If the request fails, `status = failed` — user can retry
- AI Insights are read-only after generation — no `pending_update` needed, only `pending_create`