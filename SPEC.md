# Breakyard product and engineering specification

**Status:** Design phase · **First platform:** Linux · **Executable:** `breakyard` · **License:** [MIT](LICENSE)

Breakyard lets a developer run a controlled incident against a disposable local backend, see what happened to its users, change the application, and compare the evidence. The first question it should answer is concrete: **when a URL shortener’s lookup service slows down, can it still return the right destination within the request deadline?**

This document specifies the first release and records the broader product design. The [roadmap](ROADMAP.md) sets the delivery order. The [experiment design](docs/design.md) follows one run through the proposed implementation. Commands, data examples, and scenarios below describe planned behavior.

| To understand… | Read |
|---|---|
| Who would use Breakyard and what ships first | [Audience and scope](#audience-and-scope) |
| What a developer would do with it | [Reference experiment](#reference-experiment) · [Command-line interface](#command-line-interface) |
| How the experiment is represented and executed | [Configuration and scenario model](#configuration-and-scenario-model) · [Architecture and lifecycle](#architecture-and-lifecycle) |
| What the numbers mean and how to interpret them | [Workload and measurement](#workload-and-measurement) · [Evidence and reports](#evidence-and-reports) |
| What protects the developer’s machine and data | [Isolation and recovery](#isolation-and-recovery) |
| What establishes release quality and what could come next | [Verification and release gates](#verification-and-release-gates) · [Scenario catalog](#scenario-catalog) · [Distribution and future decisions](#distribution-and-future-decisions) |

## Audience and scope

The first user is an individual developer with a Linux machine and a local HTTP service. A student can investigate how a deadline works, a maintainer can compare revisions, and an educator can share an experiment with a known fixture. Each needs the workload, the expected behavior, and enough evidence to explain the result on their machine.

| Priority | Planned capabilities |
|---|---|
| P0 · First public release | A foreground Go CLI, one disposable demo with fragile and improved variants, bounded HTTP requests, a temporary lookup delay, terminal progress, verified cleanup, local JSON history, Markdown reports, and comparable before/after runs. An explicitly configured local project can use the measurement workflow. |
| P1 · Richer local experiments | User-confirmed Compose discovery and topology, isolated dependency adapters, additional workload shapes, reusable scenarios, business assertions, optional dependency metrics, CI regression checks, and interactive run history. Adoption determines their order. |
| P2 · Research | Additional protocols, replica and coordination experiments, distributed workers, optional web and team views, privacy-preserving replay, hosted labs, and evidence summaries. Each requires a separate design and demonstrated demand. |

Early releases target authorized, disposable environments. A developer can run the core workflow offline after installation, with ordinary user permissions. Production targeting, vulnerability scanning, automatic infrastructure changes, and production-readiness certification fall outside the product scope. A repeatable schedule lets someone repeat an experiment. OS scheduling, shared resources, and dependency state can still change the observed result.

## Reference experiment

1. Start a disposable shortener and lookup service on loopback. Seed known short links and their expected destinations. Include one absent link and its expected missing-link response.
2. Warm the fixture, then request that dataset on a fixed schedule. Capture the status and check the redirect’s `Location` header locally. Following the redirect would measure a different service.
3. Enable a temporary delay on the run-owned lookup. Continue the same workload and record when the delay actually takes effect. The fragile implementation waits for each lookup. Report what happens to response latency, correctness, timeouts, and scheduled requests that could not start.
4. Clear the delay, verify ordinary lookup behavior, and measure a recovery window. Record cleanup even if an assertion failed or the operator pressed Ctrl-C.
5. Reset the fixture. The improved shortener has a shorter internal lookup deadline and a cached fallback for known links. Repeat the same dataset, schedule, external request deadline, and thresholds. Verify each destination and the missing-link response again.
6. Compare corresponding windows. A quick redirect to the wrong destination fails correctness. A useful finding identifies the affected phase, shows the observations behind it, and explains which change to test next.

## Command-line interface

| Proposed command | User-visible contract |
|---|---|
| `breakyard demo` | Launch the bundled fixture, run the incident, show progress, and report recovery. Works without service credentials, Docker, or a cloud account. |
| `breakyard init` | Write a versioned project configuration with explicit local targets and limits. Preserve existing configuration. |
| `breakyard plan` | Validate the configuration offline. Show the exact endpoints and operations, schedule, resource ceilings, assertions, permissions, and cleanup procedure. Perform no probes or mutations. |
| `breakyard run` | Perform live preflight, execute the validated experiment in the foreground, show phase changes, save evidence, and return a run ID. |
| `breakyard report <run-id>` | Read a saved run offline and produce Markdown or versioned JSON. Keep progress on stderr and machine-readable output on stdout. |
| `breakyard compare <before-id> <after-id>` | Compare matching windows. Show relevant differences in conditions and identify comparisons that cannot support a conclusion. |

Help, errors, non-interactive output, and process exit status are part of the CLI contract. Automation must distinguish a completed run with a failed assertion from invalid input, an incomplete run, an engine failure, and unverified cleanup. Publish the numeric exit codes before stabilizing the CLI. Future commands for discovery, doctor checks, scenario validation, history, and a TUI may reuse these operations. A future TUI should support keyboard navigation, explicit stop behavior, visible units, and a text equivalent for every status indicated by color.

## Configuration and scenario model

A **project** describes what the developer owns and permits. An **experiment** references named project resources and describes what to measure. A **run** records the resolved inputs and what actually happened. Keeping the three separate lets another developer inspect the experiment before granting it access to a local service.

| Record | Required information |
|---|---|
| Project | Schema version, project name, exact local endpoints and allowed operations, fixture ownership, and maximum request rate, concurrency, duration, response bytes, and artifact bytes. Later Compose resources also require confirmed service identities and dependency edges. |
| Experiment | Schema version and scenario identity, named targets and routes, workload model and budget, seed when randomness is used, timeouts, named windows, typed actions, assertions, and cleanup checks. |
| Assertion | Identifier, applicable route or fixture, observation window, metric or expected business outcome, comparison and threshold, and evidence required for evaluation. |

The following **illustrative fragment** describes the built-in demo. The field names and values explain the model. The first CLI milestone embeds the fixture, and a published configuration schema is a later release deliverable.

```yaml
version: 1
scenario: slow-lookup
target: demo-shortener
workload:
  route: known-and-missing-links
  arrivals_per_second: 4
  max_in_flight: 2
  request_timeout: 1s
windows:
  - {name: baseline, duration: 10s}
  - {name: delayed, duration: 10s, fixture_action: delay-owned-lookup}
  - {name: recovery, duration: 10s}
assertions:
  - {check: expected-redirect-or-missing-link, window: delayed}
  - {check: lookup-delay-cleared, window: recovery}
```

Use typed units and named routes. Reject unknown versions or fields, missing targets and windows, undeclared actions, out-of-budget workloads, and conflicting operations before execution. Scenarios carry declarative data. Future compound scenarios must define overlap, abort, and reversal semantics before their actions can be combined. Record the resolved version, seed, and configuration fingerprint with the run. Keep secret values outside fingerprints and exports.

## Architecture and lifecycle

Start with one Go executable and standard-library HTTP, context, timing, and flags. The CLI, orchestration, scheduling, aggregation, assertion evaluation, and reporting can call each other in process. The disposable fixture owns its HTTP endpoints and control for temporary delay. A developer’s existing application remains a separate process. [First-release process diagram](docs/design.md#process-and-data-flow)

| Component | Responsibility |
|---|---|
| Planner and orchestrator | Validate the project and scenario, resolve budgets, run preflight, own the clock and cancellation, schedule phases, and guarantee a cleanup attempt on every exit path. |
| HTTP driver and workload scheduler | Reuse bounded connections, enforce deadlines, admit work within budget, classify terminal outcomes, and account for missed arrivals. |
| Fixture control | Check ownership, report capabilities, apply a time-limited change, identify it by run ID, revert it idempotently, and verify the resulting state. |
| Collectors and aggregation | Collect route and generator measurements independently of terminal rendering. Bound queues and label sampling gaps. Keep lifecycle and cleanup records durable. |
| Assertions, findings, and reports | Evaluate declared expectations, link findings to observations, persist results, render offline, and check compatibility between runs. |

**Lifecycle.** Validate → preflight → baseline → apply delay → observe → recover → evaluate → persist. Cancellation or an engine error during an active phase goes through recovery. Track run completion, assertion outcomes, and cleanup verification independently. A valid experiment can finish, fail a latency assertion, and successfully restore its fixture.

Record intent before applying a change. Record the handle and actual apply/revert times as they become known. If the runner stops between those writes, reconcile using the stable run-owned identity. Give cleanup a fresh, bounded deadline after canceling the workload. Keep the UI and storage writer off the scheduling path so a slow terminal or disk cannot silently change the offered workload.

**Repository layout.** Introduce `cmd/breakyard` and the root Go module at M1. Add internal packages when the driver, runner, storage, and reporting acquire distinct responsibilities. Keep fixtures and their expected data together. Introduce scenario files, schemas, benchmarks, or optional adapters alongside their first implementation. Reuse vetted HTTP, TLS, parser, and histogram primitives. Implement the scheduler, lifecycle, accounting, and explanations where contributors can test and understand them.

## Workload and measurement

Start with independent HTTP reads and a fixed, bounded arrival schedule. Open-loop arrivals are scheduled independently of prior responses. A closed-loop model waits for the current operation before starting another, so throughput naturally falls as response time rises. Later profiles may add gradual ramps, steps, spikes, sustained demand, weighted routes, hot keys, or seeded journeys. A stateful journey requires declared test data, expected transitions, and explicit retry semantics.

| Measurement | Definition or interpretation |
|---|---|
| Offered demand | Scheduled arrivals per window divided by the window duration. Record scheduler lag and arrivals that miss admission when all workers are occupied. |
| Achieved rate | Report starts and completions per second separately. Requests can start in one window and finish in another. Record both timestamps and associate request latency with its start window. |
| Outcomes | Count starts, responses, transport failures, timeouts, cancellations, and dropped arrivals. Record HTTP status and fixture correctness separately. Publish each rate’s denominator. Preserve in-flight counts at window boundaries. |
| Latency | Measure elapsed time from dispatch to terminal outcome. Track schedule-to-outcome time separately to expose queueing. Report percentile method, sample population, window, and count. Merge compatible distributions before deriving an aggregate percentile. |
| Recovery | Measure from actual fault removal until the declared recovery criteria hold for the observation interval. An insufficient observation interval yields an unknown result. |
| Generator health | Report lag, missed work, in-flight requests, collector gaps, and available resource measurements. Shared-host contention can affect both the application and the generator. |

Use monotonic time for intervals and UTC timestamps for correlation. Bound request and response handling, connection use, worker count, artifact volume, and run duration. A saturated runner should expose unmet demand even if the requests it did complete look healthy.

## Evidence and reports

Store versioned JSON per run in the user’s private XDG state directory. Write complete files by atomic rename. Preserve partial status on interruption. Render Markdown from the saved record so reviewers can inspect the same evidence later. SQLite becomes a candidate when searching or comparing local history becomes cumbersome.

| Record | Fields that make a result interpretable |
|---|---|
| Run identity and conditions | Run ID, schema version, scenario version and fingerprint, seed, dataset, application revision, OS/runtime, generator placement, limits, warm-up, start and end times. |
| Event | Sequence within the run, elapsed time, UTC time, source, kind, named target, and redacted attributes. Include actual phase transitions, fault intent, observed application/removal, and cleanup verification. |
| Metric window | Phase, named route, offered and achieved demand, counts and denominators, latency distribution and sample size, generator health, and reasons for missing observations. |
| Assertion and finding | Expected and observed value, evidence references, pass/fail/unknown, and reason. Findings also include prerequisites, alternative explanations, confidence category, and a follow-up experiment. |
| Completion | Execution status, incomplete phases, cleanup status and affected owned resources, report format version, and export redaction status. |

Start with HTTP outcomes, fixture events, and generator measurements. Later sources may include scoped Linux/cgroup and container statistics, database pool or lock waits, cache hits and evictions, queue depth, and opt-in Prometheus metrics or OpenTelemetry spans. Record collection overhead and scope. Host CPU alone cannot establish which process caused contention.

**Diagnosis.** Begin with transparent rules. A recorded lookup delay followed by higher redirect latency supports a hypothesis about dependency waiting. Stronger evidence comes from the improved variant under equivalent demand. Unchanged lookup measurements, a saturated generator, or overlapping faults weaken the inference. Use evidence labels such as *observed*, *supported hypothesis*, or *insufficient data*. A missing metric yields an unknown result with a reason.

**Comparison.** Match scenario version, definitions, fixtures, schedule, warm-up, limits, thresholds, topology, generator placement, and measured demand. Identify the application revision as the intended change. Show per-phase differences, sample counts, incomplete windows, and variation across repeated trials. A run that served fewer requests cannot establish a capacity improvement from a shorter tail latency alone.

## Isolation and recovery

- **Target selection.** P0 accepts explicit loopback identities and operations for measurement. Disable inherited HTTP proxies and redirect following. Validate the address and port actually dialed. Applying a change additionally requires a disposable dependency owned by the run.
- **Data and privileges.** Use synthetic fixtures, mock external providers, and ordinary user permissions. Exclude authorization headers, cookies, raw queries, bodies, environment values, and credentials from logs and exports. Store artifacts with private permissions and constrain their paths and size.
- **Emergency stop.** Stop new arrivals on cancellation, deadline, a scope violation, or a breached resource budget. Cancel outstanding work, retain partial counters, clear the owned delay, and probe its removal. If verification fails, identify the remaining resource and recovery action in the report.
- **Runner failure.** Enforce delay expiry on the fixture side and bound fixture lifetime. Test unexpected process death, repeated cleanup, and loss of the artifact store. Recovery can claim success only when its postcondition was observed.
- **Later integrations.** Compose discovery requires confirmation and ownership checks for selected services, networks, and volumes. New adapters declare capabilities, prerequisites, resource budgets, and reversal behavior. Any future worker architecture requires authenticated jobs, expiring leases, global budgets, revocation, and explicit reporting of disconnected workers.

## Verification and release gates

| Gate | Acceptance evidence | Engineering concept |
|---|---|---|
| M1 · One request | A Linux build and loopback integration test distinguish success, deadline expiry, and caller cancellation. An out-of-scope endpoint is rejected before dialing. | HTTP lifecycle and cancellation propagation |
| M2 · One incident | The demo shows baseline, delay, and recovery. A busy generator accounts for missed arrivals. Interrupt each phase and confirm cleanup or a visible outstanding action. An unrelated local listener stays healthy. | Scheduling, backpressure, containment, and recovery |
| M3 · Compare the fix | Both variants use the same fixture. Wrong redirects fail, missing evidence is unknown, mismatched workloads are flagged, and a real recording links to a redacted report. | Tail latency, correctness, and experimental controls |
| M4 · Public MVP | Clean-machine installation from a checksummed Linux archive, a side-effect-free plan, the full local CLI workflow, export redaction, and race checks in CI. Observe five Linux developers trying the quickstart. Aim for four to finish within five minutes unaided and explain one result. | Release engineering, reproducibility, and usability |

Use a fake clock for accounting and phase boundaries, real sockets for HTTP classification, and end-to-end tests for the documented CLI sequence. Include malformed configuration, zero samples, saturated workers, a slow consumer, corrupted or interrupted writes, runner death, and repeated cleanup. Separate real networking tests with tolerances from deterministic unit tests. Add regression tests for past failures.

Benchmark the generator independently against a known fixture. Record hardware, OS, runtime, placement, offered and achieved demand, CPU/memory, scheduler lag, latency, errors, and sample counts. If comparing with an established load tester, match the workload and timeout semantics. Publish measurements from actual runs. Record shared-host limitations and generator saturation alongside them.

## Scenario catalog

The first release exercises fixed dependency latency. The catalog below preserves the wider set of reliability questions from the project brief. A scenario becomes eligible for implementation when it has a disposable fixture, meaningful assertion, bounded scope, and verifiable cleanup.

| Area | Candidate experiments and questions |
|---|---|
| Traffic and workload | Steady demand, ramps, spikes, sustained demand, breakpoint and soak tests, hot routes or keys, skewed reads/writes, duplicate operations, cold starts, slow clients, and uneven replica load. Where does queueing begin and which requests still meet their objectives? |
| Networking and dependencies | Delay, jitter, disconnection, partial responses, bandwidth constraints, isolated partitions, DNS or TLS failures, stale endpoints, dependency throttling, and gradual recovery. Which deadline or fallback determines the user-visible outcome? |
| Databases and consistency | Pool exhaustion, slow queries, lock contention, deadlocks, lost updates, oversold inventory, duplicate mock payments, partial transactions, replication lag, failover, migration, and restore validation. Which declared business invariant survives? |
| Caches and queues | Cold or unavailable caches, stampedes, stale or missing keys, eviction, backlog, duplicate delivery, ordering, poison messages, and slow or restarting workers. Can the service recover within the budget and preserve the expected data? |
| Resources and deployments | Scoped CPU or memory limits, disposable storage constraints, file descriptors, replica loss, restart loops, rolling deployment, configuration drift, mock clock skew, credential rotation with test secrets, and collector outages. What evidence distinguishes application pressure from a measurement failure? |
| Distributed coordination | Quorum loss, leader changes, expired leases, partial fan-out, cascading failures, retry amplification, and recovery bursts. These require an actual disposable replicated system, explicit consistency assumptions, and verified isolation. |

Future multi-step teaching fixtures include a hot link with cache fallback, a mock checkout with a repeated idempotency key, a queue recovering a backlog, and a faulty replica being rolled back. Other candidates examine overlapping dependency loss and peak demand, gradual resource growth, or an isolated partition. Each constituent action must have its own acceptance and recovery tests before combination.

## Distribution and future decisions

Publish the local core under MIT. At M3, show a real incident, the code change, and the measured result in a short recording. At M4, publish a Linux archive, checksum, versioned quickstart, and a redacted example report. A contributor should be able to propose a scenario using one observable symptom, one disposable fixture, and one outcome that would count as a fix. [Contribution guide](CONTRIBUTING.md)

The project's hypothesis is that combining a real local application, explicit correctness checks, and an understandable before/after report is valuable to developers learning reliability engineering. Existing load testers, dependency proxies, and observability tools offer useful comparison points for individual components. Validate the complete workflow by watching developers run it on their own projects. Track completion, time to first result, explanation quality, repeat use, cleanup verification, and contributions alongside GitHub stars.

| Decision or risk | Trigger and response |
|---|---|
| Scope and onboarding | Ship the single shortener experiment first. Choose the next capability from friction observed on a second run. |
| Storage and UI | Consider indexed SQLite history when JSON navigation becomes costly, and a TUI when users need live investigation beyond the phase summary. Preserve non-interactive output. |
| Statistics and telemetry | Check accounting against controlled fixtures and repeated trials. Surface missing collector data and shared-machine contention. |
| Third-party dependencies | At the milestone that needs one, review its license, maintenance, supported platforms, and packaging cost. Record the decision and considered alternatives. |
| Configuration compatibility | Version schemas and saved reports before inviting external scenario contributions. The file format, migration rules, and numeric exit codes need review before the public MVP. |
| Platform and distribution | The repository and binary name are Breakyard. Review release architectures, signing, registry names, and any domain or trademark questions before wider distribution. |
| Commercial extensions | Evaluate shared team history, managed isolated environments, hosted workers, or training only after validating demand. The local experiment, evidence, and safety controls stay available without an account. |
