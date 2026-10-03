# Implementation Guide — v1 (Spring Boot Modular Monolith)

This guide walks you through building v1 with **Java + Spring Boot**, in the order that minimises risk: **pure logic first, real-time last**. Every phase ends with a "Done when" checklist so you always have a working, testable system.

---

## 0. Technology Decisions

| Concern | Choice | Why |
|---|---|---|
| Language | **Java 25 LTS** (21 LTS also works) | Records, sealed types, pattern matching, virtual threads |
| Framework | **Spring Boot 4.0.x / 4.1.x** (Spring Framework 7) | Boot 3.5 left OSS support in June 2026, so start on 4.x |
| Build | Maven (Gradle works the same) | |
| Module boundaries | **Spring Modulith 2.0** | Verifies module boundaries in a test, gives you application events and an event-publication registry. This is the *modular monolith* tool |
| Web | Spring MVC (`spring-boot-starter-webmvc`) + **virtual threads** | Simple blocking code scales fine for this workload |
| Real-time | **STOMP over WebSocket** (`spring-boot-starter-websocket`) | Rooms map naturally to topics, per-user queues, built-in `@MessageMapping` |
| Security | Spring Security 7, stateless JWT (resource server + Nimbus) | Little custom code |
| Persistence | Spring Data JPA (Hibernate 7) + PostgreSQL + **Flyway** | |
| Image scanning | **OpenCV** via `org.openpnp:opencv` (bundles native libs) + **Tess4J** (Tesseract wrapper) | |
| Rate limiting | Bucket4j (in-memory) | Redis-backed later if needed |
| LLM | `LlmClient` interface; implement with the Anthropic Java SDK, Spring AI, or plain `RestClient` | Keeps the vendor swappable |
| Tests | JUnit 5, AssertJ, Testcontainers 2.x, Awaitility, ArchUnit | |
| Docs / ops | springdoc-openapi, Actuator + Micrometer/Prometheus | |

> **Boot 4 heads-up.** Several starters were renamed (e.g. `spring-boot-starter-web` → `spring-boot-starter-webmvc`, OAuth2 resource server → `spring-boot-starter-security-oauth2-resource-server`), Jackson moved to 3.x (`tools.jackson.*`), and `@MockBean` became `@MockitoBean`. **Generate the project at start.spring.io** so the starters and BOM versions are correct, and use the official Spring Boot 4.0 migration guide if you copy older examples from the internet. If you must stay on 3.x, the code in this guide still applies with the old starter names.

---

## Roadmap at a glance

| Phase | What you build | Depends on | Rough effort* |
|---:|---|---|---|
| 0 | Project setup, Modulith structure, core config | — | 2–3 days |
| 1 | Sudoku engine (solver, validator, generator, logical solver) | 0 | 1–1.5 weeks |
| 2 | Database (Flyway), auth, users | 0 | 4–5 days |
| 3 | Scoring service (pure) | 1 | 1–2 days |
| 4 | Puzzle pool + Solo game (timer, moves, submit) | 1, 2, 3 | 1 week |
| 5 | Scan & Solve (OpenCV + Tess4J) | 1 | 1–1.5 weeks |
| 6 | Rooms REST (create, list, join, private approval) | 2 | 1 week |
| 7 | Real-time layer (STOMP, presence, events) | 6 | 4–5 days |
| 8 | Match module: Competitive, then Collab | 1, 3, 7 | 1.5–2 weeks |
| 9 | Chat | 7 | 2–3 days |
| 10 | AI assistant (HintEngine + LLM + leak guard) | 1, 8, 9 | 1 week |
| 11 | Voice (WebRTC signalling + TURN) | 7 | 3–4 days |
| 12 | Leaderboards, XP, profiles | 3, 8 | 3–4 days |
| 13 | Hardening: rate limits, anti-cheat, reconnects | all | 1 week |
| 14 | Testing, CI/CD, deployment | all | 3–5 days |

\*Solo developer, excluding the frontend. Phases 4 and 5 are independent of 6, so two people can parallelise.

**Milestones:** M1 (after 4) solo game via API · M2 (after 5) scan a photo and get a solution · M3 (after 8) multiplayer matches · M4 (after 12) full feature set · M5 (after 14) production.

---

## Phase 0 — Project Setup

### Step 0.1 Generate the project

At **start.spring.io**: Maven, Java 25 (or 21), latest Boot 4.x. Add: Spring Web (MVC), Validation, Spring Data JPA, PostgreSQL Driver, Flyway, Spring Security, OAuth2 Resource Server, WebSocket, Actuator, Spring Modulith, Testcontainers, Prometheus.

Then add manually:

```xml
<dependency>
  <groupId>org.openpnp</groupId><artifactId>opencv</artifactId><version><!-- latest 4.x --></version>
</dependency>
<dependency>
  <groupId>net.sourceforge.tess4j</groupId><artifactId>tess4j</artifactId><version><!-- latest 5.x --></version>
</dependency>
<dependency>
  <groupId>com.bucket4j</groupId><artifactId>bucket4j_jdk17-core</artifactId><version><!-- latest 8.x --></version>
</dependency>
<dependency>
  <groupId>org.springdoc</groupId><artifactId>springdoc-openapi-starter-webmvc-ui</artifactId><version><!-- latest --></version>
</dependency>
<!-- test -->
<dependency>
  <groupId>com.tngtech.archunit</groupId><artifactId>archunit-junit5</artifactId><scope>test</scope>
</dependency>
```

System dependency for OCR (also in the Docker image): `sudo apt install tesseract-ocr` (Tess4J 5.x expects Tesseract 5.x).

### Step 0.2 Package structure = Modulith modules

Every **top-level package under the application package is a module**. Anything in the module's base package is its *public API*; sub-packages (e.g. `internal`) are hidden from other modules.

```
com.example.sudoku
├─ SudokuApplication.java
├─ shared/        # OPEN module: errors, Clock bean, config props, base DTOs
├─ engine/        # PURE JAVA: Difficulty, SudokuSolver, Validator, Generator, LogicalSolver, Rater
├─ identity/      # auth + users          (public: IdentityService, UserView)
│  └─ internal/   # entities, repos, controllers, security config
├─ puzzles/       # puzzle pool           (public: PuzzleService, PuzzleView)
├─ solo/          # solo games
├─ scan/          # image scan & solve
├─ scoring/       # ScoringService, leaderboard, score_records (public: ScoringService, ScoreAwarded)
├─ rooms/         # rooms, members, join requests (public: RoomService, events)
├─ match/         # match lifecycle, strategies, live room registry
├─ chat/
├─ ai/            # HintEngine, LlmClient, leak guard
├─ rtc/           # WebRTC signalling + ICE servers
└─ realtime/      # STOMP config, auth interceptor, presence, event→WebSocket broadcaster
```

**Rules (enforced by tests, see Step 0.4):**

