# Breakyard

**Give your side project its first production outage.**

Breakyard is a planned Linux CLI for developers who want to find out how their own backend behaves when a dependency slows down. The experiment runs locally, uses disposable data, and produces a report you can share with someone reviewing a fix.

**Design phase.** The first milestone is measuring a single local HTTP request. Read the [full product and engineering specification](SPEC.md) for the requirements, or the [roadmap](ROADMAP.md) for the build order and acceptance criteria.

## The first experiment

Imagine a URL shortener. Every redirect waits for a lookup. If that lookup takes longer, does the user still reach the right page in time?

1. Measure redirects against a small, known set of links to establish a baseline.
2. Add a temporary delay to the fixture’s lookup service. Watch request times and missed deadlines, then remove the delay and verify recovery.
3. Give the shortener a lookup deadline and a cached fallback for known links. Repeat the same experiment. Check the destination of each redirect as well as how long it took.

The planned report puts both runs side by side. A fast redirect to the wrong page fails the experiment. A runner that cannot keep up must say how many requests it missed. [See the experiment design](docs/design.md).

## Help shape the first release

**Which local dependency failure would you like to understand better?** A short example from your project can help define the next fixture. [Contributing guide](CONTRIBUTING.md) · [MIT license](LICENSE)
