# Road to v0.1

**Current milestone: blueprint.** The first release follows one incident from observation to a verified fix. Each milestone builds on the preceding one and has an outcome a contributor can demonstrate.

## M1 · Measure one request

**First implementation task.** Add a root Go module, the `cmd/breakyard` entry point, and a disposable loopback HTTP fixture. Have the CLI make one request to the fixture and print the outcome and elapsed time. Use the Go standard library.

**Acceptance.** Exercise success, deadline expiry, and caller cancellation separately. Reject an out-of-scope destination before dialing. Close the fixture and exit after each case. Run the tests on Linux and add a build and test check when the module lands.

**Learning exercise.** Draw a request from start through response completion. Explain where a timeout takes effect and how to distinguish it from Ctrl-C.

## M2 · Observe one incident

With M1 measuring requests, add the shortener fixture and its lookup dependency. Implement a bounded schedule and phase counters. Expose the first user-facing experiment through `breakyard demo`.

**Acceptance.** Observe baseline, delay, and recovery in the terminal. A slow lookup changes the measured latency. A busy generator counts missed arrivals. Cancel once in each phase and verify that owned resources are released or that the remaining cleanup is reported. Test expiry after the runner exits unexpectedly.

**Learning exercise.** Hold scheduled demand constant and increase fixture latency. Predict when concurrency becomes the limiting factor, then check the counters.

## M3 · Compare the fix

Use the same fixture to compare a fragile lookup with a deadline and cached fallback. Save versioned JSON evidence, render Markdown, and compare corresponding phases.

**Acceptance.** Verify known and missing links in both variants. A wrong destination fails correctness even if it returns quickly. Missing samples produce an unknown result. Flag incompatible workloads. Record a real 30–45 second walkthrough and link the redacted report from the README.

**Learning exercise.** Identify the one experimental variable deliberately changed. Explain which observations would weaken the claim that the fallback helped.

## M4 · Release the local MVP

Extend the demonstrated flow to a developer’s own explicitly configured local HTTP endpoint. Deliver the proposed `init`, `plan`, `run`, `report`, and `compare` workflow, a checksummed Linux binary, installation instructions, and a clean-machine quickstart. [CLI and report contracts](docs/design.md)

**Acceptance.** Run the quickstart without cloud credentials or elevated privileges. Verify dry-run has no side effects, exported reports omit secrets, and CI covers the CLI workflow, cancellation, target validation, and Go race checks. Publish measurements with their test conditions.

**Learning exercise.** Give five Linux developers the quickstart and observe where they get stuck. Aim for four to finish within five minutes unaided and explain one result. Revise the walkthrough before announcing v0.1.

## After the first release

Use feedback from people who run a second experiment to choose among Compose discovery, interactive run history, and additional local scenarios. Keep distributed execution and hosted team features as research until there is evidence of demand.

At launch, publish the terminal recording, its report, and a short write-up of the change to the shortener. Review stars, unique visitors, clones, and referrals weekly. Ask volunteers whether they repeated the experiment on their own service and which decision it helped them make.