1. Another module may use only the **public types in a module's base package** — never `internal`.
2. `engine` has **no dependencies** on other modules and no Spring/JPA imports.
3. Cross-module reactions use **application events** (e.g. `ScoreAwarded` → identity bumps XP), not direct calls, wherever possible.
4. No cycles. Example: `POST /rooms/{id}/start` lives in `match` (it calls `rooms`), not in `rooms`.

Declare dependencies explicitly where helpful:

```java
// match/package-info.java
@org.springframework.modulith.ApplicationModule(
    allowedDependencies = {"engine", "rooms", "scoring", "shared"})
package com.example.sudoku.match;
```

### Step 0.3 Core configuration

`application.yml`:

```yaml
spring:
  threads.virtual.enabled: true            # virtual threads for request handling
  servlet.multipart.max-file-size: 10MB
  jpa.hibernate.ddl-auto: validate         # Flyway owns the schema
  datasource.url: ${DB_URL}
sudoku:
  jwt:
    secret: ${JWT_SECRET}                  # >= 32 bytes
    access-ttl: 15m
    refresh-ttl: 30d
  scoring:
    tiers:
      EASY:   {base: 100, par-seconds: 360,  default-countdown-seconds: 900}
      MEDIUM: {base: 200, par-seconds: 720,  default-countdown-seconds: 1500}
      HARD:   {base: 350, par-seconds: 1200, default-countdown-seconds: 2400}
      EXPERT: {base: 500, par-seconds: 1800, default-countdown-seconds: 3600}
management.endpoints.web.exposure.include: health,info,prometheus
```

Foundation beans in `shared`:

```java
@Configuration
@EnableScheduling
@EnableAsync
class SharedConfig {
    @Bean Clock clock() { return Clock.systemUTC(); }        // inject everywhere, fake it in tests

    @Bean(name = "ocrExecutor")                              // CPU-bound: bounded platform threads
    ThreadPoolTaskExecutor ocrExecutor() {
        var ex = new ThreadPoolTaskExecutor();
        int n = Runtime.getRuntime().availableProcessors();
        ex.setCorePoolSize(n); ex.setMaxPoolSize(n); ex.setQueueCapacity(20);
        ex.setThreadNamePrefix("ocr-");
        ex.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy()); // -> HTTP 429
        return ex;
    }
}
```

> **Virtual threads vs CPU work:** virtual threads are for waiting (DB, HTTP, WebSocket). OCR and puzzle generation are CPU-bound, so give them a **bounded executor** like above.

Error handling — one `AppException(code, status, message)` plus a `@RestControllerAdvice` returning `ProblemDetail` (RFC 7807):

```java
@RestControllerAdvice
class ApiExceptionHandler {
    @ExceptionHandler(AppException.class)
    ProblemDetail handle(AppException e) {
        var pd = ProblemDetail.forStatusAndDetail(e.status(), e.getMessage());
        pd.setProperty("code", e.code());
        return pd;
    }
}
```

Add a servlet filter that puts a request ID into the MDC and enable structured logging (`logging.structured.format.console: ecs`).

### Step 0.4 Guard-rail tests (write on day 1)

```java
class ModularityTests {
    @Test void modulesAreRespected() {
        ApplicationModules.of(SudokuApplication.class).verify();   // fails on cycles / internal access
    }
}

@AnalyzeClasses(packages = "com.example.sudoku")
class EngineRules {
    @ArchTest static final ArchRule enginePure = noClasses().that().resideInAPackage("..engine..")
        .should().dependOnClassesThat().resideInAnyPackage("org.springframework..", "jakarta.persistence..");
}
```

### Step 0.5 docker-compose

Services: `app`, `postgres` (add `coturn` in Phase 11). `GET /actuator/health` must be green.

**Done when:** `docker compose up` serves the health endpoint; `mvn verify` runs the modularity tests; CI (build + test) is green.

---

## Phase 1 — Sudoku Engine (plain Java, no Spring)

Represent a grid as `int[81]` (`0` = empty). Write this entire phase test-first with JUnit/AssertJ.

### Step 1.1 Validator

`isValidPlacement(grid, idx, value)`, `findConflicts(grid)` (indexes in conflict — used by scan), `isComplete`, `matchesSolution`.

### Step 1.2 Solver (bitmask backtracking + MRV)

Counts up to `limit` solutions, so you can detect *unsolvable* (0) and *multiple solutions* (≥ 2).

```java
public final class SudokuSolver {
    private final int[] g;
    private final int[] rows = new int[9], cols = new int[9], boxes = new int[9];
    private final List<int[]> found = new ArrayList<>();
    private final int limit;

    private SudokuSolver(int[] grid, int limit) { this.g = grid.clone(); this.limit = limit; }

    public static List<int[]> solve(int[] grid, int limit) {
        var s = new SudokuSolver(grid, limit);
        if (!s.init()) return List.of();           // given digits already conflict
        s.search();
        return s.found;
    }

    private boolean init() {
        for (int i = 0; i < 81; i++) {
            int v = g[i];
            if (v == 0) continue;
            int b = 1 << v, r = i / 9, c = i % 9, bx = (r / 3) * 3 + c / 3;
            if (((rows[r] | cols[c] | boxes[bx]) & b) != 0) return false;
            rows[r] |= b; cols[c] |= b; boxes[bx] |= b;
        }
        return true;
    }

    private void search() {
        if (found.size() >= limit) return;
        int best = -1, bestMask = 0, bestCnt = 10;
        for (int i = 0; i < 81; i++) {
            if (g[i] != 0) continue;
            int r = i / 9, c = i % 9;
            int mask = ~(rows[r] | cols[c] | boxes[(r / 3) * 3 + c / 3]) & 0x3FE;
            int cnt = Integer.bitCount(mask);
            if (cnt < bestCnt) { best = i; bestMask = mask; bestCnt = cnt; if (cnt <= 1) break; }
        }
        if (best == -1) { found.add(g.clone()); return; }
        if (bestCnt == 0) return;
        int r = best / 9, c = best % 9, bx = (r / 3) * 3 + c / 3;
        for (int v = 1; v <= 9 && found.size() < limit; v++) {
            int b = 1 << v;
            if ((bestMask & b) == 0) continue;
            g[best] = v; rows[r] |= b; cols[c] |= b; boxes[bx] |= b;
            search();
            rows[r] &= ~b; cols[c] &= ~b; boxes[bx] &= ~b; g[best] = 0;
        }
    }
}
```

Tests: easy/hard known puzzles, invalid givens, empty grid, a puzzle with exactly two solutions. Add a **jqwik property test**: for any generated puzzle the solver finds exactly one solution.

### Step 1.3 Generator

1. Build a full grid: fill the three diagonal boxes with shuffled digits, then `solve(grid, 1)` the rest.
2. Remove cells in random order; after each removal run `solve(limit = 2)` and **keep the removal only if exactly one solution remains**.
3. Stop at the target clue count.

| Difficulty | Target clues (guideline) | Techniques needed (LogicalSolver) |
|---|---|---|
| Easy | 38–45 | Singles only |
| Medium | 32–37 | + pairs, pointing/claiming |
| Hard | 27–31 | + triples, X-Wing |
| Expert | 22–26 | Beyond the above |

