# agent-stuff

Reference material for agent-assisted work — guides written once, referenced from
many repos.

## Guides

- [`guides/package-guide.md`](guides/package-guide.md) — approaching a new Python
  package or module: naming the domain, drawing the public surface, altitudes,
  seams, errors, types, docs. Guidelines distilled from flask, fastapi, requests,
  httpx, pydantic, attrs, click, rich, sqlalchemy and polars, with `requests`
  walked end to end as a worked example.
- [`guides/cli-guide.md`](guides/cli-guide.md) — designing a command-line
  program: help and discoverability, flags, output, errors, dangerous
  operations, configuration, robustness, naming. Adapted from
  [clig.dev](https://clig.dev/), with `git` walked end to end as a worked
  example.
- [`guides/backend-app-guide.md`](guides/backend-app-guide.md) — building and
  improving a backend service, adapted from
  [The Twelve-Factor App](https://12factor.net/). Each factor carries the
  observable signals of a deviation and the legitimate reasons to deviate, plus
  what has aged since 2011 and what the industry has added. Ends with an audit
  procedure ordered by blast radius.
