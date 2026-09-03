---
title: SDK Contract Refinements - Plan
type: feat
date: 2026-09-04
artifact_contract: x-unified-plan/v1
artifact_readiness: implementation-ready
product_contract_source: x-plan-bootstrap
execution: code
---

# SDK Contract Refinements

## Goal Capsule

Port four narrowly useful behaviors from recent `Meziantou.NET.Sdk` changes into the Headless SDK without importing broader opinionated xUnit behavior. Consumers gain a less noisy ordinal-comparison rule, flexible third-party notice discovery, a configurable minimum-test guard, and conditional suppression of xUnit's generated entry-point warnings.

## Product Contract

### Requirements

- **R1 — Ordinal comparison policy:** Ship `MA0002.report_only_non_ordinal = true` in both the packaged analyzer configuration and the scaffolded `.editorconfig`, while retaining `MA0002` at warning severity.
- **R2 — Notice discovery:** Recognize project-root third-party notices using ordered probes for `THIRD-PARTY-NOTICES.TXT`, `.txt`, `.MD`, then `.md`, and pack exactly one matching file at package root. On case-insensitive filesystems, the first probe that resolves wins; exact entry casing is not a cross-platform contract.
- **R3 — Minimum test guard:** Expose `MinimumExpectedTests`, default it to `1`, and use it in the static Microsoft Testing Platform arguments. An explicit positive value replaces the default, `0` omits the argument, and `EnableDefaultTestSettings=false` still removes all SDK-owned arguments.
- **R4 — xUnit generated entry point:** Append `XUNIT_ENTRYPOINT_DISABLE_WARNINGS` exactly once when a consumer directly references `xunit.v3`, `xunit.v3.mtp-v1`, `xunit.v3.mtp-v2`, `xunit.v3.mtp-off`, `xunit.v3.core`, `xunit.v3.core.mtp-v2`, or `xunit.v3.core.mtp-off`. Preserve existing constants and do not infer xUnit from test-project classification.
- **R5 — xUnit opt-out:** Expose `EnableXunitEntryPointDisableWarnings`, defaulting to `true` when R4 detects xUnit v3; `false` prevents the SDK from adding the constant.
- **R6 — Public contract:** Document the new consumer properties introduced by R3-R5, and protect each R1-R5 behavior with package or consumer-evaluation integration tests.

### Acceptance Examples

- **AE1 (R1):** The packed analyzer editorconfig and newly scaffolded `.editorconfig` contain the MA0002 option and retain `dotnet_diagnostic.MA0002.severity = warning`.
- **AE2 (R2):** Each supported spelling produces exactly one root notice entry with the source content; exact entry casing is asserted only on a case-sensitive filesystem.
- **AE3 (R2):** On a case-sensitive filesystem, a consumer containing `.TXT` and `.md` variants packs only `.TXT` according to precedence; the test reports a platform skip where the fixture cannot exist.
- **AE4 (R3):** A default test consumer evaluates one `--minimum-expected-tests 1` pair; setting `MinimumExpectedTests=5` evaluates one pair with `5`; setting it to `0` or disabling default test settings evaluates neither.
- **AE5 (R4, R5):** A consumer with xUnit v3 receives the constant once, a consumer without xUnit does not, and the opt-out removes it while preserving unrelated constants.

### Scope Boundaries

- Do not enable full xUnit parallelization or generate assembly-level parallelization attributes.
- Do not generate global test helpers or aliases.
- Do not alias `Assert` to another assertion library or add an assertion dependency.
- Do not change MTP-only hosting, SDK-owned extension versions, analyzer severities beyond the retained MA0002 warning, or unrelated packaging policy.

### Product Contract Key Decisions

1. **Port only the four approved behaviors** *(session-settled: user-approved — chosen over broad Meziantou policy parity: the user explicitly deferred the more magical xUnit defaults for discussion).* Governs R1-R6.

## Planning Contract

### Key Technical Decisions

1. **Keep MSBuild behavior evaluation-time and consumer-overridable.** The MTP runner reads `TestingPlatformCommandLineArguments` during project evaluation, so the minimum-test property and argument remain in the existing static property groups.
2. **Resolve notice variants before packing.** Use an ordered scalar path with empty guards, mirroring README discovery, instead of including four items that may collide across filesystems.
3. **Recognize the supported xUnit v3 package family explicitly.** Detect the R4 package identities in `.targets`, where `PackageReference` items exist, and gate the constant with the upstream-compatible positive opt-out property.
4. **Test through packed consumer contracts.** Package-content and generated-consumer integration tests are the authority because the changed files ship inside the SDK packages and behave differently across consumption modes.

### Assumptions

