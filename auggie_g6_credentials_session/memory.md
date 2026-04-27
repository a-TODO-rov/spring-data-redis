# G6 Credentials/Authentication Refactor — Session Memory

> Single source of truth for this session. Update when anything changes.
> If info is here, do NOT re-run the research that produced it.

## Overall goal
Make Project Reactor (`reactor-core`) **optional** in Spring Data Redis (SDR) and Lettuce. Primary driver: GraalVM native image for non-reactive apps. Approach is empirical — use `native-test-app` to iteratively find and fix reachability leaks.

## Locations
- SDR workspace (current repo root): `/Users/aleksandar.todorov/Documents/repos/lettuce-sdr/spring-data-redis`
- Lettuce repo: `/Users/aleksandar.todorov/Documents/repos/lettuce`  (note: flat Maven layout; sources under `src/main/java`, no `lettuce-core/` subdir)
- Native test app: `/Users/aleksandar.todorov/Documents/repos/lettuce-sdr/native-test-app`
- JitPack-published forks (consumed by `native-test-app` since 2026-04-27):
  - `com.github.a-TODO-rov:spring-data-redis:reactor-optional-native-SNAPSHOT` (branch `reactor-optional-native`, PR https://github.com/a-TODO-rov/spring-data-redis/pull/1)
  - `com.github.a-TODO-rov:lettuce:reactor-optional-native-SNAPSHOT` (branch `reactor-optional-native`, PR https://github.com/a-TODO-rov/lettuce/pull/2)

## Local build tags
- SDR pom: `4.1.0-REACTOR-OPTIONAL`
- Lettuce pom: `7.6.0-REACTOR-OPTIONAL`
- JitPack-resolved short-SHAs (pinned by `~/.m2/repository/com/github/a-TODO-rov/`):
  - SDR: `reactor-optional-native-a4b985630f-1`
  - Lettuce: `reactor-optional-native-8a1cd42620-1`
- Bytecode major versions (verified 2026-04-27): SDR jar = 61 (Java 17), Lettuce jar = 52 (Java 8). Both binary-compatible with JDK 17+.

## Redis ports (docker-env + native-test-app config)
- Standalone: 16379
- Cluster: 36379, 36380, 36381
- Sentinel: 26381

## Guard helpers already in place
- Lettuce: `io.lettuce.core.resource.ReactorProvider.checkForReactorLibrary()` — call at reactive entry points.
- SDR: `DefaultRedisCacheWriter.REACTIVE_REDIS_CONNECTION_FACTORY_PRESENT` (package-private guard).

## 26 native scenarios currently green (JVM + native, no reactor on classpath; re-verified 2026-04-27 against JitPack artifacts)
basic ops → set/hash → pipeline/scripting/boundops → streams/atomiclong/list/cache → pubsub listener → hash repositories → cluster → sentinel → **password auth** → **transactions (multi/exec)** → **pattern pubsub (PSUBSCRIBE)** → **scan/hscan cursors** → **RedisCallback low-level** → **expiration/TTL/persist** → **bitmap ops** → **ACL username+password auth** (added 2026-04-27).

## Current SDR changes already landed (keep, internal only)
- `DefaultRedisCacheWriter.java` — 1-line guard fix (package-private).
- `LettuceReactiveRedisConnection.java` — `AsyncConnect.STATE` refactor (internal).
- `pom.xml` — local version tag.
Assessed: not public API, minor/patch-level; safe.

## Current Lettuce changes already landed
- `ReactorProvider` added.
- `StatefulRedisClusterConnectionImpl` + `StatefulRedisSentinelConnectionImpl` — lazy-init of `reactive` field (volatile + double-checked locking, guarded by `ReactorProvider.checkForReactorLibrary()`).
- `RedisAuthenticationHandler.subscribe()` — reflection-based Flux subscription to avoid compile-time reactor refs.
- `RedisHandshake` / `RedisURI` — throw UOE for non-immediate providers (reactor required otherwise).

---

## G6 plan — CONFIRMED (user answers: a, b, a, local, b)

### Q1=a Strict G6: base interface must be reactor-free (breaking for reactor users).
### Q2=b Minimum scope: touch only what strict-G6 forces.
### Q3=a Keep `TokenBasedRedisCredentialsProvider` reactive via new sub-interface.
### Q4 Local branch, keep `7.6.0-REACTOR-OPTIONAL`.
### Q5=b SDR: keep return type `RedisCredentialsProvider`; only change default method bodies.

### Lettuce files TO MODIFY
1. `src/main/java/io/lettuce/core/RedisCredentialsProvider.java` — rewrite base:
   - SAM: `CompletionStage<RedisCredentials> resolveCredentialsAsync()`
   - `static from(Supplier)` → `CompletableFuture.completedStage(supplier.get())`
   - `default boolean supportsStreaming()` unchanged
   - NEW `default Closeable subscribeToCredentials(Consumer<RedisCredentials>)` → UOE default
   - REMOVE `Mono resolveCredentials()` and `Flux credentials()` from base
   - Inner `ImmediateRedisCredentialsProvider`: implements `resolveCredentialsAsync()` via `CompletableFuture.completedStage(resolveCredentialsNow())`
2. `src/main/java/io/lettuce/core/ReactiveRedisCredentialsProvider.java` — NEW file:
   - `extends RedisCredentialsProvider`
   - SAM: `Mono<RedisCredentials> resolveCredentials()`
   - `default Flux<RedisCredentials> credentials()` → UOE
   - `default CompletionStage resolveCredentialsAsync()` → `resolveCredentials().toFuture()`
   - `default Closeable subscribeToCredentials(Consumer)` → bridge via `credentials().subscribe(listener)::dispose`
3. `src/main/java/io/lettuce/authx/TokenBasedRedisCredentialsProvider.java` — change `implements RedisCredentialsProvider` → `implements ReactiveRedisCredentialsProvider` + `AutoCloseable`. No body changes.
4. `src/test/java/io/lettuce/core/MyStreamingRedisCredentialsProvider.java` — switch to `implements ReactiveRedisCredentialsProvider`.
5. `src/test/java/io/lettuce/core/RedisHandshakeUnitTests.java` — inner `DelayedRedisCredentialsProvider` → `implements ReactiveRedisCredentialsProvider`.

### Lettuce files NOT touched (explicit scope cut)
- `StaticCredentialsProvider` — inherits new behavior via `ImmediateRedisCredentialsProvider`.
- `RedisAuthenticationHandler` — reflection path still works; reactor still reachable only if token-based is used.
- `RedisHandshake`, `RedisURI` — still handle only immediate; non-immediate throws UOE.

### SDR files TO MODIFY
6. `src/main/java/org/springframework/data/redis/connection/lettuce/RedisCredentialsProviderFactory.java`
   - Remove `import reactor.core.publisher.Mono;`
   - Replace both `return () -> Mono.just(AbsentRedisCredentials.ANONYMOUS);` with `return () -> CompletableFuture.completedStage(AbsentRedisCredentials.ANONYMOUS);`
   - Add `import java.util.concurrent.CompletableFuture;`
   - Return type stays `RedisCredentialsProvider` (per Q5=b).

### Build commands
- Lettuce: `mvn install -pl :lettuce-core -DskipTests -Dmaven.test.skip=true -Dformatter.skip=true -Dimpsort.skip=true -Dgpg.skip=true -Denforcer.skip=true -Dlicense.skip=true -Dcheckstyle.skip=true` in `/Users/aleksandar.todorov/Documents/repos/lettuce` (default Java).
- SDR: same flags, run `mvn install` under SDR workspace.
- Native test app verification: `cd native-test-app && mvn -Pnative native:compile` then run `./target/native-test-app`.

### Kotlin IC caveat (seen before)
If Lettuce Kotlin build fails with `IllegalArgumentException: 25.0.1`, clear `target/kotlin-ic/` and retry.

## Key code facts (do NOT re-research)
- `RedisCredentials.just(String, CharSequence)` / `just(String, char[])` — static factories.
- `AbsentRedisCredentials.ANONYMOUS` — SDR's sentinel empty-credentials object (implements `RedisCredentials`, not the provider).
- `resolveCredentialsNow()` is on `ImmediateRedisCredentialsProvider` only.
- Lettuce internal callers of `resolveCredentials()`: none in `src/main/java` (only test code). Internal paths use `resolveCredentialsNow()` (immediate) or reflection (streaming).
- `RedisHandshake.java` lines ~203–250 — immediate-only dispatch.
- `RedisURI.java` line ~982 — immediate-only toString handling.
- `RedisAuthenticationHandler.subscribe()` — already reflection-based; no change needed for this PR.

## Known remaining gaps (not in scope for this PR)
- ~~Password-authenticated native scenario~~ DONE (2026-04-24, `testPasswordAuthentication` in ScenarioRunner).
- `StreamMessageListenerContainer` scenario.
- `PSUBSCRIBE` / RESP3 scenarios.
- Master/replica read-from-replica routing.
- TLS/SSL (low risk, untested).
- `TokenBasedRedisCredentialsProvider` still reactor-bound (accepted per Q3=a).

## Session log (append as work progresses)
- 2026-04-24: Plan confirmed. Starting implementation of Lettuce edits 1–5, then SDR edit 6, then builds.
- 2026-04-24: Lettuce edits done:
  - `RedisCredentialsProvider.java` rewritten: base is Reactor-free, SAM is `resolveCredentialsAsync()`, adds `subscribeToCredentials(Consumer)`, `from(Supplier)` uses `CompletableFuture.completedFuture`. `ImmediateRedisCredentialsProvider` now implements `resolveCredentialsAsync()` (not `resolveCredentials`).
  - New file `ReactiveRedisCredentialsProvider.java`: extends base, SAM `resolveCredentials() : Mono`, default `credentials() : Flux` UOE, bridges for async + subscribeToCredentials.
  - `TokenBasedRedisCredentialsProvider` now `implements ReactiveRedisCredentialsProvider`. Javadoc link fixed.
  - Test classes `MyStreamingRedisCredentialsProvider` and inner `DelayedRedisCredentialsProvider` (in `RedisHandshakeUnitTests`) switched to `ReactiveRedisCredentialsProvider`.
  - Used Java 8 compatibility: `CompletableFuture.completedFuture(...)` (not `completedStage`, which is Java 9+).
- 2026-04-24: SDR edits done:
  - `RedisCredentialsProviderFactory.java`: removed `reactor.core.publisher.Mono` import, added `java.util.concurrent.CompletableFuture`, replaced two `() -> Mono.just(AbsentRedisCredentials.ANONYMOUS)` with `() -> CompletableFuture.completedFuture(AbsentRedisCredentials.ANONYMOUS)`.
  - `LettuceConverters.java` (newly discovered breakage): two call sites at lines 453 and 468 previously did `.resolveCredentials().block()`. Changed to `.resolveCredentialsAsync().toCompletableFuture().join()`.
- 2026-04-24: Builds green (JDK 17 required for SDR compile; system default is Java 8 Zulu which fails with `--add-exports` error — use `JAVA_HOME=/Users/aleksandar.todorov/Library/Java/JavaVirtualMachines/azul-17.0.17/Contents/Home`):
  - `mvn install -pl :lettuce-core ...` in lettuce repo → BUILD SUCCESS (14s)
  - `mvn install ...` in SDR → BUILD SUCCESS (6s)
  - `native-test-app` native image → BUILD SUCCESS (1m 16s, 6.32GB peak RSS)
- 2026-04-24: Native image ran — NO reactor `NoClassDefFoundError`. Only failure was `Connection refused: localhost/127.0.0.1:16379` because Docker isn't running on this host. Scenarios themselves not yet exercised end-to-end post-refactor.
- 2026-04-24: Docker started; `docker compose up -d` in `/Users/aleksandar.todorov/Documents/repos/lettuce-sdr/lettuce/src/test/resources/docker-env`. All endpoints responded PONG (16379, 36379-36381, 26381).
- 2026-04-24: native-test-app run post-G6 — **18/18 scenarios PASSED** (0 failed). G6 refactor verified end-to-end in native image without reactor on classpath.
- 2026-04-24: Added `testPasswordAuthentication` scenario to `native-test-app/src/main/java/com/example/app/ScenarioRunner.java`:
  - Uses `redis-standalone-4` on **port 6484** (not used by any other scenario; up via docker-compose).
  - Pattern: unauth admin CF → `serverCommands().setConfig("requirepass", "scnpass")` → authed CF built via `RedisStandaloneConfiguration.setPassword(RedisPassword.of(pwd))` → read/write ops → reset password via authed CF.
  - New imports: `RedisConnection`, `RedisPassword`, `RedisStandaloneConfiguration` (all already in SDR).
  - `RedisServerCommands.setConfig(String,String)` is `void`; `RedisConnection extends AutoCloseable` (try-with-resources works).
  - `RedisStandaloneConfiguration.setPassword` takes `RedisPassword` (NOT String).
  - `LettuceConnectionFactory#setPassword(String)` exists but is `@Deprecated since 2.0` — use the configuration object.
- 2026-04-24: Native image rebuilt in 1m 13s, ran successfully — **19/19 scenarios PASSED** (0 failed). New credentials path exercised end-to-end in native image. Password cleanup verified via `CONFIG GET requirepass` → empty.
- 2026-04-24: `native-test-app` source path is outside the SDR workspace root, so `str-replace-editor` cannot edit it directly — use shell/python for edits to that file.
- 2026-04-24: Test-class compilation NOT verified (skipped via `-Dmaven.test.skip=true`). Known callers of `.resolveCredentials()` / `.credentials()` in Lettuce test sources (31 call sites across 7 test files) will break in upstream-PR scenarios. Documented as out-of-scope; would require migrating those tests to `resolveCredentialsAsync()` / cast to `ReactiveRedisCredentialsProvider`.
- 2026-04-24: SDR test compilation fixed:
  - `LettuceTestClientResources.java`: EventBus anonymous class updated to match new Lettuce 7.x API — replaced `Flux<Event> get()` with `Closeable subscribe(Consumer<Event>)`, added `ReactiveEventBus reactive()` stub (throws UOE). Removed `reactor.core.publisher.Flux` import; added `io.lettuce.core.event.ReactiveEventBus` and `java.util.function.Consumer`.
  - `LettuceConnectionFactoryUnitTests.java`: 11 call sites migrated from `.resolveCredentials().block()` to `.resolveCredentialsAsync().toCompletableFuture().join()`.
  - `LettuceConvertersUnitTests.java`: 4 call sites migrated same way.
  - `mvn test-compile` → BUILD SUCCESS (15s).
  - `mvn test -Dtest=LettuceConnectionFactoryUnitTests,LettuceConvertersUnitTests` → **128 tests, 0 failures, 0 errors, 1 skipped**. Both unit test classes pass cleanly.

## Lettuce commits absorbed (2026-04-27)
Three upstream Lettuce commits on `reactor-optional-minimal` branch were integrated into SDR:

### 534d67071 — Implement reactive facade
- `StatefulRedisConnection.reactive()` REMOVED from the interface. Method kept only on `StatefulRedisConnectionImpl`.
- New static factory: `RedisReactiveCommands.from(StatefulRedisConnection)` — checks `instanceof StatefulRedisConnectionImpl`.
- All Lettuce internal call sites migrated from `.reactive()` to `RedisReactiveCommands.from(...)`.
- SDR impact (3 main-src, 2 test-src sites):
  - `LettuceReactiveRedisConnection.java:230` — `((StatefulRedisConnection)conn).reactive()` → `RedisReactiveCommands.from(...)`. Added import of `RedisReactiveCommands`.
  - `LettuceReactiveRedisClusterConnection.java:357,363` — `StatefulRedisConnection::reactive` method refs → `RedisReactiveCommands::from`.
  - `LettuceReactiveRedisConnectionUnitTests.java:58` — mock type changed from `StatefulRedisConnection` to `StatefulRedisConnectionImpl` (so `instanceof` check in `from()` passes).
  - `LettuceReactiveRedisClusterConnectionUnitTests.java:58` — same mock type change.
  - PubSub mocks unaffected (`StatefulRedisPubSubConnection.reactive()` still on interface).
  - `StatefulRedisClusterConnection.reactive()` still on interface — no SDR change needed for cluster direct calls.

### e1580925709 — Trace context propagation without Reactor dependency
- New `TraceContextProvider` interface, changes to `BraveTracing` and `MicrometerTracing`.
- SDR impact: **none** (SDR doesn't use `TraceContextProvider` directly).

### 1c4202fd514 — Dynamic commands with optional reactive dependency
- `RedisCommandFactory` and `DeclaredCommandMethod` refactored to lazily check Reactor availability.
- SDR impact: **none** (SDR doesn't use `RedisCommandFactory`).

### Verification (2026-04-27)
- Lettuce reinstalled from `reactor-optional-minimal` HEAD (1c4202f).
- SDR `mvn clean compile` → BUILD SUCCESS.
- SDR `mvn test-compile` → BUILD SUCCESS.
- Unit tests: 161 run, 0 failures (LettuceReactiveRedisConnectionUnitTests, LettuceReactiveRedisClusterConnectionUnitTests, LettuceReactiveSubscriptionUnitTests, LettuceReactivePubSubCommandsUnitTests, LettuceConnectionFactoryUnitTests, LettuceConvertersUnitTests).
- Reactive integration tests: 200 run, 0 failures (LettuceReactiveStringCommandsIntegrationTests, LettuceReactiveServerCommandsIntegrationTests, LettuceReactiveKeyCommandsIntegrationTests, LettuceConnectionFactoryIntegrationTests).
- SDR installed, native-test-app rebuilt (1m 3s), **19/19 native scenarios PASSED**.

## Reactive facade pattern applied to ALL connection interfaces (2026-04-27)

### Lettuce changes (reactor-optional-minimal branch)
Removed `reactive()` from 4 interfaces total (standalone done in 534d670, rest done today):

| Interface | `reactive()` removed | `from()` factory added to |
|---|---|---|
| `StatefulRedisConnection` | 534d670 | `RedisReactiveCommands.from()` |
| `StatefulRedisPubSubConnection` | today | `RedisPubSubReactiveCommands.from()` |
| `StatefulRedisClusterConnection` | today | `RedisAdvancedClusterReactiveCommands.from()` |
| `StatefulRedisClusterPubSubConnection` | today | `RedisClusterPubSubReactiveCommands.from()` |
| `StatefulRedisSentinelConnection` | today | `RedisSentinelReactiveCommands.from()` |

Additional Lettuce fixes:
- `@Override` removed from `StatefulRedisClusterConnectionImpl.reactive()` and `StatefulRedisSentinelConnectionImpl.reactive()` (no longer overriding interface method; they extend `RedisChannelHandler`, not an impl class with `reactive()`).
- `RedisCommandFactory`: `((StatefulRedisClusterConnection) connection).reactive()` → `RedisAdvancedClusterReactiveCommands.from(...)`.
- `RedisClusterPubSubReactiveCommandsImpl`: `StatefulRedisPubSubConnection::reactive` → `RedisPubSubReactiveCommands::from`.
- Kotlin extensions: `StatefulRedisClusterConnectionExtensions.kt` and `StatefulRedisSentinelConnectionExtensions.kt` updated to use `from()` factories.
- `RedisClusterPubSubReactiveCommands.from()` uses `StatefulRedisPubSubConnectionImpl` (public parent) with cast to `RedisClusterPubSubReactiveCommands` since `StatefulRedisClusterPubSubConnectionImpl` is package-private.

### SDR changes
| File | Change |
|---|---|
| `LettuceReactiveRedisConnection.java` | Cluster branch: `.reactive()` → `RedisAdvancedClusterReactiveCommands.from(...)` |
| `LettuceReactiveRedisClusterConnection.java` | `StatefulRedisClusterConnection::reactive` → `RedisAdvancedClusterReactiveCommands::from` |
| `LettuceReactivePubSubCommands.java` | `pubSubConnection.reactive()` → `RedisPubSubReactiveCommands.from(pubSubConnection)` |
| `LettuceReactiveSubscription.java` | `connection.reactive()` → `RedisPubSubReactiveCommands.from(connection)` |
| `LettuceReactivePubSubCommandsUnitTests.java` | Mock type: `StatefulRedisPubSubConnection` → `StatefulRedisPubSubConnectionImpl` |
| `LettuceReactiveSubscriptionUnitTests.java` | Mock type: `StatefulRedisPubSubConnection` → `StatefulRedisPubSubConnectionImpl` |

### Verification
- Lettuce: `mvn clean compile` BUILD SUCCESS
- SDR: `mvn clean compile` BUILD SUCCESS, `mvn test-compile` BUILD SUCCESS
- Unit tests: 161 run, 0 failures
- Integration tests: 200 run, 0 failures
- Native image: rebuilt (1m 10s), **19/19 scenarios PASSED**

## 2026-04-27 — JitPack migration, PRs, JDK 17 source level, POC repo prep

### Switched native-test-app from local `~/.m2` to JitPack-published forks
- Removed local SDR (`4.1.0-REACTOR-OPTIONAL`) and Lettuce (`7.6.0-REACTOR-OPTIONAL`) coords from `native-test-app/pom.xml`.
- Added `jitpack.io` `<repository>` block with `<snapshots><enabled>true</enabled></snapshots>`.
- Added explicit deps:
  - `com.github.a-TODO-rov:spring-data-redis:reactor-optional-native-SNAPSHOT`
  - `com.github.a-TODO-rov:lettuce:reactor-optional-native-SNAPSHOT`
- Both must be declared explicitly: SDR marks Lettuce as `<optional>true</optional>` so it is **not** transitively pulled. Verified by running `mvn dependency:tree` without the explicit Lettuce dep — neither lettuce nor any Lettuce class appeared.
- JitPack-resolved jars (cached locally):
  - `~/.m2/repository/com/github/a-TODO-rov/spring-data-redis/reactor-optional-native-a4b985630f-1/spring-data-redis-...jar` (~2.4 MB)
  - `~/.m2/repository/com/github/a-TODO-rov/lettuce/reactor-optional-native-8a1cd42620-1/lettuce-...jar` (~2.6 MB)

### PRs opened on the forks
- SDR: https://github.com/a-TODO-rov/spring-data-redis/pull/1 — *Reactor optional changes in SDR* (+303 / −57, 18 files, 3 commits, OPEN).
- Lettuce: https://github.com/a-TODO-rov/lettuce/pull/2 — *Reactor optional changes in Lettuce* (+2475 / −648, 83 files, 34 commits, OPEN).
- Asymmetry is real: SDR's job was just to stop assuming reactor types are loadable; the structural decoupling lives in Lettuce.

### Lowered `<java.version>` from 25 → 17 in `native-test-app/pom.xml`
- Reason: Spring Boot 4 baseline is Java 17 (only the **native image** requires JDK 25 / GraalVM 25). Source-level 25 was forcing GraalVM 25 even for plain `mvn package`.
- Verified end-to-end:
  - `mvn -DskipTests clean package` on **Zulu 17.0.17** (plain HotSpot, no GraalVM): BUILD SUCCESS in 2.0s.
  - `java -jar target/native-test-app-0.0.1-SNAPSHOT.jar` on Zulu 17: **26 passed, 0 failed**, startup 0.9s.
  - `mvn -Pnative -DskipTests clean native:compile` on GraalVM 25.0.2: BUILD SUCCESS in 1m 14s.
  - `./target/native-test-app`: **26 passed, 0 failed**, startup 0.064s.
- Cross-JDK build mode (jar built on JDK 17, native image on GraalVM 25) is officially supported by Spring (per Boot's docs and paketo-buildpacks/spring-boot#562). For this POC the divergence is irrelevant — verified by the 26/26 native pass.

### Verification: zero `reactor-*` artifacts on the classpath
Command for the README's "verify before running" step:
```
mvn dependency:tree | grep -iE ":reactor-|reactor-core|reactor-pool|spring-data-redis|lettuce"
```
Output (exactly two lines):
```
[INFO] +- com.github.a-TODO-rov:spring-data-redis:jar:reactor-optional-native-SNAPSHOT:compile
[INFO] \- com.github.a-TODO-rov:lettuce:jar:reactor-optional-native-SNAPSHOT:compile
```
No `io.projectreactor:*` anywhere in the tree. Confirms the *reactor-optional* contract holds at the dependency-graph level, before any code runs.

### `native-test-app` repo prepared for GitHub
- Added `.gitignore` covering Maven (`target/`), IntelliJ (`.idea/`, `*.iml`), Eclipse, VS Code, NetBeans, GraalVM dumps/hprof/reports, OS junk, logs, Spring Boot scaffolding (`HELP.md`), and **`memory.md`** (this file's own counterpart in the test-app repo, kept out of upstream).
- Added `README.md` (~180 lines) covering: PR table with line-counts, prerequisites matrix (any-JDK-17 vs GraalVM 25), classpath-verification step, expanded Redis topology with explicit service→port mapping for the upstream compose file (link: https://github.com/a-TODO-rov/lettuce/blob/main/src/test/resources/docker-env/docker-compose.yml), JVM run instructions, native run instructions, scenario list, project layout, notes.
- `git status` end state: only `.gitignore`, `README.md`, `pom.xml`, `src/` are tracked / about-to-be-tracked. `target/`, `.idea/`, `memory.md` correctly ignored.

### Spurious 401 from settings.xml
- During `mvn dependency:tree` Maven hits `https://artifactory.dev.redislabs.com/.../cloud-maven-dev` for `spring-data-build:4.1.0-SNAPSHOT` parent metadata and gets 401. Build still succeeds (JitPack-resolved jar carries enough of its parent baked in). Configured somewhere in `~/.m2/settings.xml`, **not** in `native-test-app/pom.xml`. Harmless but noisy for external clones.

## Environment gotchas
- System default `java` is Zulu 1.8 on this host. Maven builds fail with `Unrecognized option: --add-exports`. Always `export JAVA_HOME=...azul-17.0.17...` before running SDR/Lettuce builds.
- **JDK matrix for `native-test-app` (since 2026-04-27, after `<java.version>` was lowered from 25 to 17):**
  - JVM mode (`mvn package` + `java -jar`): any JDK 17+ — verified on Zulu 17.0.17.
  - Native mode (`mvn -Pnative native:compile`): still requires `JAVA_HOME=/Library/Java/JavaVirtualMachines/graalvm-25.jdk/Contents/Home` (Spring Boot 4 / Spring Framework 7 native baseline).
  - Inspecting the dependency tree only: any JDK 17+. Spring Boot 4 starter parent already declares `<java.version>17</java.version>` — `native-test-app/pom.xml` no longer overrides it.
- Docker compose file for dev env: `/Users/aleksandar.todorov/Documents/repos/lettuce-sdr/lettuce/src/test/resources/docker-env/docker-compose.yml` (also published at https://github.com/a-TODO-rov/lettuce/blob/main/src/test/resources/docker-env/docker-compose.yml). Required env: `REDIS_ENV_WORK_DIR=$(pwd)/work`. Docker Desktop must be running. Minimum services for the POC: `standalone-stack` (16379), `clustered-stack` (36379-36381), `redis-standalone-4` (6484 + 26381).

