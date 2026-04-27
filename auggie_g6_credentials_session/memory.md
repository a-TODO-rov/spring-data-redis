# G6 Credentials/Authentication Refactor — Session Memory

> Single source of truth for this session. Update when anything changes.
> If info is here, do NOT re-run the research that produced it.

## Overall goal
Make Project Reactor (`reactor-core`) **optional** in Spring Data Redis (SDR) and Lettuce. Primary driver: GraalVM native image for non-reactive apps. Approach is empirical — use `native-test-app` to iteratively find and fix reachability leaks.

## Locations
- SDR workspace (current repo root): `/Users/aleksandar.todorov/Documents/repos/lettuce-sdr/spring-data-redis`
- Lettuce repo: `/Users/aleksandar.todorov/Documents/repos/lettuce`  (note: flat Maven layout; sources under `src/main/java`, no `lettuce-core/` subdir)
- Native test app: `/Users/aleksandar.todorov/Documents/repos/lettuce-sdr/native-test-app`

## Local build tags
- SDR pom: `4.1.0-REACTOR-OPTIONAL`
- Lettuce pom: `7.6.0-REACTOR-OPTIONAL`

## Redis ports (docker-env + native-test-app config)
- Standalone: 16379
- Cluster: 36379, 36380, 36381
- Sentinel: 26381

## Guard helpers already in place
- Lettuce: `io.lettuce.core.resource.ReactorProvider.checkForReactorLibrary()` — call at reactive entry points.
- SDR: `DefaultRedisCacheWriter.REACTIVE_REDIS_CONNECTION_FACTORY_PRESENT` (package-private guard).

## 19 native scenarios currently green (JVM + native, no reactor on classpath)
basic ops → set/hash → pipeline/scripting/boundops → streams/atomiclong/list/cache → pubsub listener → hash repositories → cluster → sentinel → **password auth** (added 2026-04-24).

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

## Environment gotchas
- System default `java` is Zulu 1.8 on this host. Maven builds fail with `Unrecognized option: --add-exports`. Always `export JAVA_HOME=...azul-17.0.17...` before running SDR/Lettuce builds.
- GraalVM native build needs `JAVA_HOME=/Library/Java/JavaVirtualMachines/graalvm-25.jdk/Contents/Home`.
- Docker compose file for dev env: `/Users/aleksandar.todorov/Documents/repos/lettuce-sdr/lettuce/src/test/resources/docker-env/docker-compose.yml`. Docker Desktop must be running.

