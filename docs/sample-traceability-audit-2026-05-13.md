# Ripaira POC — Traceability & Semantic Gap Audit

**Date:** 2026-05-13
**Branch:** `feature/POC`
**Scope:** `docs/functional-requirements.md` ↔ `api/` (.NET) ↔ `mobile/` (Expo) ↔ tests
**Method:** Elicitor → Architect → Auditor sub-agent pipeline

---

## 1. Coordination Log

| Sub-agent | Source(s) | Outcome |
| --- | --- | --- |
| Elicitor | `docs/functional-requirements.md` | 50 atomic requirements extracted (44 FR across 10 areas + 5 NFR + 2 use cases). No other docs in `docs/`. |
| Architect (SDS scan) | `docs/**/*.md` | **No SDS / architecture doc exists** — only the FRD. Tracing collapses to FRD -> code -> test. |
| Architect (code) | `api/**/*.cs`, `mobile/app/`, `mobile/lib/`, `mobile/components/` | 4 .NET projects, 1 minimal API, 2 middlewares, 2 agents + orchestrator, 1 image service, 3 Expo screens, 3 first-party components, 4 libs. |
| Architect (tests) | repo scan | 1 xUnit suite: `api/Ripaira.Tests/AssessmentsEndpointTests.cs` (10 tests). **Zero mobile tests** (all `__tests__` matches are inside `node_modules`). |
| Auditor | cross-walk | Gaps and drift below. |

## 2. Executive Summary

- **Requirements covered by code:** 44 / 44 functional (100%); 2 / 5 NFRs partially (40%).
- **Requirements covered by tests:** 11 / 44 functional (~25%); all backend-side. **0% mobile test coverage.**
- **Documentation gaps:** No SDS, no NFR/architecture doc — FRD section 4 explicitly defers them.
- **Hallucinated features:** 1 confirmed (`/test` dev screen — not in FRD), 1 borderline (in-process stub orchestrator — test infra, acceptable).
- **Critical semantic drifts:** 2 (FR-AG-7 partial-result handling; FR-SUB-5 base64 size limit is documented in code but encoded against base64 length, not decoded binary).

## 3. Architectural Unit Inventory

### Backend (`api/`)

| Unit | Type | File |
| --- | --- | --- |
| `Program` | Composition root + DI + middleware pipeline + `/healthz` map | `api/Ripaira.Api/Program.cs:1` |
| `DeviceIdMiddleware` | Header validation | `api/Ripaira.Api/Middleware/DeviceIdMiddleware.cs:3` |
| `RateLimitMiddleware` | Per-device daily counter via `IMemoryCache` | `api/Ripaira.Api/Middleware/RateLimitMiddleware.cs:5` |
| `AssessmentsEndpoint.MapAssessments` | `POST /api/assessments` | `api/Ripaira.Api/Endpoints/AssessmentsEndpoint.cs:13` |
| `ImageProcessingService` | Resize to ≤1024 px, JPEG q80 | `api/Ripaira.Api/Services/ImageProcessingService.cs:7` |
| `AssessmentOptions` | `Assessments:MaxImages` binding | `api/Ripaira.Api/Configuration/AssessmentOptions.cs:3` |
| `AssessmentOrchestrator` | Planning → pricing hints → estimating; builds response | `api/Ripaira.Agents/AssessmentOrchestrator.cs:8` |
| `StubAssessmentOrchestrator` | Phase 1 placeholder; also reused by tests | `api/Ripaira.Agents/StubAssessmentOrchestrator.cs:7` |
| `PlanningAgent` | LLM call, structured `PlanningResult` | `api/Ripaira.Agents/Agents/PlanningAgent.cs:7` |
| `EstimatingAgent` | LLM call, structured `EstimatingResult` | `api/Ripaira.Agents/Agents/EstimatingAgent.cs:9` |
| `IPricingSource` / `NullPricingSource` | Pluggable pricing hints | `api/Ripaira.Agents/Pricing/IPricingSource.cs:8`, `NullPricingSource.cs:5` |
| `AgentPrompts` (+ `Prompts/planning.md`, `Prompts/estimating.md`) | Embedded resource loader | `api/Ripaira.Agents/AgentPrompts.cs:5` |
| Domain contracts | `AssessmentRequest/Response`, `ProblemStatement`, `SolutionOption`, `Estimate`, enums | `api/Ripaira.Domain/Contracts/*.cs` |

