# Contributing to Breakyard

Breakyard is in the design phase. The next implementation task is **M1 · Measure one request**. It includes a local fixture and acceptance checks small enough for one pull request. [Roadmap](ROADMAP.md)

## Propose a scenario

Open an issue describing one dependency in a local project, what a user would observe if it slowed down, and what evidence would convince you a fix worked. Include a way to reproduce the behavior with disposable data. For example, a shortener could check the exact redirect destination while its lookup is delayed. Use synthetic names and values in examples. [Open an issue](https://github.com/dhruvkshah75/breakyard/issues/new)

## Test a milestone

Pick an acceptance check from the roadmap. Once the implementation is available, try it on a disposable Linux fixture and report the project revision, environment, expected behavior, observed behavior, and cleanup result. Redact private endpoints and credentials.

For a design change, explain which user-visible behavior changes and update its acceptance check in the same pull request. Preview Markdown links and diagrams on GitHub. For an implementation change, include the focused regression check and the command you ran. [Experiment design](docs/design.md)