- `MinimumExpectedTests=0` omits only the SDK-supplied argument because MTP rejects zero; it defers to the platform's built-in behavior and does not promise that a zero-test run succeeds. `EnableDefaultTestSettings=false` remains the way to remove all SDK-owned defaults.
- `EnableXunitEntryPointDisableWarnings` is a new public MSBuild contract and intentionally matches the upstream property name.

### Risks and Mitigations

- **Duplicate constants across imports:** guard against an existing constant and assert a single occurrence in both SDK and PackageReference consumption.
- **Case-sensitive notice collisions:** ordered probes select one path deterministically; integration coverage creates multiple variants only after confirming the filesystem can distinguish them.
- **Divergent analyzer configs:** update and independently assert the packaged configuration and scaffold template.
- **Evaluation-order regressions:** keep test argument composition static and validate multiple SDK consumption modes.

## Implementation Units

### U1 — Align the MA0002 analyzer option

**Covers:** R1, R6; AE1

**Files:**

- `src/Headless.NET.Sdk/configurations/Headless.NET.Sdk.Analyzers.editorconfig`
- `src/Headless.NET.Sdk/configurations/editorconfig.txt`
- `tests/Headless.NET.Sdk.Tests.Integrations/SdkIntegrationTests.PackageAssets.cs`
- `tests/Headless.NET.Sdk.Tests.Integrations/SdkIntegrationTests.Scaffolding.cs`

**Approach:** Add the option adjacent to the retained MA0002 warning in both sources. Extend package-content and scaffold assertions so neither copy can drift unnoticed.

**Verification:** Focused package-assets and scaffolding integration tests prove both shipped paths.

### U2 — Discover third-party notice filename variants

**Covers:** R2, R6; AE2, AE3

**Files:**

- `src/Headless.NET.Sdk/build/SupportPackageInformation.targets`
- `tests/Headless.NET.Sdk.Tests.Integrations/SdkIntegrationTests.PackageAssets.cs`

**Approach:** Resolve the four supported spellings with ordered, empty-guarded probes matching upstream behavior, then pack only the resolved path at package root. Convert the existing notice test to cover each spelling and add a filesystem-gated precedence case without asserting impossible cross-platform casing guarantees.

**Verification:** Pack generated consumers and inspect the resulting nupkg entries and nuspec metadata.

### U3 — Make test defaults configurable and xUnit-aware

**Covers:** R3-R6; AE4, AE5

**Files:**

- `src/Headless.NET.Sdk/build/SupportTestProjects.targets`
- `tests/Headless.NET.Sdk.Tests.Integrations/ConsumerProject.cs`
- `tests/Headless.NET.Sdk.Tests.Integrations/SdkIntegrationTests.ProjectTypes.cs`
- `tests/Headless.NET.Sdk.Tests.Integrations/ContractConsumerBehaviorTests.Testing.cs`
- `README.md`
- `src/Headless.NET.Sdk.Test/README.md`

**Approach:** Default `MinimumExpectedTests` before the existing static argument group, substitute positive values into the argument, and omit the argument for `0`. Add explicit xUnit v3 identity detection, opt-out, and de-duplicated constant append beside the existing xUnit implicit-using behavior. Expose the relevant evaluated properties in the consumer harness, cover default/custom/zero/disabled arguments and xUnit present/absent/opt-out behavior across representative consumption modes, then document both properties.

**Verification:** Consumer evaluation asserts exact arguments and constants; a clean generated xUnit consumer restores, builds, and runs from the packed Test SDK.

## Verification Contract

Run in dependency order:

1. `dotnet restore headless-sdk.slnx`
2. `dotnet build headless-sdk.slnx --configuration Release --no-restore -p:GeneratePackageOnBuild=false --no-incremental -v:minimal -nologo`
3. `dotnet pack headless-sdk.slnx --configuration Release --no-restore --no-build --output ./artifacts/packages-results`
4. `HEADLESS_PACKAGES_DIR="$PWD/artifacts/packages-results" dotnet test tests/Headless.NET.Sdk.Tests.Integrations/Headless.NET.Sdk.Tests.Integrations.csproj --configuration Release --no-restore --no-build`
5. `HEADLESS_PACKAGES_DIR="$PWD/artifacts/packages-results" dotnet test headless-sdk.slnx --configuration Release --no-restore --no-build`
6. Run the repository formatter or formatting gate used by CI, followed by `git diff --check`.

Record command exit codes, test totals, and any platform coverage gap. Hosted Linux, Windows, and macOS CI remains the final cross-platform proof for case and MSBuild evaluation behavior.

## Definition of Done

- R1-R6 and AE1-AE5 are implemented without changing any deferred xUnit behavior.
- Packaged configs, scaffold output, consumer packages, evaluated MTP arguments, and xUnit constants have focused integration coverage.
- The customization table and Test SDK README describe defaults and opt-outs accurately.
- Release build, pack, focused integration tests, full solution tests, formatting, and diff checks pass locally, with hosted matrix validation reported separately if not run in this session.