Generating Hard/Expert can take hundreds of ms — **never generate on the request path**. Phase 4 builds a pre-generated pool.

### Step 1.4 Logical solver (needed for difficulty rating and AI hints)

Keep a candidate bitmask per cell. Each technique returns a `Step`:

```java
public record Step(Technique technique, int cellIndex, int value, String unit) {}   // unit: "row 4", "box 5"
public enum Technique { NAKED_SINGLE, HIDDEN_SINGLE, NAKED_PAIR, HIDDEN_PAIR, POINTING_PAIR, X_WING }
```

Implement in order: naked single → hidden single → pairs → pointing/claiming → X-Wing. You can ship v1 with singles + pairs; puzzles that need more are rated Expert.

### Step 1.5 Difficulty rater

`rate(puzzle)` = hardest technique required → `Difficulty`. Use it to assert generator output.

```java
public enum Difficulty { EASY, MEDIUM, HARD, EXPERT }
```

**Done when:** ≥ 95% coverage on `engine`; 100 generated puzzles per difficulty each have a unique solution and the correct rating; the ArchUnit "engine is pure" test passes.

---

## Phase 2 — Database, Auth, Users

### Step 2.1 Flyway migrations

`src/main/resources/db/migration/V1__init.sql` — tables:

| Table | Key columns |
|---|---|
| `users` | id uuid, display_name, email (unique, nullable), password_hash (nullable), is_guest, xp, level, created_at |
| `refresh_tokens` | id, user_id, token_hash, expires_at, revoked_at |
| `puzzles` | id, givens char(81), solution char(81), difficulty, clue_count, created_at |
| `solo_games` | id, user_id, puzzle_id, timer_mode, duration_sec, started_at, paused_at, paused_total_sec, board char(81), mistakes, hints_used, status, finished_at, score |
| `scan_jobs` | id, user_id, image_path, status, grid, confidence jsonb, created_at |
| `rooms` | id char(6), name, host_id, visibility, mode, timer_mode, duration_sec, difficulty, rounds, require_both_credentials, status, created_at |
| `room_members` | room_id, user_id, role, joined_at — PK(room_id, user_id) |
| `join_requests` | id, room_id, user_id, status, created_at, decided_at |
| `matches`, `match_rounds` | ids, room_id, mode, status, puzzle_id, started_at, ended_at |
| `round_results` | round_id, user_id (null = team), solve_time_sec, mistakes, hints, finished, score, placement, contributions jsonb |
| `chat_messages` | id, room_id, sender_id (null = AI), type, content, created_at |
| `score_records` | id, user_id, source, points, breakdown jsonb, created_at |
| `event_publication` | **Spring Modulith event registry** — copy the schema from the Modulith reference docs |

Important indexes:

```sql
CREATE UNIQUE INDEX ux_rooms_active_name ON rooms (lower(name)) WHERE status IN ('LOBBY','IN_PROGRESS');
CREATE INDEX ix_rooms_public ON rooms (visibility, status);
CREATE INDEX ix_chat_room_time ON chat_messages (room_id, created_at);
CREATE INDEX ix_scores_user_time ON score_records (user_id, created_at);
```

JSON columns in entities: `@JdbcTypeCode(SqlTypes.JSON) private Map<String,Object> breakdown;`

### Step 2.2 Identity module — security

Stateless JWT using the built-in resource-server support (no custom filter):

```java
@Configuration @EnableWebSecurity @EnableMethodSecurity
class SecurityConfig {

    @Bean SecurityFilterChain chain(HttpSecurity http) throws Exception {
        return http
            .csrf(c -> c.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(a -> a
                .requestMatchers("/api/v1/auth/**", "/actuator/health", "/ws/**",
                                 "/v3/api-docs/**", "/swagger-ui/**").permitAll()
                .anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
            .build();
    }

    @Bean SecretKey jwtKey(@Value("${sudoku.jwt.secret}") String secret) {
        return new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256");
    }
    @Bean JwtDecoder jwtDecoder(SecretKey key) {
        return NimbusJwtDecoder.withSecretKey(key).macAlgorithm(MacAlgorithm.HS256).build();
    }
    @Bean JwtEncoder jwtEncoder(SecretKey key) {
        return new NimbusJwtEncoder(new ImmutableSecret<>(key));
    }
    @Bean PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }
}
```

`/ws/**` is permitted at the HTTP level because browsers cannot add headers to a WebSocket handshake; the **STOMP CONNECT frame is authenticated** instead (Phase 7).

Issue tokens:

```java
String issueAccessToken(User u) {
    var now = Instant.now(clock);
    var claims = JwtClaimsSet.builder().subject(u.getId().toString())
        .issuedAt(now).expiresAt(now.plus(props.accessTtl()))
        .claim("guest", u.isGuest()).build();
    var header = JwsHeader.with(MacAlgorithm.HS256).build();
    return encoder.encode(JwtEncoderParameters.from(header, claims)).getTokenValue();
}
```

Controllers read the current user with `@AuthenticationPrincipal Jwt jwt` → `UUID.fromString(jwt.getSubject())`.

### Step 2.3 Endpoints

`POST /api/v1/auth/register | login | guest | refresh | logout`, `GET/PATCH /api/v1/users/me`, `GET /api/v1/users/{id}/stats` (stub for now).

- Refresh tokens: random 256-bit value, **stored as SHA-256 hash**, rotated on every use, revocable.
- Guests: row with `is_guest = true`, longer-lived access token, upgradeable to a real account.
- Validate request DTOs (records) with Jakarta annotations: `record RegisterRequest(@Email String email, @Size(min = 8) String password, @NotBlank @Size(max = 30) String displayName) {}`

**Done when:** register → login → protected call works; refresh rotation and logout are tested (`@SpringBootTest` + Testcontainers Postgres); guest flow works.

---

## Phase 3 — Scoring Service (pure)

Implement before the game modes so they can call it. Keep the calculation free of I/O and fully driven by configuration.