### Mobile (`mobile/`)

| Unit | Type | File |
| --- | --- | --- |
| `RootLayout` | Expo Router stack | `mobile/app/_layout.tsx:4` |
| `CaptureScreen` (index) | Capture/submit flow | `mobile/app/index.tsx:24` |
| `ResultScreen` | Result rendering | `mobile/app/result.tsx:15` |
| `TestScreen` | **Dev-only API connectivity probe** | `mobile/app/test.tsx:14` |
| `PhotoPicker` | Camera + library picker, ≤3 images, remove-thumbnail | `mobile/components/PhotoPicker.tsx:16` |
| `CategoryPicker` | Horizontal pill picker over `CATEGORIES` | `mobile/components/CategoryPicker.tsx:10` |
| `EstimateCard` | Renders option + estimate | `mobile/components/EstimateCard.tsx:19` |
| `api.ts` (`postAssessment`, `getHealth`, `ApiError`) | Fetch wrapper, base-URL env, error type | `mobile/lib/api.ts:5` |
| `device-id.ts` (`getOrCreateDeviceId`) | UUID generation + `SecureStore` persistence | `mobile/lib/device-id.ts:15` |
| `assessment-store.ts` | In-memory singleton for last response | `mobile/lib/assessment-store.ts:7` |
| `types.ts` | Hand-mirrored DTOs from `Ripaira.Domain.Contracts` | `mobile/lib/types.ts:4` |
| `categories.ts` | UI category list | `mobile/lib/categories.ts:3` |

### Tests

| Test | File:line | Requirements verified |
| --- | --- | --- |
| `Healthz_does_not_require_device_id` | `api/Ripaira.Tests/AssessmentsEndpointTests.cs:44` | FR-HC-1, FR-HC-2 |
| `Returns_stub_assessment_for_valid_request` | `:52` | FR-AG-4, FR-AG-5, FR-RES-1, FR-RES-6 (AUD), FR-SUB-7 (camelCase round-trip) |
| `Returns_400_when_device_id_missing` | `:69` | FR-DEV-3 |
| `Returns_400_when_device_id_not_a_uuid` | `:77` | FR-DEV-3 |
| `Returns_400_for_invalid_base64` | `:85` | FR-SUB-5 |
| `Returns_400_when_images_list_is_empty` | `:98` | FR-SUB-3 |
| `Accepts_up_to_three_images` | `:108` | FR-SUB-4 (positive), FR-IMG-1 (server side) |
| `Returns_400_when_too_many_images` | `:116` | FR-SUB-4 (negative) |
| `Returns_400_for_empty_description` | `:124` | FR-SUB-1 |
| `Eleventh_request_in_a_day_is_rate_limited` | `:134` | FR-RL-1, FR-RL-2 |

## 4. Traceability Matrix

Status legend: `COV` = code + test, `PART-T` = code present, no test, `PART-C` = partial code, `ORPHAN` = no code/test, `HALLU` = code without requirement, `DRIFT` = mapped but intent diverges. Confidence: H/M/L.

