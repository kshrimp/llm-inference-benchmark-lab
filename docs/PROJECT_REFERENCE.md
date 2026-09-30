# Serving case study — technical reference

This document defines the serving case study scope, architecture, methodology, cache protocol, data contracts, milestones, report outline, and compatibility checklist.

Implementation milestones remain unstarted unless actual evidence is recorded in reproducible artifacts. Engineering acceptance and paid-run restrictions remain binding.

## Contents

- [Case-study scope](#scope)
- [Architecture](#architecture)
- [Measurement methodology](#methodology)
- [Prefix-cache protocol](#prefix-cache)
- [Data contracts](#data-contracts)
- [Implementation milestones](#milestones)
- [Report and demonstration](#report)
- [Compatibility checklist](#compatibility)

<a id="scope"></a>
## Case-study scope

### Purpose

Prepare for customer-facing ML infrastructure work by demonstrating inference depth, practical systems reasoning, and clear technical communication. SGLang is the primary engine to learn and operate. A focused vLLM comparison is an optional extension after the required project.

### Customer scenario

A hypothetical document-assistant team has interactive short requests, longer document requests, and repeated questions sharing document context. Traffic is uneven. It wants responsive service with bounded GPU cost and understandable overload behavior.

This is a serving workload model, not a claim about a real customer. Before pilots, declare assumed traffic, first-token and completion targets, request-size limits, and acceptable failure rate. Freeze final evaluation targets before collecting final results; do not move targets to fit observations.

### Questions

1. What latency, throughput, and failures does a bounded warm SGLang baseline show?
2. What changes at one higher concurrency with otherwise controlled work?
3. What does one lifecycle or controlled-failure investigation reveal?
4. What limited customer recommendation does that evidence support?

Optional extensions ask about mixed-length interference, prefix reuse, broader overload behavior, or a matched vLLM comparison.

### Boundaries

One small dense text model, fixed tokenizer/weights, one precision, one GPU type, one GPU worker at a time. Use public/synthetic prompts. SGLang owns token scheduling and batching. Limit deployment configuration experiments to a small justified set.

Exclude model-quality rankings, full RAG/agent applications, dashboards, training/RL, custom CUDA, speculative-decoding experiments, multi-GPU execution, routing across replicas, autoscaling tuning, and broad engine/model sweeps. HF can support tokenizer or output sanity checks; nano-vLLM can be reading material. Neither is a required benchmark track.

### Definition of done

A fresh checkout can run CPU validation. A documented bounded GPU command reproduces a small result. Every figure resolves to raw artifacts and a manifest. Failures and missing observations remain visible. The report gives a tested configuration recommendation, its limits, and proposed next production checks. Required evidence covers a warm baseline, one concurrency tradeoff, one lifecycle/failure investigation, and a short customer recommendation. Inconclusive or negative findings are valid; deeper cache studies and vLLM comparisons are optional.

### Effort and budget

Completion is a target, not a guarantee. Carry unfinished required work forward before considering extensions. Estimate each paid stage and enforce resource/time limits before launch. No resources should remain unintentionally active.

<a id="architecture"></a>
## Architecture

### Request path

```text
Saved workload / arrival trace
             |
Separate CPU load generator
             |
Modal HTTP endpoint
             |
SGLang server: tokenize -> queue/schedule -> prefill/decode -> stream
             |
One GPU worker + engine-owned KV cache

Client records + supported server metrics -> validated artifacts -> report
```

Use WSL for development, CPU tests, and orchestration. Place the measurement client on a separate nearby CPU worker; record its region and network path. WSL-to-public-endpoint measurements are a separate deployment-latency scenario.

Modal manages container execution; the serving engine manages token-level scheduling. Configure platform input concurrency and engine request limits explicitly and record both. Do not assume either describes actual active GPU sequences. Do not add an application batcher or custom token scheduler.

### Components

| Component | Responsibility |
| --- | --- |
| Config/workload loader | Validate settings, token counts, seeds, hashes, arrival traces |
| Deployment launcher | Pinned image, readiness, resource limits, worker lifecycle |
| Load generator | Bounded async streaming requests, dispatch lag, deadlines, outcomes |
| Engine interface | Launch settings, capabilities, raw supported metrics |
| Artifact writer | Durable manifests, request/event/resource records, checksums |
| Analysis | Validation, per-class statistics, traceable figures |

For the optional vLLM extension, reuse the SGLang client measurement logic where their pinned APIs permit. Preserve raw engine outputs. Never assume OpenAI-compatible endpoints guarantee identical tokenization, templates, output lengths, or streaming semantics.

### Lifecycle and storage

Validate -> reserve budget -> launch one worker -> readiness/model initialization -> warmup -> establish cache state -> measure -> save and validate artifacts -> terminate resources and reconcile spend.

Cache model files in persistent storage if useful, with revisions recorded. Weight-file cache, container warmth, and engine prefix cache are separate states. Persist partial results before teardown on failure. Readiness means the model can serve, not merely that a process or port exists.

Apply request deadlines, bounded client pending/in-flight work, server limits where supported, and hard worker lifetime limits. Record retry policy; default measured requests to no automatic retries. Cleanup must run on success, exceptions, and client cancellation. Credential configuration is outside artifact capture.

### Timing boundaries

Online TTFT and completion latency use the client's monotonic clock. In-engine queue/scheduler measurements require supported instrumentation with documented semantics. Do not infer queue wait from TTFT. Never subtract unsynchronized server and client timestamps. Offline measurements, if later added, stay in a separate mode.

### Proposed layout

Create directories only when their milestone starts:

```text
src/inference_lab/{config,workloads,schemas,metrics,artifacts,cli}.py
src/inference_lab/serving/{client,capabilities,normalize}.py
src/inference_lab/analysis/{validate,summarize,plots}.py
infra/modal/{sglang,vllm,runner}.py
configs/{smoke,capacity,prefix_cache,lifecycle,comparison}/
tests/{unit,integration}/
results/
reports/
```

These paths describe future code; none of these interfaces is implemented yet. The vLLM adapter and prefix/comparison configs belong to optional extensions, not required scaffolding.

<a id="methodology"></a>
## Measurement methodology

### Controlled setup

Pin model/tokenizer revisions, engine versions, image digests, GPU type, precision, generation parameters, and client placement. Hold request IDs/input IDs and arrival traces constant for paired runs. Validate text/token round trips and any rendered chat templates. Use greedy generation and a verified fixed-output policy for controlled tests; retain actual output counts. If semantics cannot match, qualify or exclude the comparison.

Cross-engine differences include kernels, scheduling, memory management, and defaults. They do not isolate one mechanism. Cache ablations within SGLang are an optional mechanism experiment. The required comparison changes only client concurrency under a documented cache procedure.

### Proposed workloads

| Class | Input tokens | Target output tokens | Purpose |
| --- | ---: | ---: | --- |
| Short | 128 | 128 | Interactive baseline |
| Long input | 2048 | 128 | Prefill pressure |
| Long output | 128 | 512 | Decode occupancy |
| Mixed | 60% short, 20% long input, 20% long output | By class | Interference |
| Shared document | 2048 total, including controlled common prefix | 128 | Prefix reuse |

Only the Short class is required for the baseline/concurrency comparison; the other classes are extension candidates. Lengths include template/special tokens. These are initial design values; verify they fit the chosen model/GPU in M0. Label synthetic prompts and separately sanity-check natural-language outputs. Avoid unintended shared prefixes in control workloads. Shared-document design is specified in the project reference (prefix-cache).

### Sequence

1. Tiny SGLang correctness smoke; no performance claims.
2. Complete CPU measurement contracts and fixtures.
3. Warm short-workload baseline at concurrency one, then one bounded higher-concurrency condition.
4. One cold/warm or controlled-failure investigation, reported separately from steady-state results.
5. Short customer recommendation and reproducible evidence.
6. Optional E1/E2 work only afterward: deeper cache/system experiments or a matched vLLM comparison.

Never run a full Cartesian product by default. One engine worker runs at a time. A client concurrency ceiling must not silently convert an intended open-loop workload to closed-loop: record dispatch lag and unsent/dropped requests.

### Metrics

For successful requests: dispatch `t0`, first generated-content observation `t1`, completion `te`, actual generated tokens `N`:

- TTFT = `t1 - t0` (exclude empty/role-only stream events).
- Completion latency = `te - t0`.
- TPOT = `(te - t1)/(N - 1)` for `N > 1`; otherwise null with reason. This includes client completion overhead.
- Throughput = successful output tokens or successful requests / declared measured duration.
- SLO goodput = successful requests satisfying both predeclared TTFT and completion targets / duration.

Streaming chunks are not necessarily single tokens; chunk intervals must not be labeled inter-token latency. Report per-class p50/p95, counts, failures, timeouts, offered and dispatched rates, and dispatch lag. Failed-request partial output is separate from successful-token throughput.

Full-run duration spans first scheduled arrival to final completion or bounded drain cutoff, including unsent work in offered accounting. Any steady-state window needs explicit bounds and counting rules. Warmup is excluded. In-engine queue wait and cache counters are null unless actually observed. Resource samples identify source, interval, scope, and units; sampled peaks can miss brief maxima. Allocated KV capacity is not occupied cache.

### Repeats and validity

For the required exploratory study, predeclare bounded counts and repeat both concurrency conditions at least twice within budget; show individual repeat results and avoid tail claims unsupported by sample counts. For optional stronger steady-state studies, begin with three repeats and at least 300 completed requests per condition; increase only with a budget and statistical rationale. Per-class p95 requires enough class samples; label small samples exploratory. Do not claim p99 in version one. Save seeds; pair traces across conditions; alternate order; show repeat ranges without calling them confidence intervals.

Cold-start samples have their own count and lifecycle metadata. A few starts are illustrative, not a reliable tail estimate. Record whether weights were cached and whether a fresh worker was verified.

Invalidate corrupted artifacts, inconsistent token accounting, or client bottlenecks that prevent the intended experiment. Keep OOMs, rejections, timeouts, and overload failures as operational outcomes. They must not disappear from summaries. All performance findings remain unknown until measured.

<a id="prefix-cache"></a>
## Prefix-cache protocol

Optional E1 extension after M3. The concepts remain useful study material; this experiment is not required for project completion.

### Question and hypothesis

For repeated questions sharing an identical document prefix, does SGLang prefix caching improve first-token latency and SLO goodput? Reuse may reduce repeated prefill work, while cache pressure and scheduling can limit the benefit. Neither improvement nor its size is assumed.

This replaces the previous nano-vLLM admission-policy experiment. No engine source modification is required.

### Controlled treatment

Use one pinned SGLang version with a verified way to enable/disable prefix caching and clear its state. Confirm exact flag/API behavior in E1 before the ablation. Compare caching enabled versus disabled on identical request IDs, tokens, arrival offsets, output policy, engine budgets, and hardware.

For each setting, test:

- Shared-prefix requests: document tokens first, distinct question suffixes afterward.
- Matched unique-prefix controls: equal input/output lengths and request counts, different document prefixes.

Use a saved document-to-request mapping and verify the exact common token prefix after rendering. Similar text is not sufficient. Report unavoidable common template tokens. A single-document hot set is the MVP; a larger working-set/eviction study is optional after results justify it.

### State protocol

Warm model execution without accidentally priming measured document prefixes. Then clear/reset the prefix cache using a verified mechanism. Record how this was done. If reset is unsupported, use a fresh engine with the same documented warmup protocol.

Measure cold-prefix population separately from subsequent prefix reuse in the same warm container. If a primed-cache condition is reported, record priming requests and their cost outside the steady-state window; also show first-use cost so the recommendation does not hide it. Caching-disabled runs follow equivalent lifecycle/warmup steps.

A cold container, uncached model weights, and an empty prefix cache are distinct conditions. Never label all three simply “cold.”

### Design and evidence

Use a light offered rate and one rate near the baseline capacity knee, identical across paired settings. Freeze settings before final runs. Save seeds and alternate order. Report TTFT and completion p50/p95, TPOT, SLO goodput, throughput, failures, actual work counts, and supported cache/resource counters.

Explain cache-counter definitions from pinned code/docs; leave unavailable counters null. Latency improvement alone does not measure cache hit rate or prove reduced prefill time. Report memory pressure or regressions as readily as improvements.

### Acceptance

CPU fixtures verify token-prefix identity, unique controls, workload determinism, and grouping. A tiny GPU test validates toggle/reset behavior through supported evidence before larger runs. If treatment cannot be verified, stop the ablation and document the blocker. Keep engine comparison claims separate from causal cache claims.

<a id="data-contracts"></a>
## Data contracts

Implement and validate these contracts before publishable runs. All records carry `schema_version: 1`. There are no current result artifacts; old M0 artifacts were removed in the reset.

### Configuration

Required fields: experiment ID; mode (`serving`, `prefix_cache`, `lifecycle`, `overload`); engine; pinned model/tokenizer revisions; dtype; GPU type/count; deployment/client placement; workload hash/seed; generation parameters; engine arguments; platform input concurrency; client concurrency or arrival trace; warmup; measurement window; SLO targets; request and run deadlines; worker/container caps; spend reservation.

Cache fields specify enablement, initial state, reset/prime mechanism, and priming accounting. Lifecycle fields separately describe container state and model-file cache state. Reject placeholders, mutually incompatible load modes, missing pins, and limits inconsistent with one GPU worker. Store requested settings and resolved settings.

### Workloads

JSONL fields: request ID, class, prompt or token-artifact reference, actual input token count, target output count, arrival offset, seed, and document/prefix group where applicable. Save exact tokenizer/template identity, content hash, and token artifacts. Request IDs are unique within a run.

### Manifest

Run/experiment IDs; UTC start/end; execution status (`running`, `completed`, `failed`, `partial`); separate measurement verdict (`pending`, `valid`, `qualified`, `invalid`) and reasons; config and workload hashes; source Git revision and dirty patch/hash; engine/dependency/image pins; model/tokenizer pins; GPU/driver/runtime; placement/network path; resolved launch command; warmup/state protocol; measurement boundaries; capabilities; budget reservation and actual cost if known; artifact checksums; cleanup outcome.

Establish Git provenance before measured runs. Do not dump environment variables, credential files, authorization headers, or secrets. If actual billing is unavailable, record null and separately label any price-based estimate.

### Request records

Run/request IDs, class, document group, attempt ID, status/error/finish reason; scheduled arrival, dispatch, first content, completion offsets; named clock origin and measurement boundary; actual input/output counts and count source; TTFT, TPOT, completion latency; output digest/reference; truncation; nullable in-engine queue/admission measurements with unavailable reasons.

Record client-unsent, rejected, timed-out, failed, incomplete, and successful outcomes. Use one terminal logical outcome per offered request. Retries are disabled by default; if later enabled, retain attempts separately and never count them as independent offered requests. Preserve partial output counts without treating them as successful tokens.

### Other artifacts

```text
results/<experiment_id>/<run_id>/
  manifest.json
  config.resolved.json
  workload.jsonl
  requests.jsonl
  events.jsonl
  resources.jsonl
  summary.json
  validation.json
  engine-raw/
```

Events identify source and clock; resources identify metric, unit, source, sampling interval, and scope. Optional unsupported artifacts may be absent only when the manifest explains why. Retain raw server/client output with secrets redacted.

Summary: offered/dispatched/success/rejected/timeout/failed/unsent/incomplete counts; successful and partial token counts; throughput denominator; SLO targets and goodput; per-class sample counts/quantiles; dispatch lag; memory/cache metrics; missing-metric reasons. Repeats remain separate artifacts.

### Validation

Validate unique IDs, complete offered-outcome reconciliation, dispatch counts, event ordering, finite nonnegative durations, token counts, hashes/checksums, declared cache state, and valid successes with completion timestamps. One-token TPOT is null. Missing first content is not zero TTFT. Persist unfinished requests as incomplete after a drain cutoff. Never subtract offsets from different clocks without an explicit mapping.

<a id="milestones"></a>
## Implementation milestones

Status: reset 2026-09-17. All milestones below are unstarted. Prior compatibility work does not satisfy this plan.

The required path is **M0 → M1 → M2 → M3**. E1 and E2 are optional extensions after M3; neither gates the required recommendation. The earlier M0–M5 sequence is superseded as of 2026-09-27.

### M0 — Bounded SGLang deployment

Inventory WSL tooling and Git provenance. Select one small dense model, GPU type, and precision supported by SGLang; pin model/tokenizer, dependencies, and image. Prior Qwen/L4 choices remain candidates. Verify Modal access with a bounded CPU action before GPU work.

Inspect streaming, actual token accounting, readiness, request limits, and shutdown. Record prefix-cache configuration and a reproducible initial-state/warmup procedure; a cache-toggle ablation and vLLM support are not prerequisites. Implement the minimal smoke harness and persistent budget reservations.

Before paid execution, reconcile usage and the existing $10 smoke-only budget, dated price, all billable ceilings, one-worker enforcement, hard lifetime limits, and cleanup. Run a tiny SGLang correctness smoke and retain raw evidence.

Acceptance: SGLang serves the selected model; exact pins, streaming/output semantics, baseline state procedure, resource limits, and successful/failure cleanup are documented and verified. Smoke output establishes compatibility, not benchmark performance. Stop if compatibility cannot be established within the smoke budget.

### M1 — Trustworthy CPU measurement foundation

Implement the data contracts for the required serving and chosen lifecycle/failure modes, deterministic token-counted workloads, a bounded async client against a fake stream, metric aggregation, and artifact validation. Declare customer assumptions/SLOs before pilots. Extension-only prefix groups and cross-engine adapters are deferred; record inapplicable fields explicitly.

Acceptance: hand-calculated TTFT/TPOT/throughput/goodput fixtures pass; role-only/multi-token chunks, one-token outputs, disconnects, timeouts, unsent requests, partial runs, and dispatch lag are handled. Offered outcomes reconcile. A fresh CPU setup runs checks. Missing metrics are null with reasons.

### M2 — Baseline, one concurrency tradeoff, and one investigation

Use the pinned bounded deployment and separate CPU measurement client. Establish a warm concurrency-one baseline. Compare it with one small higher concurrency, holding model, hardware, input/output policy, cache procedure, and client placement fixed. Retain actual token counts, repeats, failures, and measurement boundaries. Select limits within an established non-smoke experiment budget; the $10 smoke authorization does not cover this stage.

Choose **one** investigation: (a) a small cold-container versus warm-container comparison, separately recording model-file and prefix-cache state; or (b) one controlled timeout/disconnect or bounded burst with partial-output accounting, drain/recovery evidence, and cleanup. A closed-loop baseline does not require a capacity-knee search or open-loop sweep.

Acceptance: artifacts reconcile with raw observations; one small result reproduces; the concurrency comparison explains observed latency/throughput behavior, including an inconclusive result; the chosen investigation has explicit conditions, observations, and limitations. Do not claim maximum capacity or production reliability from this small study.

### M3 — Short customer recommendation and demonstration

Write a short memo covering customer assumptions, the measured baseline and concurrency comparison, the chosen lifecycle/failure finding, recommended next action, cost evidence or labeled estimate, and limitations. Include artifact references and tested reproduction/cleanup steps. Rehearse a five-minute walkthrough.

Acceptance: every claim traces to evidence; observed failures and uncertainty remain visible; CPU checks and a bounded GPU reproduction are documented; recommendations apply only to the tested workload. A blocker/status memo supports learning but does not complete an unfinished engineering milestone.

### E1 — Optional deeper cache and systems experiments

After M3, verify prefix enable/disable/reset/priming semantics and CPU prefix/control fixtures before following the prefix-cache protocol. Mixed-length interference, further load points, or the unchosen lifecycle/failure investigation may be added only for a stated question and established budget.

Acceptance: treatment and state verified, matched work, retained failures, reproducible artifacts, and qualified conclusions. No autoscaling or custom scheduling is introduced.

### E2 — Optional focused vLLM comparison

After M3 (and E1 if comparing prefix reuse), verify pinned vLLM compatibility with the selected model/GPU and run a tiny bounded correctness smoke. Reuse the client/workload pipeline at matched work and common load. If hardware or model must change, rerun the SGLang baseline for that comparison.

Acceptance: tokenization, actual work, defaults, and lifecycle/cache differences reconciled; comparison reproducible and limitations explicit. Cross-engine performance does not isolate one mechanism.

### Progress protocol

Complete M0–M3 in order; optional extensions follow required completion. Update the continuation guide with commands, outcomes, spend, blockers, and the next bounded action. Create implementation directories only when needed. No CLI is promised before implementation. Use scheduled practice time and record remaining work instead of treating the three-week calendar as engineering acceptance.

<a id="report"></a>
## Report and demonstration

Status: outline only; no results for the revised project.

### Customer recommendation memo

Start with the hypothetical customer's traffic and latency requirements. Recommend a tested serving configuration, tested request/load range, and known failure behavior. Identify untested overload behavior as a production question. Explain latency/cost tradeoffs and what still needs validation before production. Do not imply a small model benchmark establishes application quality or larger-model capacity.

### Required evidence note

Document the pinned setup, timing boundaries, baseline and higher-concurrency results, chosen lifecycle/failure investigation, artifact IDs, and reproduction/cleanup commands alongside the short memo. State sample counts and limits. Additional report sections below apply only when those extensions were actually performed.

### Optional expanded technical report

1. Customer assumptions, success criteria, and architecture diagram.
2. Pinned setup, workload construction, client placement, timing boundaries, resource limits.
3. SGLang capacity: achieved load, per-class TTFT/completion p50/p95, SLO goodput, failures.
4. Mixed-length interference: short-class outcomes and competing explanations.
5. Prefix reuse: treatment/state protocol, matched controls, first-use/priming cost, supported counters, memory observations.
6. Lifecycle: cold versus warm with weight-cache state and initialization boundaries explicit.
7. Overload: scheduled versus dispatched work, rejection/timeout behavior, drain and recovery.
8. Selected vLLM comparison with equivalent work and known differences disclosed.
9. Recommendation, cost accounting, limits, and next customer questions.
10. Reproduction commands, raw artifact locations, cleanup, and expected resource needs.

### Five-minute demo

Customer problem (30s), architecture/request path (45s), baseline and concurrency evidence (75s), chosen lifecycle/failure investigation (60s), recommendation and limitations (60s), reproduction/next check (30s).

### Claim checks

Every figure resolves to configs, run IDs and raw records. Every tail has sample counts and boundaries. Show repeat variability and failures. Distinguish measured bills from price-based estimates; include idle, startup, client and storage costs where relevant. Never call chunk intervals token latency, TTFT prefill time, or reserved KV memory cache occupancy. Negative findings count. Make public claims only after the deliverables exist.

<a id="compatibility"></a>
## Compatibility checklist

Status: unstarted after the 2026-09-17 scope reset. This is a checklist, not evidence of compatibility. Previous nano-vLLM findings and dependency pins do not apply to the revised plan.

### Evidence to collect

| Area | Required check | Current outcome |
| --- | --- | --- |
| WSL | Python/tool inventory; existing environment suitability | Not checked for revised plan |
| Provenance | Git source revision and dirty-state capture | Pending |
| Modal | Bounded authenticated CPU execution | Unverified; prior session timed out |
| Model | Immutable weights/tokenizer supported by SGLang; vLLM check deferred to E2 | Unselected |
| GPU/precision | SGLang support, memory headroom, price | Unselected |
| Dependencies | Pinned engine versions, locked images, runtime imports | Unselected |
| Output | Natural-text sanity, greedy/fixed-length semantics, actual token counts | Unverified |
| Streaming | First-content event and completion/error behavior | Unverified |
| Prefix cache | Required: documented baseline configuration/state procedure. E1 only: ablation toggle/reset/priming/counters | Unverified |
| Limits | Platform concurrency versus engine limits, client caps, deadlines | Unverified |
| Lifecycle | Readiness, persistent weight cache, cold/warm identification | Unverified |
| Cleanup | Resource termination after success/failure/cancellation | Unverified |
| Budget | Price, prior usage, persistent reservations, worker/time caps | Pending |

### Smoke protocol

Start with CPU config/tokenization/stream-parsing checks. Before remote GPU execution, record immutable versions, identical tiny natural-text fixtures, bounded output lengths, and maximum billable resource duration. Account for possible startup/compilation time and provider timeout semantics.

Launch SGLang, verify readiness and a few requests, record baseline cache configuration/state procedure, collect evidence, and terminate. The equivalent vLLM smoke is deferred to E2; cache-ablation verification is deferred to E1. Keep one GPU worker at a time. Save failures and actual output counts. These are compatibility checks, not benchmark results.

The old Qwen3-0.6B/L4 combination can be reconsidered but is not accepted by default. Current Modal examples may use much larger models/hardware; copy architectural patterns only after adapting resource limits and verifying API versions.

### Evidence entry template

Date; check; exact command/config; pinned source references; observed result; artifact location/checksum; limitation; resource usage/cleanup; remaining budget; next step.

Mark M0 complete only when all acceptance conditions in the project reference (milestones) have actual evidence. Unknown timings/counters stay unknown.