```java
@ConfigurationProperties("sudoku.scoring")
public record ScoringProperties(Map<Difficulty, Tier> tiers) {
    public record Tier(int base, int parSeconds, int defaultCountdownSeconds) {}
}

@Service
public class ScoringService {
    private final ScoringProperties props;
    public ScoringService(ScoringProperties props) { this.props = props; }

    public int puzzleScore(Difficulty d, double solveSec, int mistakes, int hints, Double parOverride) {
        var t = props.tiers().get(d);
        double par = parOverride != null ? parOverride : t.parSeconds();
        double timeM = Math.min(1.5, Math.max(0.5, 1.5 - 0.5 * solveSec / par));
        double accM  = Math.max(0.5, 1 - 0.10 * mistakes);
        double hintM = Math.max(0.6, 1 - 0.05 * hints);
        double score = t.base() * timeM * accM * hintM;
        if (mistakes == 0 && hints == 0) score += 0.10 * t.base();
        return (int) Math.max(Math.round(score), Math.round(0.20 * t.base()));
    }

    public int partialScore(Difficulty d, int correctFilled, int emptyAtStart) {
        return (int) Math.round(0.30 * props.tiers().get(d).base()
                                 * correctFilled / Math.max(1, emptyAtStart));
    }

    public int placementBonus(Difficulty d, int place) {                 // place is 1-based
        double pct = switch (place) { case 1 -> 0.25; case 2 -> 0.15; case 3 -> 0.10; case 4 -> 0.05; default -> 0; };
        return (int) Math.round(props.tiers().get(d).base() * pct);
    }

    public double teamPar(Difficulty d, int players) {
        return props.tiers().get(d).parSeconds() * (1 - 0.10 * (players - 1));
    }

    public List<PlayerTotals> rank(List<PlayerTotals> players) {
        return players.stream().sorted(
            Comparator.comparingInt(PlayerTotals::totalScore).reversed()
                .thenComparingDouble(PlayerTotals::avgSolveSec)
                .thenComparingInt(PlayerTotals::totalMistakes)
                .thenComparingDouble(PlayerTotals::lastRoundFinishSec)).toList();
    }

    public int xpFor(int score, boolean winner) { return (int) Math.round(score / 10.0 * (winner ? 1.2 : 1.0)); }
}
```

Also in this module (added in Phase 12): `award(...)` persists a `score_records` row and publishes the event other modules react to:

```java
public record ScoreAwarded(UUID userId, ScoreSource source, int points, int xp) {}
```

**Tests (table-driven):** the README worked examples (Medium 8 min/1 mistake → **210**, Hard 25 min/3 mistakes/2 hints → **193**, Easy 3 min perfect → **135**), `t = 0`, `t ≫ par`, many mistakes, the 20% floor, ranking tie-breakers. Instantiate with `new ScoringService(props)` — no Spring context needed.

**Done when:** every formula is covered by unit tests, and changing a number in `application.yml` changes behaviour without code edits.

---

## Phase 4 — Puzzle Pool and Solo Game

### Step 4.1 Puzzles module

- `PuzzleService.takeRandom(Difficulty)` — returns an unused puzzle from the pool
- `PuzzleRefillJob` keeps a minimum stock per difficulty:

```java
@Component
class PuzzleRefillJob {
    @Scheduled(fixedDelayString = "PT30S")
    void refill() {
        for (Difficulty d : Difficulty.values())
            while (repo.countByDifficultyAndUsedFalse(d) < MIN_STOCK)
                repo.save(PuzzleEntity.from(generator.generate(d)));
    }
}
```

(Run the generator on the bounded executor if refilling takes long; also run one refill at startup.)

- `POST /api/v1/puzzles/generate`, `GET /api/v1/puzzles/{id}` — **never expose `solution`** (use a `PuzzleView` record without it).

### Step 4.2 Timer design

No per-game timers. Store timestamps and compute; inject `Clock` so tests can control time:

```java
Duration elapsed(SoloGame g, Instant now) {
    Duration paused = g.getPausedTotal();
    if (g.getPausedAt() != null) paused = paused.plus(Duration.between(g.getPausedAt(), now));
    return Duration.between(g.getStartedAt(), now).minus(paused);
}
Duration remaining(SoloGame g, Instant now) {                // COUNTDOWN
    return Duration.ofSeconds(g.getDurationSec()).minus(elapsed(g, now));
}
```

Expiry is checked **lazily** on every move / submit / get (`remaining <= 0` → status `FAILED`). A `@Scheduled` sweeper closes abandoned games.

### Step 4.3 Solo game service

| Endpoint | Server logic |
|---|---|
| `POST /games` | Take puzzle, create row (`board = givens`, `started_at = now`) |
| `PUT /games/{id}/cells` | Must be `RUNNING`, check expiry, cell not a given; compare with solution → wrong digit ⇒ `mistakes++` (dedupe by cell+value); persist and return state |
| `POST /games/{id}/hint` | Reveal one correct cell using the LogicalSolver, `hintsUsed++` |
| `POST /games/{id}/pause` / `resume` | Stopwatch only, total pause capped (e.g. 10 min) |
| `POST /games/{id}/submit` | Verify board == solution → `scoring.puzzleScore(...)` → `scoring.award(...)` → return breakdown |
| `GET /games/history` | Paginated (`Pageable`) |

Every response includes server-computed `elapsedSec` / `remainingSec`. Check ownership on every call (`game.userId == jwt.sub`) — consider `@PostAuthorize` or a service-level assertion.

**Done when (M1):** you can start a game, place cells, use a hint, submit and see the right score from Swagger UI; countdown expiry fails the game; the solution is never in any response.

---

## Phase 5 — Scan & Solve

### Step 5.1 OpenCV + Tess4J setup

```java
@Component
class OpenCvLoader {
    @PostConstruct void load() { nu.pattern.OpenCV.loadLocally(); }   // org.openpnp: bundles natives
}
```

> OpenCV `Mat` objects hold native memory. **Release every `Mat`** (`mat.release()` in `finally`) or you will leak memory under load.

### Step 5.2 Pipeline steps (small, testable classes)

**GridDetector**

```java
Mat extractGrid(Mat bgr) {
    final int SIZE = 450;
    Mat gray = new Mat(), blur = new Mat(), th = new Mat();
    Imgproc.cvtColor(bgr, gray, Imgproc.COLOR_BGR2GRAY);
    Imgproc.GaussianBlur(gray, blur, new Size(7, 7), 0);
    Imgproc.adaptiveThreshold(blur, th, 255, Imgproc.ADAPTIVE_THRESH_GAUSSIAN_C,
                              Imgproc.THRESH_BINARY_INV, 11, 2);

    List<MatOfPoint> contours = new ArrayList<>();
    Imgproc.findContours(th, contours, new Mat(), Imgproc.RETR_EXTERNAL, Imgproc.CHAIN_APPROX_SIMPLE);
    MatOfPoint biggest = contours.stream()
        .max(Comparator.comparingDouble(c -> Imgproc.contourArea(c)))
        .orElseThrow(GridNotFoundException::new);

    MatOfPoint2f c2f = new MatOfPoint2f(biggest.toArray()), approx = new MatOfPoint2f();
    Imgproc.approxPolyDP(c2f, approx, 0.02 * Imgproc.arcLength(c2f, true), true);
    if (approx.total() != 4) throw new GridNotFoundException();

    Point[] ordered = orderPoints(approx.toArray());               // TL, TR, BR, BL
    MatOfPoint2f src = new MatOfPoint2f(ordered);
    MatOfPoint2f dst = new MatOfPoint2f(new Point(0, 0), new Point(SIZE - 1, 0),
                                        new Point(SIZE - 1, SIZE - 1), new Point(0, SIZE - 1));
    Mat transform = Imgproc.getPerspectiveTransform(src, dst);
    Mat out = new Mat();
    Imgproc.warpPerspective(gray, out, transform, new Size(SIZE, SIZE));
    return out;                                                    // release the temporaries!
}

static Point[] orderPoints(Point[] p) {
    Point tl = Arrays.stream(p).min(Comparator.comparingDouble(q -> q.x + q.y)).get();
    Point br = Arrays.stream(p).max(Comparator.comparingDouble(q -> q.x + q.y)).get();
    Point tr = Arrays.stream(p).min(Comparator.comparingDouble(q -> q.y - q.x)).get();
    Point bl = Arrays.stream(p).max(Comparator.comparingDouble(q -> q.y - q.x)).get();
    return new Point[]{tl, tr, br, bl};
}
```