| Req ID | Summary | Code Artifact(s) | Test(s) | Status | Conf |
| --- | --- | --- | --- | --- | --- |
| **FR-DEV-1** | Generate + persist UUID on first launch | `mobile/lib/device-id.ts:15` (`getOrCreateDeviceId`, `SecureStore`) | — | PART-T | H |
| **FR-DEV-2** | Send `X-Device-Id` on every API request | `mobile/lib/api.ts:37` | — (mobile); indirectly via API 400 path | PART-T | H |
| **FR-DEV-3** | API rejects missing/invalid `X-Device-Id` with 400 | `DeviceIdMiddleware.cs:8` | `:69`, `:77` | COV | H |
| **FR-DEV-4** | Device ID not tied to PII | `device-id.ts` (no PII); `RateLimitMiddleware.cs:23` (key by GUID only) | — | PART-T | M |
| **FR-RL-1** | 10 successful `/api/*`/UTC-day per device | `RateLimitMiddleware.cs:8,23` (`DailyLimit=10`, key includes `yyyyMMdd` UTC) | `:134` | COV | H |
| **FR-RL-2** | 429 + plain-text body when exceeded | `RateLimitMiddleware.cs:35` | `:134` (status only; body text not asserted) | COV | M |
| **FR-RL-3** | Mobile surfaces 429 as recoverable alert (not crash) | `mobile/app/index.tsx:59` (`ApiError` -> `Alert.alert`) | — | PART-T | M |
| **FR-RL-4** | UTC-day reset; in-memory only | `RateLimitMiddleware.cs:23,27` (`AbsoluteExpirationRelativeToNow=25h`) | — (only positive 10/11 test) | PART-T | M |
| **FR-HC-1** | `GET /healthz` -> 200 `{status:"ok"}` | `Program.cs:63` | `:44` (status only; body not asserted) | COV | M |
| **FR-HC-2** | `/healthz` skips device-id + rate limit | `DeviceIdMiddleware.cs:10`, `RateLimitMiddleware.cs:11` (both gate on `/api`) | `:44` | COV | H |
| **FR-IMG-1** | 1–3 images per assessment (mobile) | `mobile/app/index.tsx:22` (`MAX_IMAGES=3`), `PhotoPicker.tsx:17` | — | PART-T | H |
| **FR-IMG-2** | Camera or library | `PhotoPicker.tsx:34,19` | — | PART-T | H |
| **FR-IMG-3** | Remove a selected image | `PhotoPicker.tsx:52,66` | — | PART-T | H |
| **FR-IMG-4** | Submit disabled until image + desc + category | `mobile/app/index.tsx:37,106` (`canSubmit` gating) | — | PART-T | H |
| **FR-SUB-1** | Non-empty description -> else 400 | `AssessmentsEndpoint.cs:26` | `:124` | COV | H |
| **FR-SUB-2** | Category must be defined enum | `AssessmentsEndpoint.cs:29` (`Enum.IsDefined`) | — (no explicit "unknown category" test) | PART-T | M |
| **FR-SUB-3** | ≥1 image required | `AssessmentsEndpoint.cs:32` | `:98` | COV | H |
| **FR-SUB-4** | ≤ `Assessments:MaxImages` (default 3) | `AssessmentsEndpoint.cs:35`, `AssessmentOptions.cs:7` | `:108`, `:116` | COV | H |
| **FR-SUB-5** | Each `base64` required, well-formed, ≤~10 MB decoded | `AssessmentsEndpoint.cs:11` (`MaxBase64LengthPerImage=14_000_000`), `:44–58` | `:85` (invalid base64 only; size limit untested) | DRIFT / PART-T | M |
| **FR-SUB-6** | Reject non-image content with 400 | `AssessmentsEndpoint.cs:65` (catches ImageSharp `UnknownImageFormat`/`InvalidImageContent`) | — (test uses `!!!not-base64!!!`, fails earlier at base64 decode) | PART-T | M |
| **FR-SUB-7** | JSON in/out, camelCase, string enums | `Program.cs:36` (`ConfigureHttpJsonOptions`), `AgentJsonOptions.cs:8` | `:52` (round-trip implicitly relies on it) | COV | H |
| **FR-IMG-PROC-1** | Resize so neither edge > 1024 px | `ImageProcessingService.cs:9,17–24` (`MaxEdge=1024`, `ResizeMode.Max`) | — | PART-T | H |
| **FR-IMG-PROC-2** | Re-encode JPEG q80 | `ImageProcessingService.cs:10,27` (`JpegQuality=80`) | — | PART-T | H |
| **FR-IMG-PROC-3** | Pre-processing failures -> 400, not 500 | `AssessmentsEndpoint.cs:65` | — | PART-T | M |
| **FR-AG-1** | Invoke Planning with category + description + images | `AssessmentOrchestrator.cs:28`, `PlanningAgent.cs:20–40` | — (real path mocked out by test factory) | PART-T | H |
| **FR-AG-2** | Planning returns 1 `ProblemStatement` + ≥1 `SolutionOption` | `AgentResults.cs:7`, `Prompts/planning.md` (not inspected) | — | PART-T | M |
| **FR-AG-3** | Estimating produces an `Estimate` per option, matched by `optionId` | `EstimatingAgent.cs:22`, `AssessmentOrchestrator.cs:32` | — (no integrity check that lengths match) | PART-T | M |
| **FR-AG-4** | Aggregate into single `AssessmentResponse` | `AssessmentOrchestrator.cs:34–41` | `:52` | COV | H |
| **FR-AG-5** | Response carries GUID `assessmentId` + UTC `generatedAt` | `AssessmentOrchestrator.cs:23,40` | `:52` (asserts non-empty GUID; `generatedAt` not asserted) | COV | M |
| **FR-AG-6** | Fixed indicative disclaimer | `AssessmentOrchestrator.cs:14` | `:52` (asserts substring on the **stub** disclaimer "STUB", not the real one) | DRIFT | M |
| **FR-AG-7** | On agent failure -> 5xx; no partial results | No explicit try/catch in `AssessmentOrchestrator`; `PlanningAgent.cs:60` throws `InvalidOperationException` -> ASP.NET returns 500 by default. **No explicit fail-closed guarantee** in code. | — | DRIFT | L |
| **FR-AG-8** | Pluggable `IPricingSource`; POC null source | `IPricingSource.cs:8`, `NullPricingSource.cs:5`, `Program.cs:19` | — | PART-T | H |
| **FR-RES-1** | Every option has matching estimate | Relies on LLM in `EstimatingAgent` honouring `Prompts/estimating.md`; no server-side reconciliation | — (stub satisfies it trivially) | DRIFT | L |
| **FR-RES-2** | `severity` enum | `ProblemStatement.cs:9` (`Severity` enum) + `Program.cs:39` (camelCase JSON enum) | — | PART-T | H |
| **FR-RES-3** | `complexity` enum | `SolutionOption.cs:10` | — | PART-T | H |
| **FR-RES-4** | `confidence ∈ [0,1]` | Typed as `double` only; **no range validation** | — | DRIFT (no enforcement) | L |
| **FR-RES-5** | `{ low, high }` with `low ≤ high` | `Estimate.cs:3` (`CostRange` shape); **no `low ≤ high` enforcement** | — | DRIFT (shape only) | M |
| **FR-RES-6** | `currency` ISO-4217-ish, default AUD | `StubAssessmentOrchestrator.cs:37` (AUD); real path relies on LLM output; **no server-side default/validation** | `:64` (asserts `AUD` on stub) | DRIFT | M |
| **FR-UI-1** | Issue title + severity badge + confidence % | `result.tsx:39,43,45`, `SEVERITY_COLOR` | — | PART-T | H |
| **FR-UI-2** | Each option: name, description, complexity, cost range | `EstimateCard.tsx:23,29,25,41` | — | PART-T | H |
| **FR-UI-3** | Disclaimer prominent and distinct | `result.tsx:59` (`styles.disclaimer` yellow card) | — | PART-T | H |
| **FR-UI-4** | "New assessment" clears state + returns to capture | `result.tsx:29,63`, `assessment-store.ts:14` | — | PART-T | H |
| **FR-UI-5** | No result -> redirect to capture | `result.tsx:18` (`router.replace('/')`) | — | PART-T | H |
| **FR-ERR-1** | Surface non-2xx as alert with status+body | `index.tsx:59`, `api.ts:21,45`, `ApiError` | — | PART-T | H |
| **FR-ERR-2** | Submit re-enabled after failure | `index.tsx:67` (`finally { setSubmitting(false) }`) | — | PART-T | H |
| **FR-ERR-3** | No auto-retry | No retry code path exists (negative requirement satisfied by absence) | — | PART-T | M |
| **FR-CFG-1** | Read `OpenAI:ApiKey`; lazy-throw if missing | `Program.cs:25` (throws inside factory only when `IChatClient` is resolved) | — (the test factory bypasses this by replacing the orchestrator) | PART-T | H |
| **FR-CFG-2** | Read `OpenAI:Model` (code default `gpt-4o`, appsettings `gpt-4.1-mini`) | `Program.cs:28`, `appsettings.json:13` | — | PART-T | H |
| **FR-CFG-3** | Read `Assessments:MaxImages` (default 3) | `Program.cs:17`, `AssessmentOptions.cs:7`, `appsettings.json:9` | `:108`, `:116` (default-value path only) | COV | H |
| **FR-CFG-4** | Mobile reads `EXPO_PUBLIC_API_BASE_URL`, falls back to `localhost:5262` | `mobile/lib/api.ts:3,11` | — | PART-T | H |
| **NFR-PERF-1** | E2E 10–30 s; UI shows progress indicator | `index.tsx:109` (`ActivityIndicator` + "Analysing… (10–30s)") | — | PART-T | M |
| **NFR-SEC-1** | POC open CORS, must tighten | `Program.cs:44–50` (`AllowAnyOrigin/Header/Method` with explicit POC comment) | — | PART-T (intentional POC) | H |
| **NFR-SEC-2** | Secrets via user-secrets/env, never checked in | `Program.cs:25` (no inline key); `appsettings.json` carries only `Model`, no key | — | PART-T | H |
| **NFR-LOG-1** | Log assessment IDs, image counts, **LLM token usage** | `AssessmentOrchestrator.cs:24` (id + count), `PlanningAgent.cs:54`, `EstimatingAgent.cs:51` (token usage) | — | PART-T | H |
| **NFR-AVAIL-1** | Rate-limit + image processing fail closed | `RateLimitMiddleware.cs:35`, `AssessmentsEndpoint.cs:65` (both return error, don't silently pass) | — | PART-T | H |
| **UC-1** | End-to-end capture -> result | All capture path artifacts above | partial (`:52` exercises happy path with stub) | PART-T | H |
| **UC-2** | New assessment after viewing result | `result.tsx:29,63`, `assessment-store.ts:14` | — | PART-T | H |

## 5. Semantic Gap Analysis

### 5.1 Orphan Requirements (FRD entries with no code/test backing)

None — every FRD requirement maps to at least one code artifact. (This is the upside of an FRD authored alongside the POC code rather than ahead of it.)

### 5.2 Hallucinated Features (code/tests with no requirement backing)

| Artifact | Location | Verdict |
| --- | --- | --- |
| `TestScreen` and `/test` route | `mobile/app/test.tsx:14`, registered in `_layout.tsx:16`, linked from `index.tsx:119` as "Dev: API test" | **HALLUCINATED** for the user-facing product. The FRD does not mention a connectivity-probe screen. It is honest dev plumbing (uses `getHealth`), but it ships in the mobile binary and is reachable from the capture screen. Recommendation: gate behind `__DEV__` or remove before Phase 4. |
| `getHealth()` mobile helper | `mobile/lib/api.ts:15` | **Implicit**, supports `TestScreen` only. Same disposition as above. |
| `StubAssessmentOrchestrator` (and the test factory that swaps it in) | `api/Ripaira.Agents/StubAssessmentOrchestrator.cs:7`, `RipairaWebAppFactory.cs:11` | Acceptable test infra, but **the production DI registers `AssessmentOrchestrator` only** (`Program.cs:34`), so the stub never reaches prod. Keep, but rename `StubAssessmentOrchestrator` to make its test-only intent explicit, or move it into `Ripaira.Tests`. |
| Expo template scaffolding (`themed-text.tsx`, `themed-view.tsx`, `parallax-scroll-view.tsx`, `hello-wave.tsx`, `external-link.tsx`, `haptic-tab.tsx`, `components/ui/*`) | `mobile/components/` | **Unused dead code** from Expo create-template. None are imported by `app/`. Cosmetically hallucinated, but harmless — recommend prune. |
| `obj/` build artefacts under each `Ripaira.*` project | various | Build output; ignore. |

### 5.3 Semantic Drift / Misalignments (mapped but intent diverges)

1. **FR-SUB-5 — base64 size limit is measured wrong.** FRD: "decode to no more than ~10 MB binary." Code: `MaxBase64LengthPerImage = 14_000_000` characters (`AssessmentsEndpoint.cs:11`). Base64 is ~4/3 the size of decoded bytes, so 14M chars ≈ 10.5 MB decoded — close, but the *check happens on the base64 string length* (`AssessmentsEndpoint.cs:47`), not on `sourceBytes.Length` after decode. Result: an image whose decoded size is just over 10 MB can still slip through if its base64 happens to be slightly compressible (it won't be), or — more practically — the comment in the FRD doesn't match the actual decision boundary. **Recommendation:** check `sourceBytes.Length > 10 * 1024 * 1024` after `Convert.FromBase64String`.

2. **FR-AG-6 — disclaimer tested on the wrong path.** The real disclaimer is the lengthy "Indicative only…" string in `AssessmentOrchestrator.cs:14`. The only test of disclaimer behaviour asserts `Assert.Contains("STUB", body.Disclaimer)` (`AssessmentsEndpointTests.cs:65`), which only ever runs against `StubAssessmentOrchestrator`. There is **no test that the production disclaimer is present** in any response — even a unit test of `AssessmentOrchestrator` would catch this.

3. **FR-AG-7 — "partial results SHALL NOT be returned" is not enforced.** `AssessmentOrchestrator.AssessAsync` (`api/Ripaira.Agents/AssessmentOrchestrator.cs:18`) has no try/catch. If `PlanningAgent` succeeds but `EstimatingAgent` throws, the exception propagates and the framework returns 500 (good), but if the LLM returns a `PlanningResult` with 3 options and `EstimatingResult` with only 2 estimates (which `FR-RES-1` forbids), the response is happily aggregated and returned — that **is** a partial result reaching the client.

4. **FR-RES-1 / FR-RES-4 / FR-RES-5 / FR-RES-6 — response invariants are entrusted to the LLM.** None of these are enforced server-side:
   - No check that `Estimates.Count == Options.Count` or that every `optionId` matches a known `Id` (FR-RES-1).
   - `confidence` typed only as `double`; no `[0,1]` clamp/validate (FR-RES-4).
   - `CostRange(decimal Low, decimal High)` has no `low ≤ high` invariant; record could be constructed with `low=100, high=50` (FR-RES-5).
   - `currency` is a free-form `string`; no default fallback to "AUD" if the LLM returns null/empty (FR-RES-6).

   The agent prompts (`Prompts/planning.md`, `Prompts/estimating.md`) are the only line of defence and were not inspected — they may or may not instruct the LLM correctly, but **prompts are not validation**.

5. **FR-RL-2 body text not asserted.** Test `:134` asserts status code 429 but not the "plain-text body indicating the limit" wording. Low-risk drift.

6. **FR-HC-1 body shape not asserted.** Test `:44` asserts 200 only; the `{"status":"ok"}` body is implicit. Low-risk.

7. **FR-AG-5 timestamp not asserted.** `generatedAt` is set (`AssessmentOrchestrator.cs:40` and `StubAssessmentOrchestrator.cs:42`) but no test asserts it is present, non-default, or UTC.

8. **FR-DEV-1 UUID is not RFC 4122 v4 cryptographically.** `mobile/lib/device-id.ts:8` uses `Math.random`. The code comment explicitly acknowledges this. Strictly the FRD only says "UUID device identifier"; this is acceptable for the POC, but worth surfacing if FR-DEV-4 ever evolves into a real identity claim.

### 5.4 Missing Test Coverage for Covered Requirements

- All mobile-side requirements (FR-DEV-1/2, FR-IMG-1..4, FR-UI-1..5, FR-ERR-1..3, FR-CFG-4, FR-RL-3) — **0% mobile test coverage**, no Jest/RNTL/Detox config in `mobile/`.
- `FR-IMG-PROC-1/2/3` — `ImageProcessingService` has no direct unit tests; resize/quality behaviour is not asserted, only that valid PNG round-trips through the endpoint.
- `FR-SUB-2` — no test for an unknown `category` value.
- `FR-SUB-5` (size limit) — no test for a base64 payload exceeding the cap.
- `FR-SUB-6` — the "non-image content" test (`:85`) trips the base64 decoder before reaching the image-format check, so the `UnknownImageFormatException` branch (`AssessmentsEndpoint.cs:65`) is never exercised.
- `FR-AG-1..3, FR-AG-5..7` — the test factory swaps out the real orchestrator, so the actual planning -> estimating composition is never exercised in CI. Consider a fake `IChatClient` rather than a fake `IAssessmentOrchestrator`.
- `FR-AG-6` — production disclaimer string is untested (see Drift #2).
- `FR-RES-*` — no contract/schema tests; no `AssessmentOrchestrator` unit tests.
- `FR-RL-4` — no test for UTC-day rollover behaviour (would require an injectable clock or `IMemoryCache` time abstraction).
- `FR-CFG-1` — no test for the "fails to start only when chat client is first resolved" behaviour.
- All NFRs — none have automated verification.

## 6. Risk-Ranked Recommendations

| # | Severity | Recommendation | Cost |
| --- | --- | --- | --- |
| **1** | **High** | Validate the `AssessmentResponse` server-side before returning it: `Estimates.Count == Options.Count`, each `optionId ∈ Options.Id`, `confidence ∈ [0,1]`, `low ≤ high` on every `CostRange`, `currency` defaults to "AUD". Failure -> 502 ("upstream model returned malformed result") not a partial 200. Fixes the FR-AG-7 / FR-RES-1/4/5/6 drift in one stroke. | ~1 day |
| **2** | **High** | Fix FR-SUB-5: enforce the 10 MB cap on **decoded** bytes (`sourceBytes.Length`), not on base64-string length. Add a unit test with a >10 MB payload. | ~1 hr |
| **3** | **High** | Cover the production disclaimer (FR-AG-6) with a test that runs `AssessmentOrchestrator` against a fake `IChatClient`, **not** against the stub. Refactor `RipairaWebAppFactory` to swap `IChatClient` instead of `IAssessmentOrchestrator` so all agent integration paths become testable. | ~½ day |
| **4** | **Medium** | Stand up a minimal mobile test stack (Jest + React Native Testing Library) and cover FR-IMG-4 (submit gating), FR-UI-5 (no-result redirect), FR-ERR-1/2 (error path), FR-RL-3 (429 path) at minimum. Phase 4 will need this regardless. | ~1 day |
| **5** | **Medium** | Gate `TestScreen` + the "Dev: API test" link behind `if (__DEV__)`, or remove from the app entry navigator and reach it via a Metro-only route. It is the one clear hallucinated production feature. | ~30 min |
| **6** | **Medium** | Add unit tests for `ImageProcessingService` (FR-IMG-PROC-1/2): assert that a 4096×4096 input becomes ≤1024 px on the long edge and that the output is JPEG. | ~½ day |
| **7** | **Medium** | Author an SDS / architecture doc (or at minimum an ADR set) so the next audit has a middle layer to trace through. The FRD itself defers this — Phase 4 is the cheapest time to do it. | ~½ day |
| **8** | **Low** | Add negative tests for FR-SUB-2 (unknown category), FR-SUB-6 (non-image bytes that *are* valid base64, e.g. a tiny text file b64-encoded), and FR-RL-2 body text. | ~2 hr |
| **9** | **Low** | Rename or relocate `StubAssessmentOrchestrator` so its test-only intent is unambiguous (move under `Ripaira.Tests` or rename to `FakeAssessmentOrchestrator` + an `internal` accessor). | ~30 min |
| **10** | **Low** | Prune Expo template leftovers in `mobile/components/` (`themed-*.tsx`, `parallax-scroll-view.tsx`, `hello-wave.tsx`, `external-link.tsx`, `haptic-tab.tsx`, `components/ui/*`). | ~30 min |
| **11** | **Low** | Document FR-DEV-1's `Math.random`-based UUID as a known POC limitation, with a tracking issue to swap to `expo-crypto.randomUUID()` before any non-POC release. | ~15 min |

## 7. Files of Note (for quick navigation)

- `docs/functional-requirements.md` — the sole spec
- `api/Ripaira.Api/Program.cs` — DI / middleware / routing
- `api/Ripaira.Api/Endpoints/AssessmentsEndpoint.cs` — primary validation surface
- `api/Ripaira.Api/Middleware/DeviceIdMiddleware.cs`, `RateLimitMiddleware.cs`
- `api/Ripaira.Api/Services/ImageProcessingService.cs`
- `api/Ripaira.Agents/AssessmentOrchestrator.cs` — response assembly + the line where invariant validation belongs
- `api/Ripaira.Agents/Agents/PlanningAgent.cs`, `EstimatingAgent.cs`
- `api/Ripaira.Domain/Contracts/` — DTOs (where `low ≤ high` / `confidence ∈ [0,1]` invariants could live)
- `api/Ripaira.Tests/AssessmentsEndpointTests.cs` — the only test suite
- `mobile/app/index.tsx` — capture flow / submit gating
- `mobile/app/result.tsx` — result rendering
- `mobile/app/test.tsx` — dev probe (recommend gating)
- `mobile/components/PhotoPicker.tsx`, `CategoryPicker.tsx`, `EstimateCard.tsx`
- `mobile/lib/api.ts`, `device-id.ts`, `assessment-store.ts`, `types.ts`
