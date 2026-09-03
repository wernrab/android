# OpenCloud Android

Native Android client for OpenCloud (fork of ownCloud Android client). Kotlin, Gradle multi-module.

## Modules

- `opencloudDomain` — pure Kotlin business logic: models, `UseCase`s, repository *interfaces*. No Android framework deps beyond androidx.appcompat.
- `opencloudData` — implements domain repository interfaces. Contains Room (local) and remote (HTTP/WebDAV) data sources, mappers between entities/remote DTOs and domain models, and DB migrations.
- `opencloudComLibrary` — low-level networking/OpenCloud API client (OkHttp, Moshi), built on top of the `android-dav` library (via JitPack, see `com.github.opencloud-eu:android-dav` in `opencloudComLibrary/build.gradle`).
- `opencloudApp` — the Android application: UI (`presentation`), `ViewModel`s, Koin DI wiring, workers, services, sync adapters. Depends on `opencloudDomain` and `opencloudData`.
- `opencloudTestUtil` — shared test fixtures/factories (e.g. `OCCapability.kt`) used across modules' test sources.

Each feature (e.g. `capabilities`, `spaces`, `sharing`, `transfers`) follows the same layering across modules:
- `opencloudDomain/.../domain/<feature>/{model, usecases}` + a `<Feature>Repository` interface.
- `opencloudData/.../data/<feature>/{db, datasources, datasources/mapper, datasources/implementation, repository}` implementing that interface — typically a `Local<Feature>DataSource` (Room) and `Remote<Feature>DataSource` (network) combined in `OC<Feature>Repository`.
- `opencloudApp/.../presentation/...` for `ViewModel`s and UI, plus wiring in `dependecyinjection/` (`UseCaseModule`, `RepositoryModule`, `LocalDataSourceModule`, `RemoteDataSourceModule`, `ViewModelModule`, `CommonModule` — Koin modules).

Dependency direction is strictly `opencloudApp -> opencloudData -> opencloudDomain` (data depends on domain to implement its interfaces; domain has no dependency on data or app).

The app has a single product flavor dimension `management` with flavor `original` (an `mdm` flavor is commented out in `opencloudApp/build.gradle`). Build/task names therefore include `Original`, e.g. `testOriginalDebugUnitTest`, `connectedOriginalDebugAndroidTest`.

## Build & test

- Build: `./gradlew clean build` (first run downloads the Gradle wrapper; requires Android SDK, API 36 compile/target, min SDK 26, JDK 17).
- Unit tests per module: `./gradlew :opencloudDomain:testDebugUnitTest`, `:opencloudData:testDebugUnitTest`, `:opencloudComLibrary:testDebugUnitTest`, `:opencloudApp:testOriginalDebugUnitTest`.
- Single test class: append `--tests "fully.qualified.ClassName"`, e.g. `./gradlew :opencloudDomain:testDebugUnitTest --tests "eu.opencloud.android.domain.capabilities.usecases.GetStoredCapabilitiesUseCaseTest"`.
- Instrumented/connected tests (need device/emulator): `./gradlew :opencloudData:connectedAndroidTest`, `./gradlew :opencloudApp:connectedOriginalDebugAndroidTest -Pandroid.testInstrumentationRunnerArguments.annotation=eu.opencloud.android.IntegrationTest`.
- Lint/static analysis: `detekt` plugin is applied to all subprojects (config in `config/detekt/detekt.yml`, `maxIssues: 0`). Run with `./gradlew detekt`. `check_code_script.sh` checks for GPL/Apache license headers in `.java`/`.kt` files under each module's `src/` and runs `./gradlew ktlintFormat`.
- Coverage: `./gradlew jacocoAggregatedReport` (aggregates JVM unit test coverage across `opencloudApp`, `opencloudComLibrary`, `opencloudData`, `opencloudDomain`); `jacocoAggregatedCoverageVerification` enforces a 20% minimum line coverage.
- Test naming: unit tests mirror the source package path 1:1 (e.g. `.../domain/capabilities/usecases/GetStoredCapabilitiesUseCaseTest.kt` tests `.../domain/capabilities/usecases/GetStoredCapabilitiesUseCaseTest.kt`). Shared/common test helpers for a module live under `src/test-common/java`.

## Conventions

- Every source file starts with a GPLv2 (or, for newer files, Apache-2.0) license header comment — copy the header from a neighboring file in the same module rather than omitting it.
- Package root is `eu.opencloud.android`; domain code is under `.domain`, data under `.data`.
- Use cases extend `BaseUseCase<ResultType, Params>` and take a nested `data class Params(...)` even for a single parameter.
- Dependency injection uses Koin; new use cases/repositories/data sources/view models must be registered in the matching module under `opencloudApp/src/main/java/eu/opencloud/android/dependecyinjection/`.
- Room schema changes: `opencloudData` uses KSP with `room.schemaLocation` pointing at `opencloudData/schemas` — bump the DB version and add a migration (see `data/migrations`) rather than just changing an `Entity`.
- The `android-dav` composite build integration in `settings.gradle`/root `build.gradle` is normally disabled (commented out) — only used locally for testing unstacked `android-dav` fixes; leave commented unless explicitly asked to enable it.
- Branch names for contributions follow `feature/`, `fix/`, `improvement/`, `technical/` prefixes (CI matches on these); commit messages follow Conventional Commits (`feat:`, `fix:`, `test:`, `chore:`).
