# Versioning

This document describes how the OpenTelemetry Kotlin SDK uses version numbers and how it
guarantees (or explicitly does not guarantee) compatibility between releases. It is the
Kotlin-specific document required by the
[OpenTelemetry versioning and stability specification][otel-versioning-spec].

It complements, and does not replace, [`RELEASING.md`](RELEASING.md) (the release mechanics) and
[`CONTRIBUTING.md`](CONTRIBUTING.md) (the developer workflow).

## Scope

Every module published from this repository is versioned and released together under a single
version. The canonical version lives in [`gradle.properties`](gradle.properties) (`version`) and
is applied to all artifacts in the `io.opentelemetry.kotlin` group (for example `api`, `sdk-api`,
`implementation`, `compat`, `exporters-otlp`, `semconv`, `config-*`, `instrumentation/*`, ...).

There is no per-module version. If you depend on several OpenTelemetry Kotlin modules, keep them
on the same version.

## Status: pre-1.0

The SDK is currently in the `0.x` line (`version=0.9.0` at the time of writing). We have **not**
declared a `1.0` stable release, so consumers should expect the public surface to keep evolving.
Reaching `1.0` is tracked separately and will be announced; until then the pre-1.0 rules below
apply.

## Versioning scheme

We follow [Semantic Versioning 2.0.0][semver] with the number formatted as `MAJOR.MINOR.PATCH`.

### Before `1.0.0`

Because the API is not yet frozen, the `0.MINOR.PATCH` line is used as follows:

- **MINOR** (`0.8.0` -> `0.9.0`): new functionality, and **any** breaking change to the stable
  public API. Breaking changes are allowed here and are documented under **Migration notes** in
  [`CHANGELOG.md`](CHANGELOG.md).
- **PATCH** (`0.9.0` -> `0.9.1`): backward-compatible bug fixes only. No new public API and no
  breaking changes.

A change that breaks the stable public API between two versions must therefore bump the MINOR
number while we are on `0.x`.

### From `1.0.0` onwards

Once `1.0.0` is declared, standard semver applies:

- **MAJOR**: breaking changes to the stable public API.
- **MINOR**: backward-compatible additions of functionality.
- **PATCH**: backward-compatible bug fixes.

Breaking changes will only ever land in a new MAJOR release.

## Stable vs. experimental API

OpenTelemetry Kotlin is a Kotlin Multiplatform library that is still converging on its final API,
so not every declaration is covered by the compatibility guarantees above. We distinguish two
tiers:

### Experimental API (`@ExperimentalApi`)

Declarations that may still change are annotated with
[`@ExperimentalApi`](api/src/commonMain/kotlin/io/opentelemetry/kotlin/ExperimentalApi.kt), a
`@RequiresOptIn` marker.

- Experimental declarations can change or be **removed at any time, without notice and without a
  version bump**. The contract of `@ExperimentalApi` is precisely "subject to breaking change
  without warning".
- Using them requires an explicit opt-in (the Kotlin compiler warning/`@OptIn`). We deliberately
  keep the opt-in level at `WARNING` so that experiments are easy to try, but treat any code that
  opts in as taking on the churn risk.
- New APIs are added as `@ExperimentalApi` first. [CONTRIBUTING.md](CONTRIBUTING.md) instructs
  contributors to annotate new APIs as experimental "until they are considered stable". Promotion
  from experimental to stable (dropping the annotation) is a non-breaking change; the reverse is
  **not** and will not be done silently.

### Stable public API

A public declaration that is **not** marked `@ExperimentalApi` is part of the stable surface and is
covered by the semver rules above within its version line (MINOR-boundary breaking changes while
pre-1.0, MAJOR-boundary breaking changes from 1.0).

The `compat` module is a thin facade over [opentelemetry-java][otel-java]; its API follows the same
`@ExperimentalApi`/stable distinction and its behaviour is additionally constrained by the
underlying Java SDK.

## How compatibility is enforced

Binary compatibility of the stable public API is not a promise we rely on memory for — it is
checked in CI.

- We use the [Kotlin binary-compatibility-validator][bi-compat-validator]. Each API-bearing module
  keeps a checked-in dump of its public declarations per target, under
  `<module>/api/<target>/<module>.api` (for example `api/api/jvm/api.api`).