**DigitRecognizer** — per cell:

1. Trim ~12% border to remove grid lines, Otsu threshold.
2. **Blank test:** look at the centre half of the cell; if `Core.countNonZero(centre) < 0.03 * centre.total()` → empty cell (skip OCR, confidence 1.0).
3. Convert the `Mat` to `BufferedImage` (`Imgcodecs.imencode(".png", …)` → `ImageIO.read`), pad with a white border.
4. OCR in single-character mode:

```java
Tesseract t = new Tesseract();
t.setDatapath(tessdataPath);                    // e.g. /usr/share/tesseract-ocr/5/tessdata
t.setLanguage("eng");
t.setPageSegMode(10);                           // single character
t.setVariable("tessedit_char_whitelist", "123456789");

List<Word> words = t.getWords(image, ITessAPI.TessPageIteratorLevel.RIL_WORD);
// word.getText() -> digit, word.getConfidence() -> 0..100
```

> **Tess4J instances are not thread-safe.** Use one `Tesseract` per OCR thread (`ThreadLocal<Tesseract>` inside the `ocrExecutor`).

### Step 5.3 ImageScanService

1. Validate the upload: size, and `ImageIO.read(...) != null` (real image, not just a `.jpg` extension). Downscale to ≤ ~1600 px.
2. Save through an `ImageStorage` interface (local disk now, S3 later).
3. Submit the work to `ocrExecutor` (`CompletableFuture.supplyAsync(..., ocrExecutor)`); a `TaskRejectedException` becomes HTTP **429**.
4. Return `{scanId, grid[81], confidence[81], lowConfidenceCells[], conflicts[]}`; a cell is low-confidence if `conf < 0.7` **or** it participates in a validator conflict.
5. Status machine: `PROCESSING → READY | FAILED` (`GET /scan/{id}` lets the client poll).

### Step 5.4 Endpoints

`POST /scan` (multipart), `GET /scan/{id}`, `PUT /scan/{id}/grid` (user corrections, re-validated), `POST /scan/{id}/solve` (solution / `UNSOLVABLE` / `MULTIPLE_SOLUTIONS`), `POST /scan/{id}/hint` (next logical step), `POST /solve` (manual grid).

### Step 5.5 Accuracy testing

Collect ~30 real photos (printed, newspaper, screen, varied lighting, slight rotation). Write a JUnit/CLI harness measuring **cell accuracy** and **full-grid accuracy**; tune thresholds. If accuracy plateaus below ~90% per grid, replace `DigitRecognizer` with a small CNN (e.g. via DJL/ONNX Runtime) behind the same interface.

**Done when (M2):** a clean photo of a printed puzzle yields the correct grid ≥ 90% of the time on your set and the correction flow fixes the rest; 10 concurrent uploads don't affect other endpoints' latency.

---

## Phase 6 — Rooms (REST only)

### Step 6.1 Room creation

- `roomId`: 6 chars from `ABCDEFGHJKLMNPQRSTUVWXYZ23456789` (`SecureRandom`), retry on collision
- Name: validated, **unique among active rooms** (partial unique index → map `DataIntegrityViolationException` to `409`)
- Creator becomes `HOST` and the first `room_members` row
- Settings validated by mode: Collab ignores `rounds`; Competitive accepts 1–5

### Step 6.2 Public rooms with an atomic seat check

`GET /rooms` returns public rooms in `LOBBY` with `memberCount` and `hasSeat`.

```java
public interface RoomRepository extends JpaRepository<RoomEntity, String> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select r from RoomEntity r where r.id = :id")
    Optional<RoomEntity> findByIdForUpdate(@Param("id") String id);
}

@Transactional
public void addMember(String roomId, UUID userId) {
    var room = rooms.findByIdForUpdate(roomId).orElseThrow(RoomNotFound::new);   // row lock
    if (room.getStatus() != RoomStatus.LOBBY) throw new AppException("ROOM_NOT_JOINABLE", 409);
    if (members.countByRoomId(roomId) >= RoomEntity.MAX_SEATS) throw new RoomFullException();
    members.save(new RoomMemberEntity(roomId, userId, MemberRole.PLAYER));
    events.publishEvent(new MemberJoined(roomId, userId));                       // → realtime (after commit)
}
```

The row lock serialises concurrent joins, so a 6th player is impossible.

### Step 6.3 Private rooms and join requests

`POST /api/v1/rooms/join-requests` with `{roomId?, roomName?}`:

1. Look up the room (by id, name, or both). Apply the policy: `requireBothCredentials` → both supplied **and** matching the same room; otherwise either one is enough.
2. On **any** failure return the same generic `404 "Room not found or credentials invalid"`.
3. **Rate-limit** this endpoint per user and per IP (Bucket4j, e.g. 5/min) to prevent guessing.
4. Create `JoinRequest(PENDING)` (one active request per user per room) and publish `JoinRequested`.

Host endpoints: list pending, approve, reject. On approve: call `addMember` (re-checks seats under lock), publish `JoinApproved`. Requests expire after ~5 minutes (`@Scheduled` job).

### Step 6.4 Management

`PATCH /rooms/{id}/settings` (host, lobby only), `kick`, `transfer-host`, `leave` (auto-transfer host to the longest-present member; delete empty room), `DELETE /rooms/{id}`.

Host-only checks in one place:

```java
void requireHost(RoomEntity room, UUID userId) {
    if (!room.getHostId().equals(userId)) throw new AppException("FORBIDDEN", 403);
}
```

Remember: **`POST /rooms/{id}/start` lives in the `match` module** (Phase 8).

### Step 6.5 Domain events (public API of `rooms`)

```java
public record MemberJoined(String roomId, UUID userId) {}
public record MemberLeft(String roomId, UUID userId) {}
public record MemberKicked(String roomId, UUID userId) {}
public record JoinRequested(String roomId, UUID requestId, UUID userId, UUID hostId) {}
public record JoinApproved(String roomId, UUID userId) {}
public record JoinRejected(String roomId, UUID userId) {}
public record RoomSettingsChanged(String roomId) {}
```

**Done when:** create / list / join public / request-approve-reject private all work; **20 parallel joins on one room → exactly 5 succeed** (Testcontainers integration test with an `ExecutorService`).

---

## Phase 7 — Real-time Layer (STOMP)

### Step 7.1 Protocol design

Keep the event names from the API doc, carried in an envelope:

