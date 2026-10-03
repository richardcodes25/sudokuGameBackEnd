# API & Class Reference (v1 Monolith)

Conventions: REST under `/api/v1`, JSON bodies, JWT bearer auth (guests get a short-lived JWT via `/auth/guest`). Real-time traffic uses a single WebSocket endpoint `/ws`.

---

## 1. REST APIs

### 1.1 Auth & User

| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/register` | Create account (email, password, display name) |
| POST | `/auth/login` | Returns access + refresh token |
| POST | `/auth/guest` | Play without account (display name only) |
| POST | `/auth/refresh` | Refresh access token |
| POST | `/auth/logout` | Revoke refresh token |
| GET | `/users/me` | Current profile (level, XP, avatar) |
| PATCH | `/users/me` | Update display name / avatar |
| GET | `/users/{userId}/stats` | Games played, win rate, best times, avg solve time per difficulty |

### 1.2 Scan & Solve (photo → grid)

| Method | Endpoint | Description |
|---|---|---|
| POST | `/scan` | Upload image (multipart). Returns `scanId`, 9×9 grid, per-cell confidence, list of low-confidence cells |
| GET | `/scan/{scanId}` | Fetch scan result (supports async processing) |
| PUT | `/scan/{scanId}/grid` | User corrects misread digits |
| POST | `/scan/{scanId}/solve` | Solve the confirmed grid → full solution (or `UNSOLVABLE` / `MULTIPLE_SOLUTIONS`) |
| POST | `/scan/{scanId}/hint` | Next logical step only (technique + target cell), not the whole solution |
| POST | `/solve` | Solve a grid sent directly as JSON (manual entry) |

### 1.3 Puzzle

| Method | Endpoint | Description |
|---|---|---|
| POST | `/puzzles/generate` | `{difficulty}` → new puzzle (unique solution guaranteed) |
| GET | `/puzzles/{puzzleId}` | Puzzle givens only (solution never exposed) |

### 1.4 Solo Game ("Play with Computer")

| Method | Endpoint | Description |
|---|---|---|
| POST | `/games` | Start game: `{difficulty, timerMode: STOPWATCH \| COUNTDOWN, durationSec?}` |
| GET | `/games/{gameId}` | Current state (board, elapsed/remaining, mistakes, hints) |
| PUT | `/games/{gameId}/cells` | Place / clear a digit `{row, col, value}`; server validates and counts mistakes |
| POST | `/games/{gameId}/notes` | Toggle pencil marks |
| POST | `/games/{gameId}/hint` | Use a hint (penalised in score) |
| POST | `/games/{gameId}/pause` · `/resume` | Pause / resume (stopwatch only, capped) |
| POST | `/games/{gameId}/submit` | Final check → result + score breakdown |
| POST | `/games/{gameId}/abandon` | Give up |
| GET | `/games/history` | Paginated past games |

### 1.5 Rooms

| Method | Endpoint | Description |
|---|---|---|
| POST | `/rooms` | Create room: `{name, visibility, mode, timerMode, durationSec?, difficulty, rounds, requireBothCredentials?}` |
| GET | `/rooms` | List **public** rooms (filter: hasSeat, mode, difficulty) |
| GET | `/rooms/{roomId}` | Room info: host, members, settings, status |
| POST | `/rooms/{roomId}/join` | Join a **public** room (first-come-first-in, 409 if full) |
| POST | `/rooms/join-requests` | Join a **private** room: `{roomId?, roomName?}` (both required if host enabled it) → creates a pending request |
| GET | `/rooms/{roomId}/join-requests` | (Host) pending requests |
| POST | `/rooms/{roomId}/join-requests/{requestId}/approve` | (Host) admit player |
| POST | `/rooms/{roomId}/join-requests/{requestId}/reject` | (Host) reject player |
| DELETE | `/rooms/{roomId}/join-requests/{requestId}` | Requester cancels |
| PATCH | `/rooms/{roomId}/settings` | (Host, lobby only) change mode/timer/difficulty/visibility/credential policy |
| POST | `/rooms/{roomId}/kick/{userId}` | (Host) remove member |
| POST | `/rooms/{roomId}/transfer-host/{userId}` | (Host) hand over host role |
| POST | `/rooms/{roomId}/leave` | Leave room (auto-transfer host if needed) |
| POST | `/rooms/{roomId}/start` | (Host) start match |
| DELETE | `/rooms/{roomId}` | (Host) close room |

### 1.6 Match, Chat, AI, Leaderboard

| Method | Endpoint | Description |
|---|---|---|
| GET | `/rooms/{roomId}/match` | Current match snapshot (board, timer, per-player progress) |
| GET | `/rooms/{roomId}/match/results` | Final results + score breakdown |
| GET | `/rooms/{roomId}/messages` | Chat history (cursor pagination) |
| POST | `/rooms/{roomId}/ai/hint` | Ask the AI assistant for a hint (also reachable by chat `@AI`) |
| POST | `/rooms/{roomId}/ai/voice` | Upload short voice clip ("Ask AI" push-to-talk) → transcribed → handled like a chat question |
| GET | `/rooms/{roomId}/leaderboard` | In-room standings |
| GET | `/leaderboards` | Global boards: `scope=weekly\|alltime`, `metric=score\|avgTime`, `difficulty` |
| GET | `/rtc/ice-servers` | STUN/TURN credentials for WebRTC |

---

## 2. WebSocket Events (`/ws`)

Transport is **STOMP over WebSocket** at `/ws`. The client sends the JWT in the STOMP `CONNECT` frame (`Authorization: Bearer …`), then subscribes. Every server message is an envelope `{type, payload, seq}`, where `type` is one of the event names below.

| Purpose | Destination |
|---|---|
| Room broadcasts (server → members) | `SUBSCRIBE /topic/room.{roomId}` |
| Private events (join results, host notifications, errors) | `SUBSCRIBE /user/queue/events`, `/user/queue/errors` |
| WebRTC signalling (private) | `SUBSCRIBE /user/queue/rtc` · `SEND /app/rtc/signal` |
| Full room snapshot (`room:state`) | `SUBSCRIBE /app/room/{roomId}/state` |
| Client actions | `SEND /app/room/{roomId}/cell` · `/cursor` · `/chat` · `/ai` · `/ready` |

The client-to-server names listed below (`cell:place`, `chat:send`, …) correspond to those `/app/room/{roomId}/…` destinations.

**Server → Client**

| Event | Payload / Purpose |
|---|---|
| `room:state` | Full room snapshot (on subscribe / reconnect) |
| `member:joined` · `member:left` · `member:kicked` | Roster changes |
| `join:requested` | (Host only) new pending request |
| `join:approved` · `join:rejected` | Sent to requester |
| `settings:updated` | Lobby settings changed |
| `match:countdown` · `match:started` | Match begins (puzzle givens included) |
| `timer:sync` | Server clock tick (every few seconds; client interpolates) |
| `cell:updated` | (Collab) someone placed a digit |
| `cursor:moved` | (Collab) selected-cell highlighting |
| `progress:updated` | (Compete) % filled of each opponent — never the digits |
| `player:finished` | (Compete) someone completed |
| `round:finished` · `match:finished` | Scores + standings |
| `chat:message` | Human message |
| `ai:message` | AI assistant reply / hint |
| `rtc:offer` · `rtc:answer` · `rtc:ice` | WebRTC signalling relay |

**Client → Server**

`room:subscribe`, `room:unsubscribe`, `cell:place`, `cell:clear`, `cursor:move`, `chat:send`, `ai:ask`, `ready:set`, `rtc:offer`, `rtc:answer`, `rtc:ice`, `ping`.

---

## 3. Classes

### 3.1 Domain Entities

| Class | Key fields |
|---|---|
| `User` | id, displayName, email?, isGuest, xp, level, createdAt |
| `Puzzle` | id, givens[81], solution[81], difficulty, clueCount |
| `Grid` / `Cell` | row, col, value, isGiven, notes |
| `ScanJob` | id, userId, imagePath, status, grid, confidence[81], createdAt |
| `SoloGame` | id, userId, puzzleId, timerMode, durationSec, startedAt, pausedTotal, mistakes, hintsUsed, status |
| `Room` | id (auto), name, hostId, visibility, mode, timerMode, durationSec, difficulty, rounds, requireBothCredentials, status, maxSeats = 5 |
| `RoomMember` | roomId, userId, role (HOST/PLAYER), joinedAt, isReady, isConnected |
| `JoinRequest` | id, roomId, userId, status (PENDING/APPROVED/REJECTED/CANCELLED), createdAt |
| `Match` | id, roomId, mode, status, currentRound, startedAt |
| `MatchRound` | id, matchId, puzzleId, startedAt, endedAt |
| `PlayerRoundResult` | roundId, userId, solveTimeSec, mistakes, hints, finished, score, placement |
| `TeamRoundResult` | roundId, teamTimeSec, mistakes, hints, score, contributions{userId: cells} |
| `ChatMessage` | id, roomId, senderId / `AI`, type (TEXT/AI_HINT/SYSTEM), content, createdAt |
| `ScoreRecord` | id, userId, source (SOLO/COMPETE/COLLAB), points, breakdown(JSON), createdAt |
| `HintRecord` | id, ownerId, level, technique, targetCell, createdAt |

### 3.2 Enums

`Difficulty {EASY, MEDIUM, HARD, EXPERT}` · `TimerMode {STOPWATCH, COUNTDOWN}` · `RoomVisibility {PUBLIC, PRIVATE}` · `RoomMode {COLLAB, COMPETITIVE}` · `RoomStatus {LOBBY, IN_PROGRESS, FINISHED, CLOSED}` · `MemberRole {HOST, PLAYER}` · `JoinRequestStatus` · `MatchStatus {WAITING, COUNTDOWN, RUNNING, FINISHED}`

### 3.3 Controllers / Gateways

`AuthController`, `UserController`, `ScanController`, `PuzzleController`, `SoloGameController`, `RoomController`, `MatchController`, `ChatController`, `AIController`, `LeaderboardController`, `RtcController`, `RoomGateway` (WebSocket), `SignalingGateway`.

### 3.4 Services

| Service | Responsibility |
|---|---|
| `AuthService` | Register/login/guest, JWT issue & refresh, password hashing |
| `UserService` | Profiles, XP/level updates, stats aggregation |
| `ImageScanService` | Orchestrates scan pipeline, stores image, returns grid + confidence |
| `ImagePreprocessor` | OpenCV: grayscale, blur, adaptive threshold, deskew |
| `GridDetector` | OpenCV: largest quadrilateral contour, perspective warp, 9×9 split |
| `DigitRecognizer` | Tesseract on each cell (`--psm 10`, whitelist 1-9), blank-cell detection, confidence |
| `SudokuGenerator` | Generates puzzles with unique solution per difficulty |
| `SudokuSolver` | Backtracking + constraint propagation; counts solutions (max 2) |
| `LogicalSolver` | Human-technique solver (naked/hidden singles, pairs, X-Wing…) used for hints & difficulty rating |
| `SudokuValidator` | Row/col/box conflicts, completeness, compare to solution |
| `DifficultyRater` | Rates puzzle by techniques needed |
| `SoloGameService` | Game lifecycle, move validation, mistake/hint counting |
| `TimerService` | Server-authoritative stopwatch/countdown, pause rules, expiry callbacks |
| `RoomService` | Create/list/get/update/close rooms, seat limits, host transfer |
| `JoinRequestService` | Private-room join flow, credential check (id / name / both), host approve/reject |
| `RoomRegistry` | In-memory live state of active rooms + WebSocket sessions |
| `MatchService` | Chooses strategy (`CollabMatchStrategy` / `CompetitiveMatchStrategy`), runs rounds |
| `CollabMatchStrategy` | Shared board, conflict resolution (last-write-wins per cell + version), team result |
| `CompetitiveMatchStrategy` | Independent boards, same puzzle, per-player timers, placements |
| `ScoringService` | Implements the scoring formulas (see README) |
| `LeaderboardService` | Room standings + global boards |
| `ChatService` | Persist & broadcast messages, detect `@AI` mentions, rate-limit |
| `AIAssistantService` | Entry point for AI questions in a room |
| `HintEngine` | Uses `LogicalSolver` to pick a hint (levels 1-3), enforces hint budget, **never** returns raw solution |
| `LLMClient` | Turns a structured hint into a friendly reply; output filtered by `SolutionLeakGuard` |
| `SolutionLeakGuard` | Rejects AI text that states digits for empty cells beyond the allowed hint level |
| `SpeechToTextClient` | Converts voice clip to text for the AI |
| `SignalingService` | Relays WebRTC offers/answers/ICE between members |
| `IceServerProvider` | Issues short-lived TURN credentials |
| `EventBus` | In-process pub/sub (`MatchFinished`, `ScoreAwarded`, …) — becomes Kafka in v2 |

### 3.4 Repositories & Infrastructure

`UserRepository`, `PuzzleRepository`, `ScanJobRepository`, `SoloGameRepository`, `RoomRepository`, `JoinRequestRepository`, `MatchRepository`, `ChatRepository`, `ScoreRepository`, `ImageStorage` (local/S3), `CacheClient` (Redis, optional), `OcrWorkerPool`, `Scheduler`, `RateLimiter`.

### 3.5 Core Domain Diagram

```mermaid
classDiagram
    class User { id; displayName; xp; level }
    class Room { id; name; visibility; mode; timerMode; maxSeats=5 }
    class RoomMember { role; isReady }
    class JoinRequest { status }
    class Match { status; currentRound }
    class MatchRound { startedAt; endedAt }
    class Puzzle { givens; solution; difficulty }
    class PlayerRoundResult { solveTimeSec; mistakes; hints; score; placement }
    class TeamRoundResult { teamTimeSec; mistakes; hints; score }
    class ChatMessage { sender; type; content }

    User "1" --> "*" RoomMember
    Room "1" --> "1..5" RoomMember
    Room "1" --> "*" JoinRequest
    User "1" --> "*" JoinRequest
    Room "1" --> "0..1" Match
    Match "1" --> "1..5" MatchRound
    MatchRound --> Puzzle
    MatchRound "1" --> "*" PlayerRoundResult
    MatchRound "1" --> "0..1" TeamRoundResult
    Room "1" --> "*" ChatMessage