- Any change to the public API must regenerate these dumps with `./gradlew apiDump` and commit the
  result; CI fails if the committed dump does not match the code. This makes accidental breaking
  changes visible in the PR diff.
- A dump that can only be satisfied by removing or altering a public symbol is, by definition, a
  breaking change and must be reflected in the version number per the scheme above.

Kotlin Multiplatform specifics:

- **JVM / Android**: guarded by the per-target `.api` dumps described above.
- **JS and iOS/Native**: compiled against the language rules in the
  [Kotlin Multiplatform compatibility guide][kmp-compat]. We target the toolchain/Kotlin version
  documented in [CONTRIBUTING.md](CONTRIBUTING.md); a change to the *minimum* required Kotlin,
  Gradle or (for iOS) Xcode version is considered a breaking change and follows the same versioning
  rules.

## Semantic conventions (`semconv`)

The `semconv` module is **generated** from the upstream OpenTelemetry
[semantic conventions][semconv-registry] registry and exposes those definitions to Kotlin callers.

- The module carries no independent public API of its own (`containsPublicApi=false`); it mirrors
  upstream attribute/event names.
- Upstream semantic conventions are themselves versioned independently. When the registry changes,
  the generated symbols may change with it. Treat generated `semconv` symbols as
  `@ExperimentalApi`: their names and types are subject to upstream evolution and do not, on their
  own, force a MAJOR bump of the SDK.
- The specific registry version that produced a release's `semconv` artifacts is recorded in the
  corresponding entry of [`CHANGELOG.md`](CHANGELOG.md).

## Releases, pre-releases and snapshots

- Stable releases are tagged `vX.X.X` and cut from a `release/vX.X.X` branch by the process in
  [RELEASING.md](RELEASING.md). They are published to the `io.opentelemetry.kotlin` group by the
  release workflow.
- The SDK ships at an approximately **monthly** cadence (a convention, subject to maintainer
  discretion — see RELEASING.md). A release with no user-visible change may be skipped.
- Builds are versioned with a `-SNAPSHOT` suffix when snapshot publishing is enabled
  (`./gradlew publishToMavenLocal -PsnapshotPublish=true`, see CONTRIBUTING.md). **Snapshots are not
  stable**: they are provided for testing the latest work, carry no compatibility guarantees, and
  may be overwritten at any time. Never pin a production dependency on a `-SNAPSHOT`.
- We do not publish `-alpha`/`-rc` pre-releases as a policy today; if that changes it will be
  documented here first.

## Support and security

While the project is pre-1.0 we expect consumers to track the latest release; older `0.x` lines are
not separately maintained. Fixes are applied to `main` and shipped in the next release.

A formal support matrix (how many versions are kept current once we are post-1.0, security-only
backport policy, and end-of-life criteria) will be defined alongside the `1.0` effort and added to
this document. Vulnerabilities should be reported following the
[OpenTelemetry security disclosure policy][otel-security].

## Summary for consumers

| Your situation | What to do |
|----------------|------------|
| You use a stable public API | Safe to upgrade within the same version line; read **Migration notes** before a MINOR (pre-1.0) or MAJOR (≥1.0) bump. |
| You `@OptIn(ExperimentalApi::class)` | Expect it to break between any two versions; re-check the changelog every upgrade. |
| You use generated `semconv` symbols | Treat as experimental; names may follow upstream registry changes. |
| You see a `-SNAPSHOT` | Testing only. Always use a tagged `vX.X.X` release in production. |

[otel-versioning-spec]: https://opentelemetry.io/docs/specs/otel/versioning-and-stability/
[semver]: https://semver.org/
[bi-compat-validator]: https://github.com/Kotlin/binary-compatibility-validator
[kmp-compat]: https://kotlinlang.org/docs/multiplatform/multiplatform-compatibility-guide.html
[semconv-registry]: https://github.com/open-telemetry/semantic-conventions
[otel-java]: https://github.com/open-telemetry/opentelemetry-java
[otel-security]: https://github.com/open-telemetry/.github/blob/main/SECURITY.md
