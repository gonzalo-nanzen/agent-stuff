# Building a backend service

Guidelines for creating and improving backend applications, adapted from
**[The Twelve-Factor App](https://12factor.net/)** by Adam Wiggins and
contributors at Heroku — still the best short statement of what makes a service
deployable, scalable and operable.

**These are guidelines, not rules.** Twelve-factor was written in 2011 against
Heroku-shaped deployments. Most of it has aged extremely well; some of it hasn't,
and one part actively conflicts with current security practice (§III). Sections
at the end cover what has aged and what the industry has added since.

## How to use this when you find a deviation

This guide is written to be checkable. Each factor carries a **Deviation looks
like** block listing concrete, observable signals — things you can grep for or
see in a diff — and, where they exist, **Legitimate reasons to deviate**.

When you find a deviation, the useful move is not to fix it reflexively:

1. **Name the factor.** "This keeps session state in process memory — factor VI."
   Naming it turns a vague smell into a discussable decision.
2. **Decide whether it is deliberate or incidental.** Incidental deviations are
   usually bugs waiting to surface under load or during a deploy. Deliberate ones
   are trade-offs someone made, possibly correctly.
3. **If deliberate, check that the reason is written down** somewhere the next
   reader will find it — a comment at the site, an ADR, a note in the service's
   README. An unexplained deliberate deviation decays into an incidental one as
   soon as the person who decided it leaves.
4. **If incidental, say what it will cost** in concrete terms — "this breaks when
   we run more than one replica" — rather than citing the factor as authority.
5. **Never silently normalise a deliberate deviation.** If a service pins itself
   to one replica on purpose, "fixing" its in-memory cache is a regression, not an
   improvement.

The factors are not equally important. **III (config), VI (stateless processes),
IX (disposability) and XI (logs)** are the ones whose violation causes production
incidents. The rest mostly cost you time and confidence.

---

## I. Codebase — one codebase in version control, many deploys

One-to-one between codebase and app. Many deploys of it — production, staging,
every developer's laptop — all running the same codebase at possibly different
commits.

If two things share a codebase, they're one app. If one app spans two codebases,
it's a distributed system, and each part is its own app that can independently
comply.

**Deviation looks like:**
- The same code copy-pasted into two services instead of extracted into a library
  and pulled through the dependency manager.
- Environment-specific branches (`production`, `staging`) rather than one trunk
  deployed at different commits.
- Deploy-time patching — files edited on the server, config templated into source
  during deploy.

**Legitimate reasons to deviate:** a monorepo holding several apps is *not* a
violation, despite the wording. The rule is one codebase per app; a monorepo with
clear per-app boundaries and independent deployability satisfies it. What matters
is whether the apps can be built and released independently.

## II. Dependencies — declare and isolate them explicitly

Declare every dependency in a manifest, and use isolation at runtime so the
surrounding system's packages can't leak in. **Both, not either.** A manifest
without isolation still lets a system-wide package satisfy an import that you
never declared.

The test: a new developer with only the language runtime and the package manager
can check out and build with one command.

**Deviation looks like:**
- Unpinned or absent lockfile; `pip install` without a resolved lock.
- Shelling out to a binary that isn't declared anywhere — `curl`, `git`,
  `imagemagick`, `psql`. These exist on your laptop and may not exist in the
  container.
- Tests that pass locally and fail in CI on an import, or vice versa.
- Setup instructions in the README that involve installing anything system-wide.

**Legitimate reasons to deviate:** system libraries that genuinely belong to the
base image (libc, OpenSSL, a database client library) are normally declared in the
Dockerfile instead of the language manifest. That's still explicit declaration —
just in the other manifest. What isn't fine is depending on something declared in
neither.

## III. Config — store config in the environment

Config is everything that varies between deploys: connection strings, credentials,
hostnames, feature toggles. It must live outside the code.

**The litmus test is the sharpest thing in the whole document:** *could you open
source this repository right now, without leaking a single credential?* If not,
config and code are entangled.

What is *not* config: anything constant across deploys. Route tables, DI wiring,
framework setup. Those belong in code.

Twelve-factor also argues against grouping config into named environments
(`development`, `staging`, `production`) because the groups multiply as deploys
multiply. Treat each variable as independent.

**Deviation looks like:**
- Credentials, tokens or connection strings in tracked files. Grep the history,
  not just the working tree.
- `if env == "production":` branching in application code. The code should not
  know which deploy it is; it should read differing values.
- A committed `config/production.yaml` that has to be edited to deploy elsewhere.
- Defaults that silently work in development and fail obscurely in production —
  a `SECRET_KEY` defaulting to `"dev"` is a production incident waiting for its
  moment.

**Where this factor has aged badly — read this part.** Twelve-factor says to put
*everything*, including secrets, in environment variables. Do not do this for
secrets. Environment variables are inherited by every child process, visible in
`docker inspect` and `systemctl show`, frequently dumped by crash reporters, and
easy to leak through a shell substitution or a debug endpoint.

The current split:

| | mechanism |
|---|---|
| Non-secret config | environment variables — factor III as written |
| Secrets | a secrets manager, or a file mounted at a path named by an env var |

The env-var-shaped interface survives (`DATABASE_URL_FILE=/run/secrets/db_url`);
what changes is that the *value* isn't in the environment. This is the same
conclusion the [CLI guide](cli-guide.md) reaches from the other direction.

## IV. Backing services — treat them as attached resources

A backing service is anything you reach over the network: database, cache, queue,
SMTP, object storage, third-party API. The code must not distinguish between one
you run and one you rent. Both are a URL and credentials in config.

The test: could you swap the local Postgres for a managed one by changing only
config? Could you detach a sick database and attach a restored replica without a
deploy?

**Deviation looks like:**
- A hostname, port or bucket name hardcoded anywhere outside config.
- Code that special-cases a provider — `if using_s3:` branches around storage
  calls instead of one storage interface with two configured implementations.
- Direct filesystem access to something another process also writes, in place of
  an actual service.
- Assuming a service is on `localhost`.

**Legitimate reasons to deviate:** genuinely embedded stores — SQLite as an
application file format, an on-disk index that *is* the app's data structure
rather than a shared service. Being clear about which you have matters: SQLite as
a local cache is fine, SQLite as your production database means factors VI and
VIII do not apply to you and you should say so out loud.

## V. Build, release, run — keep the three stages strictly separate

- **Build** — code at a commit → an executable artifact, with dependencies
  fetched and assets compiled.
- **Release** — that build + this deploy's config → an immutable, uniquely
  identified release.
- **Run** — execute a release.

Releases are append-only and immutable. You cannot change one; you make a new
one. That is what makes rollback a selection rather than a rebuild.

Complexity belongs in build, which a human is watching. Run happens at 3am when a
node reschedules, and should be as close to trivial as you can make it.

**Deviation looks like:**
- Building the image at deploy time, or on the target host.
- Anything that mutates the artifact at startup — fetching dependencies, compiling
  assets, downloading config, `git pull` in an entrypoint.
- Rolling back requires a rebuild, or isn't possible.
- Releases that aren't uniquely identified, so "what's running in prod?" needs
  investigation.
- Image tags that move. `:latest` in a deployment manifest means you cannot say
  what is running or reproduce it.

**Note on migrations.** Database migrations sit awkwardly across this boundary —
they're factor XII work that must happen between release and run. Running them
from an application startup hook makes every replica race for the lock and makes
rollback dangerous. Run them as a distinct one-off process against the same
release.

## VI. Processes — execute the app as stateless, share-nothing processes

Anything that must persist goes to a backing service. Memory and local disk are
usable as a single-transaction scratchpad and for nothing else, because the next
request will land on a different process and a restart wipes everything.

Sticky sessions are called out by name as a violation.

**Deviation looks like:**
- Session state in process memory.
- An in-process cache that is treated as authoritative rather than as an
  optimisation — where a cold process returns *wrong* answers, not just slow ones.
- Uploads written to local disk and served back later.
- Background jobs held in an in-memory queue, or scheduled with an in-process
  timer, so they vanish on restart.
- Module-level mutable state accumulating across requests.
- Anything that only works because there's exactly one replica.

**The test that finds all of these:** run two replicas behind a round-robin load
balancer and kill one at random during a test run. Every deviation above surfaces
within minutes.

**Legitimate reasons to deviate:** a read-through cache of immutable data,
warmed on demand and correct when empty, is fine. Long-lived connections
(WebSockets, SSE) inherently pin a client to a process — that's a real constraint,
and the answer is to make the *state* recoverable so a reconnect to a different
process works, not to pretend the pinning doesn't exist.

## VII. Port binding — export services by binding to a port

The app is self-contained. It includes its own server library and binds a port.
It is not injected into an external webserver container at runtime.

The contract with the platform is exactly one thing: bind this port, serve
requests. Routing from a public hostname is the platform's job.

**Deviation looks like:**
- The app can't be started with a single command that makes it serve.
- Requiring an external webserver's config file to function at all — as opposed to
  having a reverse proxy in front for TLS and routing, which is normal.
- A hardcoded port instead of one read from config.

A consequence worth noticing: one app binding a port can *be* a backing service
for another, attached by URL. That's factor IV and VII composing.

## VIII. Concurrency — scale out via the process model

Assign work types to process types — HTTP to web processes, background work to
worker processes — and scale by running more of them. In-process threads and
async are fine *within* a process; they are not the mechanism you scale with.

Processes should not daemonize or write PID files. Process supervision belongs to
systemd, the container runtime, or the orchestrator.

**Deviation looks like:**
- Background work running inside the web process, so scaling HTTP capacity also
  scales job concurrency, and a deploy drops in-flight jobs.
- A self-managed thread pool or subprocess fan-out doing what another replica
  should do.
- The app daemonizing itself, forking to background, or writing a PID file.
- Vertical scaling as the only lever — the answer to load is always a bigger
  machine.

**Legitimate reasons to deviate:** small, genuinely fire-and-forget work
(emitting a metric, warming a cache) doesn't need a queue and a worker fleet. The
line is durability: if losing the work on restart matters, it needs a real queue
and a separate process type.

## IX. Disposability — fast startup, graceful shutdown

Processes can be started or killed at any moment, and the app is better for it —
faster deploys, faster scaling, tolerance for node loss.

**Startup** should take seconds, not minutes.

**Shutdown on SIGTERM:** a web process stops accepting new connections, finishes
in-flight requests, exits. A worker returns its current job to the queue (NACK)
rather than dropping it.

**Sudden death:** the process will also be SIGKILLed sometimes. Jobs must be
reentrant — idempotent, or transactional — so re-running one is safe.

**Deviation looks like:**
- No SIGTERM handler at all, so deploys drop in-flight requests.
- Startup that blocks on migrations, cache warming, or a slow dependency check.
- Jobs that are not idempotent and have no transaction around them, so a retry
  double-charges or double-sends.
- Cleanup that only runs on graceful shutdown, with no repair path at startup.
- A readiness probe that reports ready before the app can actually serve.

**The test:** SIGKILL a worker mid-job and re-run. If the outcome differs from a
single clean run, the job isn't reentrant.

## X. Dev/prod parity — keep environments as similar as possible

Twelve-factor names three gaps and argues for closing all of them: **time**
(hours between write and deploy, not weeks), **personnel** (the people who write
it deploy it), and **tools** (the same backing services everywhere).

The tools gap is the one that still bites. SQLite locally and Postgres in
production means your tests exercise different SQL semantics, different type
coercion, different transaction behaviour, and different failure modes than
production.

**Deviation looks like:**
- Different database engines between local and production.
- An in-memory or fake implementation of a backing service used in the test suite
  with no integration test against the real one.
- Local development that can't run the full stack, so the first real integration
  happens in staging.
- Long-lived release branches; batched releases.

**Legitimate reasons to deviate:** fakes and in-memory doubles in *unit* tests are
correct — speed matters and that's what they're for. The deviation is having no
layer that runs against the real thing. Managed services with no local equivalent
(a cloud queue, a vendor API) are a real constraint; the mitigation is a
contract test against the real service in CI, not a fake you trust blindly.

## XI. Logs — treat logs as event streams

The app writes plain text, one event per line, to `stdout`, unbuffered. It does
not open log files, rotate them, or decide where they go. Capture, aggregation,
routing and retention are the execution environment's job.

**Deviation looks like:**
- The app opening a log file, configuring rotation, or writing to a fixed path.
- Shipping logs from inside the app — an SDK that posts to a log service directly.
  That couples your app to a vendor and buffers events inside a process that can
  die.
- Buffered stdout, so logs arrive late or are lost on SIGKILL.
- Log lines that span multiple lines without structure — a raw traceback in a
  line-oriented collector becomes forty unrelated events.

**The modern refinement.** Twelve-factor says plain text; today, structured JSON
to stdout is generally better — one object per line, with a request/correlation id
on every event. It keeps the "one event per line" property while making the stream
queryable. Emitting human-readable lines in development and JSON in production is
a reasonable and common departure.

The other refinement is that logs are now one of three signals, alongside metrics
and traces. Twelve-factor predates that split; see the additions section.

## XII. Admin processes — run one-off tasks against the same release

Migrations, backfills, a REPL against live data, a one-time repair script. These
run in the same environment, against the same release and config, as the app's
long-running processes.

Admin code ships with application code, so it cannot drift out of sync with the
schema and models it operates on.

**Deviation looks like:**
- Migrations run from a developer's laptop against production.
- Ad-hoc SQL pasted into a console, unversioned and unreviewed, with no record of
  what ran.
- Scripts that live outside the repo, or in a separate repo on a different
  release cadence.
- An admin task using a different dependency set, a different config source, or a
  different database user with wider grants than it needs.

---

## What has aged, and what has been added

Being able to say *"this factor is dated"* is as useful as being able to say
*"this violates factor VI."*

**Aged badly:**

- **Secrets in environment variables (III).** Covered above. The rest of factor
  III is intact; this specific mechanism is not.
- **"Multiple apps sharing code is a violation" (I).** Written before monorepos
  were normal. The real requirement is independent buildability and deployability.

**Aged into being someone else's job:** port binding (VII), the process model
(VIII), and log routing (XI) are now largely handled by the container runtime and
orchestrator. They're still correct — they're just satisfied by the platform
rather than by your code. The failure mode has shifted from "did you implement
this?" to "did you fight the platform that already does it?"

**Aged perfectly:** config separation, stateless processes, disposability,
build/release/run immutability, dev/prod parity. Fifteen years on, these are still
where production incidents come from.

**Commonly added since** — from Kevin Hoffman's *Beyond the Twelve-Factor App* and
general practice:

- **API first.** Design and publish the contract before the implementation, so
  consumers can build against it and it can be tested independently.
- **Telemetry.** Logs alone are not observability. Metrics (rates, errors,
  durations) and traces (a request's path across services) are separate signals
  with separate tooling. Emit all three; treat instrumentation as a feature, not
  as debugging residue.
- **Authentication and authorization as a first-class concern**, designed in from
  the start rather than layered on at the edge.
- **Health checks as a contract.** Liveness ("am I wedged?") and readiness ("can I
  serve?") are different questions and need different endpoints. Conflating them
  causes restart loops under dependency failure.
- **Graceful degradation.** Twelve-factor assumes backing services are available.
  Timeouts, retries with backoff and jitter, circuit breakers, and a defined
  degraded mode are now table stakes.

---

## Worked example — auditing a typical service

A composite of the pattern that shows up most often: a FastAPI or Express service
with Postgres, deployed as a container. Not a specific repository — the point is
the shape of the audit, and these are the deviations that recur.

Order the audit by blast radius, not by factor number.

**Start with the four that cause incidents.**

*III — config.* Grep the tracked tree and the git history for credentials.
Then grep for `if.*env.*==.*prod` and equivalents; every hit is code that knows
which deploy it is. Check every config default: a default that works in
development and is wrong in production is worse than a missing one, because it
fails silently. Confirm secrets come from a mounted file or a secrets manager, not
from the environment directly.

*VI — statelessness.* Look for module-level mutable containers, in-process
caches, uploads written to local disk, and `@app.on_event("startup")` handlers
that build state. Then run two replicas and kill one under load; anything stateful
surfaces immediately. This is the single highest-yield check in the audit.

*IX — disposability.* Is there a SIGTERM handler? Does it stop accepting
connections and drain, or just exit? Are jobs idempotent — is there a transaction
or an idempotency key? Time a cold start; if it's over a few seconds, find out
what's blocking and whether it belongs in the build stage instead.

*XI — logs.* Confirm stdout only, unbuffered, one event per line, with a
correlation id threaded through. Look for any logging handler writing to a path,
and for vendor SDKs shipping logs from inside the process.

**Then the structural ones.**

*V — build/release/run.* Look at the entrypoint. Anything that installs, compiles,
downloads, or migrates at startup belongs earlier. Check whether image tags are
immutable — a moving `:latest` means you can't say what's running. Check that
rollback is selecting a previous release, not rebuilding one.

*XII — migrations.* The most common single deviation in this shape of service is
migrations running from an application startup hook. Every replica races for the
advisory lock, startup time becomes a function of migration time (breaking IX),
and a rollback can't easily reverse the schema. Move them to a one-off process
against the same release, run between release and run.

*IV — backing services.* Grep for hostnames, ports and bucket names outside
config. Look for provider-specific branches around storage or mail.

*II — dependencies.* Is there a lockfile, and is it enforced in CI? Then grep for
`subprocess`, `exec`, `system` — every shelled-out binary is an undeclared
dependency unless it's installed in the image explicitly.

*VIII — concurrency.* Is background work running inside the web process? Look for
in-process schedulers and `BackgroundTasks`-style fire-and-forget on work that
matters. If losing it on restart would be a problem, it needs a real queue.

*X — parity.* Does the test suite run against real Postgres anywhere, or only
against SQLite and fakes? Can a developer run the whole stack locally?

*I and VII* are usually satisfied by the repository layout and the container
image, and rarely need attention.

**Writing it up.** For each deviation: name the factor, state the concrete failure
it produces ("in-memory rate limiter — each replica enforces its own limit, so the
effective limit is N× the configured one"), and say whether it looks deliberate.
Deviations with a written reason are decisions, and should be left alone unless
the reason no longer holds. Deviations without one are the backlog.
