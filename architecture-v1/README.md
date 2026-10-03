# Sudoku Platform

A sudoku application where you can **scan a paper puzzle and solve it**, **play solo against the computer**, or **play with friends** in collaborative or competitive rooms with voice, chat, and an AI hint assistant.

> **Status:** v1 — modular monolith (single server). A scalable v2 microservice design is described in [Roadmap](#9-roadmap-v2).

---

## Table of Contents

1. [Features](#1-features)
2. [Scan & Solve](#2-scan--solve)
3. [Play Modes](#3-play-modes)
4. [Scoring System](#4-scoring-system)
5. [AI Assistant](#5-ai-assistant)
6. [Architecture (v1)](#6-architecture-v1)
7. [APIs & Classes](#7-apis--classes)
8. [Suggested Tech Stack](#8-suggested-tech-stack)
9. [Roadmap (v2)](#9-roadmap-v2)
10. [Open Design Decisions](#10-open-design-decisions)

---

## 1. Features

| Area | Summary |
|---|---|
| **Scan & Solve** | Take a photo of a sudoku, detect the numbers (OpenCV + Tesseract), correct mistakes, get the solution or step hints |
| **Play with Computer** | Choose difficulty and timer type (stopwatch / countdown) |
| **Play with Friends** | Public or private rooms, up to 5 players, host-controlled |
| **Collab Mode** | Everyone solves one shared board; voice chat, text chat, AI hint assistant |
| **Competitive Mode** | Same puzzle for everyone, independent boards, ranked by score and average time |
| **Scoring** | Difficulty × speed × accuracy × hint usage, plus placement bonuses |

---

## 2. Scan & Solve

The user photographs a printed sudoku. The server returns a 9×9 grid for the user to confirm, then solves it.

**Pipeline**

1. **Preprocess (OpenCV):** grayscale → Gaussian blur → adaptive threshold
2. **Find the grid:** largest 4-corner contour → perspective warp to a square
3. **Split:** divide into 81 cells, trim borders, remove leftover grid lines
4. **Blank detection:** cells with too few foreground pixels are marked empty (skips OCR)
5. **Recognise digits (Tesseract):** per-cell, single character mode (`--psm 10`, whitelist `123456789`), with confidence
6. **Validate:** check for row/column/box conflicts; flag low-confidence cells
7. **User confirmation:** UI highlights uncertain cells for quick correction
8. **Solve:** backtracking solver with constraint propagation; detects *unsolvable* and *multiple-solution* puzzles

**Notes**
- OCR is CPU-heavy: run it on a bounded executor (dedicated thread pool), not on the request thread.
- Tesseract struggles with handwriting and poor lighting. If accuracy is not enough, swap `DigitRecognizer` for a small MNIST-style CNN — the interface stays the same.
- Scan & Solve is a utility mode: **no score, no leaderboard**.

---

## 3. Play Modes

### 3.1 Play with Computer (solo)

- Choose **difficulty**: Easy / Medium / Hard / Expert
- Choose **timer**:
  - **Stopwatch** — counts up; faster is better
  - **Countdown** — pick a duration; running out ends the game (partial credit applies)
- Features: pencil notes, hints (penalised), pause (stopwatch only), mistake counter
- The server owns the clock and the solution; the client only sends moves

### 3.2 Play with Friends

**Rooms**

| Property | Description |
|---|---|
| `roomId` | Auto-generated, unique |
| `name` | Chosen by host |
| `visibility` | Public or Private |
| `maxSeats` | 5 |
| `mode` | Collab or Competitive |
| `timerMode` | Stopwatch or Countdown |

**Public room** — listed for everyone. No credentials. First come, first in, as long as a seat is free.

**Private room**
- Not listed. Host decides the credential policy:
  - `roomId` **or** `roomName`, or
  - **both** `roomId` **and** `roomName` required
- Even with valid credentials, **the host must approve** each joiner (protects against guessing).
- The host can wait in the lobby, edit settings, approve/reject/kick players, or start the match any time.

**Join flow (private)**

```
Player enters roomId / name  →  server validates credentials  →  JoinRequest(PENDING)
→ host gets real-time notification  →  Approve → player enters room
                                    →  Reject  → player is notified
```

**Room screen** shows the host, all members (name, avatar, ready/connected status), settings, and live state.

### 3.3 Collaborative Mode

- All players work on **one shared board** in real time (live cell updates, coloured cursors)
- **Voice chat** (WebRTC) and **text chat** inside the room
- **AI assistant** is a room member: players can ask by chat (`@AI …`) or push-to-talk; it only gives hints, never the solution (see [AI Assistant](#5-ai-assistant))
- One result for the whole team

### 3.4 Competitive Mode

- Host selects number of **rounds** (1–5)
- Every player gets the **same puzzle** on their **own board**
- Opponents see only each other's **progress %**, never digits
- Ranked by **total score**, with **average solve time** as tiebreaker (and shown alongside)
- Chat and voice available; **the AI assistant is disabled in competitive mode** to keep it fair *(change if you want AI hints to be penalised instead)*

---

## 4. Scoring System

### 4.1 Per-puzzle score

```
Score = Base × TimeMultiplier × AccuracyMultiplier × HintMultiplier
```

(Rounded to an integer; a **completed** puzzle never scores below `20% × Base`.)

**Base points and par time**

| Difficulty | Base | Par time | Default countdown |
|---|---:|---:|---:|
| Easy | 100 | 6 min | 15 min |
| Medium | 200 | 12 min | 25 min |
| Hard | 350 | 20 min | 40 min |
| Expert | 500 | 30 min | 60 min |

**Multipliers**

| Factor | Formula | Range |
|---|---|---|
| **Time** | `clamp(1.5 − 0.5 × (t / par), 0.5, 1.5)` | 0.5 – 1.5 |
| **Accuracy** | `max(0.5, 1 − 0.10 × mistakes)` | 0.5 – 1.0 |
| **Hints** | `max(0.6, 1 − 0.05 × hints)` | 0.6 – 1.0 |

- `t` = solve time in seconds (for countdown mode: `duration − remaining`). Pause time is excluded.
- A **mistake** = placing a digit that does not match the solution (counted once per cell+value, validated by the server).
- A **hint** = a hint button press or an AI hint delivered.
- **Flawless bonus:** `+10% × Base` if mistakes = 0 and hints = 0.

**Worked examples**

| Case | Calculation | Score |
|---|---|---:|
| Medium, 8 min, 1 mistake, 0 hints | 200 × 1.167 × 0.9 × 1.0 | **210** |
| Hard, 25 min, 3 mistakes, 2 hints | 350 × 0.875 × 0.7 × 0.9 | **193** |
| Easy, 3 min, perfect | 100 × 1.25 × 1 × 1 + 10 | **135** |

### 4.2 Solo mode specifics

- **Stopwatch:** formula above.
- **Countdown:** same formula; if time runs out, the game is **failed** and awards **partial credit**:
  `PartialScore = 0.3 × Base × (correctCellsFilled / emptyCellsAtStart)`
- Abandoned games score 0.

### 4.3 Competitive mode

Each round, every player gets their per-puzzle score (finish = formula, **DNF = 0**), plus a **placement bonus** based on finishing order:

| Placement | 1st | 2nd | 3rd | 4th | 5th |
|---|---:|---:|---:|---:|---:|
| Bonus | +25% Base | +15% Base | +10% Base | +5% Base | 0 |

(Only players who finished get a bonus.)

**Match standings** (multi-round):

1. **Total score** (sum of rounds) — higher wins
2. **Average solve time** over finished rounds — lower wins
3. **Total mistakes** — fewer wins
4. Earliest finish time in the final round

Average solve time is also exposed as its own leaderboard column.

### 4.4 Collaborative mode

The team gets **one score**, using the same formula with team-level inputs:

- `t` = team solve time, compared against an **adjusted par**: `par × (1 − 0.10 × (players − 1))` (a 5-player team has 60% of the par time, so big teams are not rewarded just for being big)
- `mistakes` = all players' mistakes combined
- `hints` = AI hints + hint-button uses
- Everyone present gets the team score
- **Anti-leech rule:** a player who contributed **< 5%** of correct cells receives 50% of the team score
- **MVP badge** goes to the highest correct-cell contributor (cosmetic)

### 4.5 XP & levels (profile progression)

- `XP = round(score / 10)`; competitive winners get `+20%` XP
- `Level = floor(sqrt(totalXP / 50)) + 1`
- **Global leaderboards:** weekly / all-time, by total score, and by best average solve time per difficulty (competitive & solo separately)

### 4.6 Anti-cheat basics

- Timer, solution, and validation are **server-side only**; the solution never leaves the server
- Moves arriving faster than a human threshold (e.g. < 150 ms between placements, repeatedly) are flagged
- Competitive progress messages contain a percentage only
- Puzzle IDs are single-use per round

---

## 5. AI Assistant

The AI is a **member of the collaborative room** that replies to chat or voice questions with **hints, never the solution**.

**Design: deterministic logic first, LLM for wording only**

```
User question ──► HintEngine (LogicalSolver) ──► structured hint ──► LLM (friendly wording) ──► SolutionLeakGuard ──► room
```

- `LogicalSolver` finds the next logical technique on the *current shared board* (e.g. naked single in box 5).
- Hint levels (the more levels the team uses, the larger the penalty):
  1. **Nudge** — "Look at row 4, something is forced there."
  2. **Technique** — "Box 5 has a hidden single."
  3. **Location** — names the specific cell/unit, **still not the digit**
- `SolutionLeakGuard` blocks any LLM output that reveals a digit for an empty cell.
- Rate limit: 1 hint per 30 s per room; maximum hint budget per match.
- Voice: push-to-talk clip → speech-to-text → same pipeline.

---

## 6. Architecture (v1)

**v1 is a modular monolith**: one deployable server with strictly separated modules, communicating in-process. See the diagram: [`docs/architecture-v1.mermaid`](docs/architecture-v1.mermaid).

Key decisions:

- **REST** for CRUD flows, **WebSocket** for real-time (room state, moves, chat, timer sync)
- **PostgreSQL** as the system of record; **Redis optional** (cache, rate limit, leaderboards)
- **Live room state in memory** (`RoomRegistry`), persisted on important transitions
- **Voice:** WebRTC peer-to-peer mesh (fine for ≤ 5 peers) with STUN/TURN; the server only relays signalling
- **OCR in a bounded executor** so image processing never blocks real-time traffic
- **Internal event bus** (in-process) so modules (match → scoring → leaderboard) are already decoupled — this is what becomes Kafka in v2
- **Server-authoritative** timer and validation

Limitation to know: because live room state is in memory, v1 runs as a **single instance** (vertical scaling only).

---

## 7. APIs & Classes

Full reference: [`docs/api-and-classes.md`](docs/api-and-classes.md)

API groups (REST `/api/v1` + WebSocket `/ws`):

| Group | Examples |
|---|---|
| Auth & User | `/auth/register`, `/auth/login`, `/auth/guest`, `/users/me` |
| Scan | `/scan`, `/scan/{id}/grid`, `/scan/{id}/solve` |
| Puzzle | `/puzzles/generate` |
| Solo game | `/games`, `/games/{id}/cells`, `/games/{id}/submit` |
| Rooms | `/rooms`, `/rooms/{id}/join`, `/rooms/join-requests`, `/rooms/{id}/start` |
| Match & Chat | `/rooms/{id}/match`, `/rooms/{id}/messages` |
| AI | `/rooms/{id}/ai/hint`, `/rooms/{id}/ai/voice` |
| Leaderboard | `/rooms/{id}/leaderboard`, `/leaderboards` |
| WebSocket | `room:state`, `cell:updated`, `chat:message`, `ai:message`, `rtc:*`, … |

Main classes: entities (`User`, `Room`, `RoomMember`, `JoinRequest`, `Match`, `MatchRound`, `Puzzle`, …), services (`ImageScanService`, `SudokuGenerator/Solver/Validator`, `RoomService`, `JoinRequestService`, `MatchService` with Collab/Competitive strategies, `ScoringService`, `HintEngine`, `AIAssistantService`, …), and repositories.

---

## 8. Suggested Tech Stack

Backend: **Java + Spring Boot 4.x** (Spring Framework 7). Step-by-step build plan: [`docs/implementation-guide-v1.md`](docs/implementation-guide-v1.md).

| Concern | Choice |
|---|---|
| Language / framework | Java 25 LTS (21 works), Spring Boot 4.x, Spring MVC with virtual threads |
| Modular monolith | **Spring Modulith 2.0** (module boundaries verified in tests, application events) |
| Real-time | STOMP over WebSocket (rooms = topics, per-user queues) |
| Security | Spring Security, stateless JWT (resource server) |
| Image scanning | OpenCV (`org.openpnp:opencv`) + Tesseract via Tess4J |
| Database | PostgreSQL + Spring Data JPA + Flyway |
| Cache / rate limit | Bucket4j in-memory (Redis optional later) |
| Voice | WebRTC + coturn (STUN/TURN) |
| AI | `LlmClient` interface (LLM API for hint wording); speech-to-text service for voice |
| Storage | Local disk → S3-compatible |
| Observability | Actuator, Micrometer + Prometheus |
| Deployment | Docker (glibc base image with Tesseract), single instance behind Nginx/Caddy |

---

## 9. Roadmap (v2)

**Is the monolith a good choice for v1?** Yes. One team, one deployable, no network boundaries between modules, easy debugging and transactions, fastest path to a working product. Keep module boundaries strict and talk through interfaces/events so splitting later is mechanical.

**Signals that it is time to split**

- Concurrent rooms exceed what one instance can hold in memory / sockets
- OCR or AI jobs hurt real-time latency even with worker pools
- Teams want to deploy independently
- Need for analytics, replays, or anti-cheat pipelines on event streams

**v2 design:** [`docs/architecture-v2.mermaid`](docs/architecture-v2.mermaid)

| v1 | v2 |
|---|---|
| One server | Services: Identity, Puzzle, Solo Game, Room, Match, Scoring, Leaderboard, Chat, AI Assistant, Scan, Notification |
| In-process event bus (Spring application events) | **Kafka** for domain events (`match.*`, `score.*`, `room.*`, `chat.*`) |
| In-process OCR worker pool | **RabbitMQ / SQS** job queues + autoscaled OCR and STT workers |
| In-memory room state | **Redis** (presence, live room state, pub/sub) behind stateless realtime gateways |
| WebRTC P2P mesh | **SFU** (LiveKit / mediasoup) for stable voice |
| Single Postgres | Database per service; chat in Cassandra/Mongo; leaderboards in Redis ZSETs |
| Logs only | OpenTelemetry, Prometheus, Grafana, ClickHouse analytics |
| Scoring called inline | Scoring is an **event consumer** (`match.round.finished` → `score.awarded`) |

Migration path: extract the **Scan** worker first (heaviest, most isolated), then **Realtime Gateway**, then Scoring/Leaderboard via events.

---

## 10. Open Design Decisions

Things this README assumes — confirm or change:

1. **"Play with Computer"** is interpreted as *solo play against a generated puzzle*, not a bot opponent racing you. A bot (with a configurable solve speed) could be added later.
2. **Room name uniqueness:** private rooms can be joined by name alone, so names should be **unique among active rooms** (otherwise a name lookup is ambiguous).
3. **Accounts:** guests are allowed (display name only); scores/leaderboards for guests are not persisted globally.
4. **AI in competitive mode** is disabled. Alternative: allow it with a heavier hint penalty.
5. **Disconnects:** a player who drops mid-match has a 60 s grace period to reconnect; the host leaving transfers host to the longest-present member.
6. **Reconnection:** WebSocket clients resend `room:subscribe` and receive a full `room:state` snapshot.