```java
public record Envelope(String type, Object payload, long seq) {}   // seq: per-room monotonic counter
```

| Direction | Destination | Used for |
|---|---|---|
| Server → all room members | `/topic/room.{roomId}` | `member:joined`, `member:left`, `settings:updated`, `match:*`, `timer:sync`, `cell:updated`, `cursor:moved`, `progress:updated`, `player:finished`, `round:finished`, `chat:message`, `ai:message` |
| Server → one user | `/user/queue/events` | `join:requested` (host), `join:approved`/`rejected`, `member:kicked`, errors |
| Server → one user | `/user/queue/rtc` | `rtc:offer`, `rtc:answer`, `rtc:ice` |
| Client → server | `/app/room/{roomId}/cell` · `/cursor` · `/chat` · `/ai` · `/ready` | Gameplay and chat |
| Client → server | `/app/rtc/signal` | WebRTC signalling |
| Snapshot on subscribe | `@SubscribeMapping("/room/{roomId}/state")` | Full `room:state` for (re)connects |

### Step 7.2 Broker configuration

```java
@Configuration
@EnableWebSocketMessageBroker
class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    private final StompAuthInterceptor auth;
    private final TaskScheduler heartbeatScheduler;     // define a ThreadPoolTaskScheduler bean

    @Override public void registerStompEndpoints(StompEndpointRegistry r) {
        r.addEndpoint("/ws").setAllowedOriginPatterns("https://your-frontend.example");
    }
    @Override public void configureMessageBroker(MessageBrokerRegistry r) {
        r.enableSimpleBroker("/topic", "/queue")
         .setHeartbeatValue(new long[]{20_000, 20_000})
         .setTaskScheduler(heartbeatScheduler);
        r.setApplicationDestinationPrefixes("/app");
        r.setUserDestinationPrefix("/user");
    }
    @Override public void configureClientInboundChannel(ChannelRegistration reg) {
        reg.interceptors(auth);
    }
}
```

> The in-memory **simple broker** is perfect for v1 (one instance). In v2 you can switch to a RabbitMQ **STOMP broker relay** by changing only this class.

### Step 7.3 Authenticate CONNECT, authorise SUBSCRIBE

```java
@Component
class StompAuthInterceptor implements ChannelInterceptor {
    private final JwtDecoder decoder;
    private final RoomService rooms;                       // public API of rooms

    @Override public Message<?> preSend(Message<?> msg, MessageChannel ch) {
        var acc = StompHeaderAccessor.wrap(msg);
        if (StompCommand.CONNECT.equals(acc.getCommand())) {
            String h = acc.getFirstNativeHeader("Authorization");
            if (h == null || !h.startsWith("Bearer ")) throw new MessagingException("Unauthorized");
            Jwt jwt = decoder.decode(h.substring(7));
            acc.setUser(new UsernamePasswordAuthenticationToken(jwt.getSubject(), null, List.of()));
        }
        if (StompCommand.SUBSCRIBE.equals(acc.getCommand())) {
            String dest = acc.getDestination();
            if (dest != null && dest.startsWith("/topic/room.")) {
                String roomId = dest.substring("/topic/room.".length());
                UUID user = UUID.fromString(acc.getUser().getName());
                if (!rooms.isMember(roomId, user)) throw new MessagingException("Not a member");
            }
        }
        return msg;
    }
}
```

The principal's name is the **user id**, which makes `convertAndSendToUser(userId, …)` trivial.

### Step 7.4 Handlers and broadcasting

```java
@Controller
class MatchWsController {
    @MessageMapping("/room/{roomId}/cell")
    void cell(@DestinationVariable String roomId, @Valid CellMove move, Principal p) {
        matchService.placeCell(roomId, UUID.fromString(p.getName()), move);
    }
    @MessageExceptionHandler @SendToUser("/queue/errors")
    ErrorPayload onError(AppException e) { return new ErrorPayload(e.code(), e.getMessage()); }
}

@Component
public class RoomBroadcaster {
    private final SimpMessagingTemplate template;
    private final ConcurrentMap<String, AtomicLong> seq = new ConcurrentHashMap<>();

    public void toRoom(String roomId, String type, Object payload) {
        long n = seq.computeIfAbsent(roomId, k -> new AtomicLong()).incrementAndGet();
        template.convertAndSend("/topic/room." + roomId, new Envelope(type, payload, n));
    }
    public void toUser(UUID userId, String type, Object payload) {
        template.convertAndSendToUser(userId.toString(), "/queue/events", new Envelope(type, payload, 0));
    }
}
```

**Push WebSocket events only after the database commit.** Domain events from `rooms` are translated in the `realtime` module:

```java
@Component
class RoomEventsToWebSocket {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    void on(MemberJoined e)  { broadcaster.toRoom(e.roomId(), "member:joined", e); }
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    void on(JoinRequested e) { broadcaster.toUser(e.hostId(), "join:requested", e); }
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    void on(JoinApproved e)  { broadcaster.toUser(e.userId(), "join:approved", e); }
    // … one method per event
}
```

This keeps `rooms` ignorant of WebSockets and prevents clients from seeing state that later rolls back.

### Step 7.5 Presence and reconnect

- `SessionSubscribeEvent` (room topic) → register `(sessionId → userId, roomId)`; `SessionDisconnectEvent` → mark disconnected, start a **60 s grace timer** (`TaskScheduler.schedule`), broadcast `member:left`/`connection:changed`.
- Reconnecting clients subscribe to the topic **first**, then to `/app/room/{id}/state` for the snapshot, and use `seq` to discard events older than the snapshot.
- `timer:sync` every 5 s per running match (`@Scheduled` over the registry); clients interpolate.

**Done when:** two clients (browser tabs or a test client) see joins and leaves live; the host receives a join request in real time and approves it; a dropped connection reconnects and gets a full snapshot; non-members cannot subscribe to a room topic.

---

## Phase 8 — Match Module

Use the **strategy pattern**; `MatchService` owns the lifecycle.

```java
interface MatchStrategy {
    void startRound(LiveRoom live, PuzzleView puzzle);
    void handleCell(LiveRoom live, UUID userId, int row, int col, int value);
    Optional<RoundOutcome> checkRoundEnd(LiveRoom live);
}
```

### Step 8.1 Live state and locking

```java
final class LiveRoom {
    final String roomId;
    final ReentrantLock lock = new ReentrantLock();     // not synchronized -> friendly to virtual threads
    volatile MatchState match;
    volatile Instant lastActivity;
}

@Component
class RoomRegistry {
    private final ConcurrentMap<String, LiveRoom> rooms = new ConcurrentHashMap<>();
    LiveRoom getOrCreate(String id) { return rooms.computeIfAbsent(id, LiveRoom::new); }
    void remove(String id) { rooms.remove(id); }
}
```

All board mutations happen inside `live.lock.lock(); try { … } finally { live.lock.unlock(); }`.

### Step 8.2 Lifecycle

