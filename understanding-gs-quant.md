# Reading `gs-quant`: Engineering Notes on a Production Quant Library

<p align="center">
<img alt="type" src="https://img.shields.io/badge/type-engineering_case_study-4B5563?style=flat-square"/>
<img alt="subject" src="https://img.shields.io/badge/subject-goldmansachs%2Fgs--quant-1a1a2e?style=flat-square"/>
<img alt="license" src="https://img.shields.io/badge/gs--quant_license-Apache_2.0-2E8B57?style=flat-square"/>
<img alt="author" src="https://img.shields.io/badge/notes_by-Kartik_Singh-e07a00?style=flat-square"/>
</p>

> **What this document is.** These are my notes from reading Goldman Sachs's open-source [`gs-quant`](https://github.com/goldmansachs/gs-quant) library — **not a project I wrote**. The code, the design decisions, and the years of quant expertise behind it belong to Goldman Sachs and its contributors, under the Apache 2.0 license. What follows is my own read on *why* it's put together the way it is, and what I'm taking from it into my own work. Sources for every specific claim are linked at the bottom.

---

## Why I picked this one

Most open-source repos I read for practice are toy projects with no real production pressure behind them. `gs-quant` isn't one of those. It's the toolkit Goldman Sachs's own sales, strategists, and risk teams use daily to price derivatives and manage risk — released publicly under Apache 2.0, with 11,000+ external stars and a release cadence that's still active as of mid-2026. Reading a codebase that has to survive real trading desks, real audits, and outside contributors at the same time is a different exercise than reading a library built purely for GitHub stars, and I wanted to see what that pressure actually does to the engineering.

## Quick facts

| | |
|---|---|
| Maintainer | Quant developers at Goldman Sachs |
| First released | ~2018–2019 (per its `NOTICE` file) |
| License | Apache 2.0 |
| Community | 11,000+ stars, 1,500+ forks |
| Language | Python (3.10+) |
| Cadence | Still shipping — latest tagged release was mid-July 2026 |
| Repo | [github.com/goldmansachs/gs-quant](https://github.com/goldmansachs/gs-quant) |

## What it actually does

The package is organized by domain rather than by technical layer, so the top-level module list reads almost like a glossary of quant finance:

- **`instrument` / `markets` / `models`** — define tradable products (swaps, caps, floors, options…) and the pricing contexts they're evaluated in
- **`risk`** — computes and aggregates sensitivities (delta, vega, and similar greeks) across a single instrument or an entire portfolio
- **`data` / `datetime` / `timeseries`** — pulls market data and runs analytics (rolling volatility, correlation) over it
- **`backtests`** — replays strategies against historical data
- **`entities` / `target`** — typed domain objects generated from Goldman's internal API schemas
- **`tracing`** — OpenTelemetry instrumentation wired through the request layer
- **`workflow`** — higher-level orchestration built on top of the above

Two audiences share one package: institutional Goldman Sachs clients with real credentials get live pricing, risk, and data access, while anyone can install it and use the analytics, timeseries, and backtesting layers standalone with no Goldman account at all.

## Six patterns worth stealing

**1. Resilience lives at one boundary, not scattered through the codebase.**
Outbound calls run through a single session layer, and that's the only place retry logic, timeouts, and error translation happen. Requests that fail with a 500/502/503/504 get retried with exponential backoff before giving up; everything else — a bad auth token, a malformed request, a session that was never initialized — becomes one of a small number of typed exceptions instead of a raw HTTP error leaking out. One place to look when something network-related breaks, instead of a dozen.

**2. Optional installs are shaped around *who's* installing, not just *what* they need.**
`pyproject.toml` splits extras into `notebook`, `internal`, `test`, `develop`, and — a genuinely interesting recent addition — `mcp`. A researcher in a notebook, a Goldman employee with internal access, a CI runner, a contributor setting up a dev environment, and now an LLM agent calling `gs-quant` as a tool all get exactly the dependencies their role needs. That last group is worth calling out on its own: `fastmcp`, `uvicorn`, `pydantic`, and a bundled `skills/*.md` folder mean gs-quant is being wired up to be called *as MCP tools*, not just imported as a Python library. For anyone interested in AI product work, that's a live example of a serious financial engineering org exposing existing analytics through an agent-facing interface instead of standing up a parallel one.

**3. One source of truth for the version number.**
Versioning runs through `versioneer`, which derives the package version straight from git tags (`release-x.y.z`) at build time. Nobody hand-edits a version string and forgets to bump it — the tag *is* the version.

**4. House style is enforced by the linter, not just written down in a doc.**
`ruff`'s `banned-api` rules physically block importing deprecated internal modules (pointing to the stable replacement right in the error message) and block direct `pytz` imports in favor of `zoneinfo`. Import-convention rules force `import pandas as pd` / `import numpy as np` everywhere. None of this depends on a contributor remembering a wiki page — it fails the pre-commit hook if you get it wrong.

**5. The test suite doesn't need a Goldman Sachs login to run.**
A root-level `conftest.py` registers a mock-data plugin that lets the suite skip integration tests requiring live credentials. That's what makes it possible for an outside contributor, or a public CI runner, to run a meaningful chunk of the tests with zero access to Goldman's actual infrastructure.

**6. One codebase, two release pipelines.**
The internal GitLab CI config and the public GitHub mirror ship from the same source, with an internal-only artifact-upload stage in the GitLab pipeline that has no equivalent in the public-facing story at all. It's a clean way to publish open source without forking the internal and external codebases into two things that slowly drift apart.

*(Small bonus, because it's rare to see it actually confirmed rather than just promised: the `NOTICE.txt` already credits an outside contributor through Goldman's lightweight DCO process instead of a signed CLA — proof the low-friction contribution model described in `CONTRIBUTING.md` gets used in practice, not just documented.)*

## What I'm taking into my own repos

- **Persona-shaped optional dependencies** are going straight into Verifyr — right now its FastAPI backend has one dependency list; splitting `test` / `worker` / `dev` extras would let the Arq/Redis worker install without pulling in Playwright, and vice versa.
- **Linter-enforced import conventions** are an easy win anywhere I keep leaving the same "please import it this way" comment in review — ruff can just enforce it instead.
- **Mock-data-first test design** is the one I care about most long-term: it's the difference between a portfolio repo a stranger can clone and `pytest` immediately, and one where half the suite silently needs secrets they'll never have.

## Explore it yourself

- Repo — [github.com/goldmansachs/gs-quant](https://github.com/goldmansachs/gs-quant)
- Docs — [developer.gs.com/docs/gsquant](https://developer.gs.com/docs/gsquant/)
- Package — [pypi.org/project/gs-quant](https://pypi.org/project/gs-quant/)

---

## Attribution

`gs-quant` is © Goldman Sachs, licensed under Apache 2.0. Every design decision, pattern, and line of code discussed above belongs to Goldman Sachs and its contributors — this document only reflects on publicly available material (the repo's configuration, its public documentation, and its release history). These notes themselves are © Kartik Singh.