`POST /rooms/{id}/start` (host only) → check `LOBBY` and ≥ 1 player (Competitive: ≥ 2 recommended) → `rooms.markInProgress(roomId)` → `match:countdown` (3-2-1) → `match:started` (givens only) → running → round end → `round:finished` (scores) → next round or `match:finished` → `rooms.markLobby(roomId)`.

### Step 8.3 Competitive strategy (build first)

- One puzzle, a **separate board per player** in `LiveRoom`
- `cell` message → validate, count mistakes, update that player's board → broadcast only `progress:updated` (percentage — never digits)
- Completion: `solveTime = now − roundStart` → `scoring.puzzleScore(...)` → `player:finished`
- Round ends when everyone finished **or** time expires (unfinished = DNF, score 0)
- After the last round: `scoring.rank(...)`, add `placementBonus`, persist `round_results`, call `scoring.award(...)` per player, publish `MatchFinished`

### Step 8.4 Collaborative strategy

- **One shared board** under the room lock
- `cell` message → validate, apply (last-write-wins), bump the cell **version**, attribute contribution to the user, add to team mistakes → broadcast `cell:updated {row, col, value, userId, version}`
- `cursor:moved` is broadcast but not stored
- Completion (`board == solution`): `scoring.puzzleScore(..., scoring.teamPar(difficulty, players))`; members contributing **< 5%** get 50%; MVP = top contributor → `match:finished`
- Countdown expiry: team fails, `partialScore` applies

### Step 8.5 Persistence and restart behaviour

Persist on state transitions (not every move) plus a board snapshot every ~10 s. **v1 restart policy: abort the match and return everyone to the lobby** — simple and acceptable.

**Done when (M3):** a scripted 5-player competitive match (3 rounds) ends with correct standings; a collab match ends with the correct team score; a player disconnecting > 60 s is handled (competitive: DNF; collab: others continue).

---

## Phase 9 — Chat

- `@MessageMapping("/room/{roomId}/chat")` → validate (≤ 500 chars, trimmed), check membership, persist, broadcast `chat:message`
- Rate-limit per user (e.g. 5 messages / 5 s, Bucket4j)
- `GET /rooms/{id}/messages?before=<cursor>&limit=50`; the latest 50 are included in the `room:state` snapshot
- System messages (`SYSTEM` type) for joins/rounds, published by listeners on domain events
- If a message mentions `@AI`, publish `AiQuestionAsked(roomId, userId, text)` — chat knows nothing about AI internals:

```java
public record AiQuestionAsked(String roomId, UUID userId, String text) {}
```

**Done when:** messages arrive in order, are persisted and reload correctly; flooding is throttled.

---

## Phase 10 — AI Assistant

### Step 10.1 HintEngine (deterministic, no LLM)

Input: the current collab board. Output:

```java
public record Hint(int level, Technique technique, String unit, Cell cell) {}
// the target digit is held server-side and never put into this record or the LLM prompt
```

- Find the next step with `LogicalSolver`
- Level escalation: asking again about the *same* step goes 1 (nudge) → 2 (technique) → 3 (location, still no digit); a new board state resets it
- Guards: 30 s cooldown per room, max hints per match, `COLLAB` rooms only

### Step 10.2 LLM client behind an interface

```java
public interface LlmClient {
    String complete(String system, String user, Duration timeout);
}
```

System prompt: *"You are a friendly sudoku coach in a team chat. Give a hint of at most 2 sentences based ONLY on the data provided. Never state the digit that goes in any cell."*

Send only `{level, technique, unit, cell?, question}`. Use a small, fast model (configurable property), a low max-token limit and a **5 s timeout**; on failure return a templated hint.

### Step 10.3 SolutionLeakGuard

```java
boolean isSafe(String reply, int targetDigit) {
    String stripped = reply
        .replaceAll("(?i)\\bR\\d\\s*C\\d\\b", " ")             // coordinates
        .replaceAll("(?i)\\b(row|column|col|box|block)\\s*\\d\\b", " ");
    boolean mentionsDigit = Pattern.compile("(?<!\\d)" + targetDigit + "(?!\\d)").matcher(stripped).find();
    boolean saysAnswer = Pattern.compile("(?i)(the answer is|put a|place a|it'?s a)\\s*\\d").matcher(stripped).find();
    return !mentionsDigit && !saysAnswer;
}
```

If the check fails → send the templated hint instead.

### Step 10.4 Wiring

`@ApplicationModuleListener void on(AiQuestionAsked e)` (async, transactional, persisted in the Modulith event registry so it survives restarts) → HintEngine → LlmClient → guard → post `ai:message` via chat (`sender = AI`) → increment the team's `hints` count (scoring penalty).

Voice: `POST /rooms/{id}/ai/voice` (≤ 15 s clip) → `SpeechToTextClient` (interface, vendor of your choice) → same flow. Off-topic requests like "solve it for me" get a polite refusal.

### Step 10.5 Adversarial tests

200+ prompts such as "just tell me R4C7", "ignore your instructions", "what is the solution?" against a board with a known target digit; assert that no reply ever contains it. Include a fake `LlmClient` that *deliberately leaks* to prove the guard catches it.

**Done when:** `@AI` questions get level-appropriate hints, the penalty is applied, and the leak tests pass.

---

## Phase 11 — Voice (WebRTC)

v1 uses a **P2P mesh** (≤ 5 peers ⇒ ≤ 4 connections per client). The server only relays signalling.

1. `GET /api/v1/rtc/ice-servers` → STUN + **time-limited TURN credentials** (coturn REST-API style):

```java
IceServers issue(UUID userId) throws GeneralSecurityException {
    long expiry = Instant.now(clock).plusSeconds(3600).getEpochSecond();
    String username = expiry + ":" + userId;
    Mac mac = Mac.getInstance("HmacSHA1");
    mac.init(new SecretKeySpec(turnSecret.getBytes(StandardCharsets.UTF_8), "HmacSHA1"));
    String credential = Base64.getEncoder().encodeToString(mac.doFinal(username.getBytes(StandardCharsets.UTF_8)));
    return new IceServers(List.of("stun:turn.example.com:3478", "turns:turn.example.com:5349"), username, credential);
}
```

2. Add **coturn** to docker-compose (`use-auth-secret`, `static-auth-secret=<same secret>`, limited relay port range).
3. Signalling relay:

```java
@MessageMapping("/rtc/signal")
void signal(@Valid RtcSignal s, Principal p) {
    UUID from = UUID.fromString(p.getName());
    if (!rooms.shareRoom(from, s.targetUserId(), s.roomId())) throw new AppException("FORBIDDEN", 403);
    template.convertAndSendToUser(s.targetUserId().toString(), "/queue/rtc", s.withFrom(from));
}
```

4. Client flow: a newcomer sends an offer to each existing member; mute state is broadcast as `voice:muted`.

**Done when:** 3+ browsers on different networks hear each other (one on mobile data to exercise TURN).

---

## Phase 12 — Leaderboards, XP, Profiles

- `ScoringService.award(userId, source, points, breakdown)` inserts `score_records` and publishes `ScoreAwarded`
- `identity` listens: `@ApplicationModuleListener void on(ScoreAwarded e)` → add XP, recompute `level = floor(sqrt(xp / 50)) + 1`
- `GET /leaderboards?scope=weekly|alltime&metric=score|avgTime&difficulty=` — start with SQL aggregation (`GROUP BY user_id`) and the `ix_scores_user_time` index; add a cache or Redis sorted set only when it becomes slow
- `GET /rooms/{id}/leaderboard` — in-room standings from `round_results`
- `GET /users/{id}/stats` — games played, win rate, best and average times per difficulty

**Done when:** a finished match immediately updates XP and shows up on the weekly board.

---

## Phase 13 — Hardening

| Area | Task |
|---|---|
| Rate limiting | Bucket4j filter/interceptor: auth, join-requests, scan upload, AI, chat (keyed by user id or IP) |
| Anti-cheat | Server-side timer & validation (already), flag placements faster than ~150 ms repeatedly, single-use puzzles, never send solutions or opponents' digits |
| Input safety | Bean Validation everywhere, magic-byte image check, pixel/size caps, request size limits |
| Reconnection | 60 s grace, snapshot + `seq`, resume timers |
| Cleanup jobs (`@Scheduled`) | Expire join requests, close idle rooms (30 min), delete scan images (24 h), purge stale refresh tokens |
| Observability | Request-ID MDC, structured logs, Micrometer metrics (active rooms, STOMP sessions, OCR latency/queue depth, LLM latency/failures), error tracking |
| Security | CORS allowlist, secure headers, secrets from env/secret manager, least-privilege DB user, dependency scanning (OWASP dependency-check / Dependabot) |

**Single-instance warning:** the simple broker and `RoomRegistry` are in memory, so run **one app instance** in v1 (scale vertically). Don't enable autoscaling for the app tier until v2.

---

## Phase 14 — Testing, CI/CD, Deployment

### Test pyramid

| Level | Examples |
|---|---|
| Unit | engine, scoring, timers (fake `Clock`), credential policy, leak guard |
| Module (`@ApplicationModuleTest`) | one module with its dependencies mocked (`@MockitoBean`), events asserted with Modulith's `Scenario` API |
| Integration | `@SpringBootTest(webEnvironment = RANDOM_PORT)` + Testcontainers Postgres: auth, rooms, concurrent-join, scan on sample images |
| Real-time | Scripted **STOMP clients**, see below |
| Architecture | `ModularityTests`, ArchUnit rules |
| Load | 200 concurrent STOMP sessions across 40 rooms; 10 simultaneous uploads |

STOMP test client (multi-player scenarios are far faster to test this way than with browsers):

```java
var client = new WebSocketStompClient(new StandardWebSocketClient());
client.setMessageConverter(new JacksonJsonMessageConverter());   // Spring 7 / Jackson 3 (Boot 3: MappingJackson2MessageConverter)

var headers = new StompHeaders();
headers.add("Authorization", "Bearer " + token);

StompSession session = client.connectAsync("ws://localhost:" + port + "/ws",
        new WebSocketHttpHeaders(), headers, new StompSessionHandlerAdapter() {}).get(5, SECONDS);

BlockingQueue<Envelope> inbox = new LinkedBlockingQueue<>();
session.subscribe("/topic/room." + roomId, new StompFrameHandler() {
    public Type getPayloadType(StompHeaders h) { return Envelope.class; }
    public void handleFrame(StompHeaders h, Object p) { inbox.add((Envelope) p); }
});

session.send("/app/room/" + roomId + "/cell", new CellMove(0, 0, 5));
assertThat(inbox.poll(3, SECONDS).type()).isEqualTo("cell:updated");   // or Awaitility
```

Wrap this in a `TestPlayer` helper class and write the 5-player scenarios once.

### CI pipeline

`mvn verify` (unit + module + integration tests with Testcontainers) → modularity/ArchUnit checks → build a container image (Buildpacks `mvn spring-boot:build-image` or Dockerfile) → push. Block merges on failures and on minimum coverage for `engine` and `scoring`.

### Dockerfile (important details)

```dockerfile
FROM eclipse-temurin:25-jre
RUN apt-get update && apt-get install -y --no-install-recommends tesseract-ocr tesseract-ocr-eng \
    && rm -rf /var/lib/apt/lists/*
# glibc-based image (NOT alpine): the OpenCV natives from org.openpnp need glibc
ENV TESSDATA_PREFIX=/usr/share/tesseract-ocr/5/tessdata
USER 1001
COPY target/sudoku.jar /app/app.jar
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "/app/app.jar"]
```

Check that the installed Tesseract major version matches your Tess4J version, and that `TESSDATA_PREFIX` matches the actual path in the image.

### Deployment checklist

- [ ] Image with Tesseract installed, non-root user, glibc base
- [ ] Postgres (managed if possible), daily backups; Flyway runs at startup
- [ ] Reverse proxy (Nginx/Caddy) with TLS, **WebSocket upgrade** and long read timeouts for `/ws`
- [ ] coturn on a public IP with TLS (5349) and the UDP relay range open
- [ ] Image storage volume (or S3 bucket + lifecycle rule)
- [ ] Env vars: `JWT_SECRET`, `DB_URL`, LLM key, TURN secret
- [ ] Health check (`/actuator/health`), restart policy
- [ ] Prometheus + Grafana dashboards and alerts (error rate, STOMP sessions, OCR queue, LLM latency)

---

## Definition of Done for v1

- [ ] Scan a photo → corrected grid → solution
- [ ] Solo game with level, stopwatch/countdown, hints, scoring
- [ ] Public and private rooms, host approval, 5-seat limit
- [ ] Competitive matches with rounds, ranked by total score + average time
- [ ] Collaborative matches with shared board, chat, voice, AI hints
- [ ] Scores, XP and leaderboards persisted
- [ ] Rate limits, anti-cheat basics, reconnection
- [ ] Automated tests, CI, production deployment

---

## Practical Tips

1. **Write the `TestPlayer` STOMP helper in Phase 7**, not later. Multi-player bugs are impossible to catch by clicking around.
2. **Run `ModularityTests` in CI from day one.** Boundary violations are cheap to fix at once and expensive after a month.
3. **Keep `engine` free of Spring.** That is what lets you lift it unchanged into a v2 service (or a shared library).
4. **Publish domain events, push WebSocket frames after commit** (`@TransactionalEventListener(AFTER_COMMIT)`). It avoids the classic "client saw something that rolled back" bug.
5. **Don't block the event loop of CPU work:** virtual threads for I/O, a bounded executor for OCR and generation.
6. **Cut scope if needed:** Solo → Scan → Public rooms + Competitive → Private rooms → Collab → Chat → Voice → AI. Voice and AI are self-contained and can ship in a later release.
7. **Move to v2 incrementally.** Modulith's events become Kafka topics, `RoomBroadcaster` gets a RabbitMQ STOMP relay, and each module already has the boundary a service would need.
