# Designing a Python package

Distilled from ten packages that got it right: **flask** 3.1, **fastapi** 0.135,
**requests** 2.32/main, **httpx** 0.28, **pydantic** 2.12, **attrs** 26.1,
**click** 8.3, **rich** 14.2, **sqlalchemy** 2.0.49, **polars** 1.44. Every claim
below was checked against installed source or the upstream repo, not recalled.

## How to read this

Most published-package machinery exists for one reason: **they cannot reach their
callers.** They can't grep your code, can't fix your call site, can't email you.
Deprecation cycles, tombstone modules, vendored previous majors, semver, stable
error-code URLs — all of it is the price of unreachable consumers.

In a monorepo you own every call site. So each rule here states:

> what the exemplar does → why it's true **for them** → what survives **for us** →
> what doesn't transfer

Copying the ceremony without the reason is how a package gets slow and ornate for
free. Copying the reasoning is the point.

The eight sections follow the order you actually make the decisions.

---

# Part 1 — What good looks like

## 1. Name the domain first

**1.1 — The module docstring states the constraint, not the contents.**

httpx opens `_exceptions.py` by drawing its entire exception tree in ASCII,
indentation-nested. That drawing *is* the design doc, and it lives with the
classes rather than in a wiki that will rot.

`nanzen_tools` does the same for an invariant:

> Seven, and only seven. […] a fifth *table* reader means either a fifth table or
> a tool that overlaps another, and both are decisions worth noticing.

*For us:* the docstring is the first thing a reader — human or agent — loads. The
contents are derivable from the file tree. The invariant is not, and it's the
thing a future change will violate.

**1.2 — Module names are domain nouns, not layer names.**

requests: `sessions`, `adapters`, `auth`, `cookies`, `models`, `status_codes`.
Flask: `blueprints`, `ctx`, `globals`, `templating`, `signals`, `views`.
polars goes further and makes `dataframe/`, `lazyframe/`, `expr/`, `series/` into
*packages*, because those objects are 10k–13k lines each.

Both requests and Flask do have a `utils.py` / `helpers.py` — as leaves, never as
the spine.

*For us:* a `utils.py` is fine. A `utils.py` that grew into the biggest file is
the package telling you it never had a domain.

**1.3 — One package, one vocabulary.**

SQLAlchemy splits `sql/` (Core expression language) from `orm/` (unit of work):
different vocabularies, so different subpackages, each with its own thin front
door — `orm/__init__.py` is 171 lines over a 4.8 MB subtree.

*For us:* if reading the package needs two glossaries, it's two packages.

## 2. Draw the public surface

**2.1 — Curate the front door. It is a teaching surface.**

- requests exports 21 names — and only **10 of its ~24 exceptions**, the ones
  you'd realistically `except`. `ProxyError`, `SSLError`, `RetryError` and the
  rest live one import deeper in `requests.exceptions`.
- rich exports **five**: `get_console`, `reconfigure`, `print`, `inspect`,
  `print_json`. Everything else is `from rich.table import Table`. The module tree
  *is* the API index.
- polars exports **208** — and pays for it with a deprecating `__getattr__` and a
  self-flagged leak: `from polars._utils.wrap import wrap_df  # TODO: remove need
  for importing wrap utils at top level`.

*For us:* `__init__.py` is what a reader loads first to learn the package.
Fifteen names teach the shape; two hundred hide it.

**2.2 — Re-export explicitly, always.**

```python
from .core import Command as Command   # PEP 484 explicit re-export
```

click's entire `__init__.py` is 62 lines of that form; Flask's is 62 lines of it
too. httpx instead hand-maintains an `__all__` of 67 names. Either tells a type
checker the name is public API — a bare `from .x import Y` does not.

Never `from .x import *` at the front door.

**2.3 — Two layers of `__all__`.**

httpx's private modules each declare their own `__all__` (`_client.py`'s is
exactly `["USE_CLIENT_DEFAULT", "AsyncClient", "Client"]`), which is what makes
the package-level star-imports safe. Module `__all__` gates what leaks upward;
package `__all__` is the contract.

**2.4 — Type aliases are not free to export.**

httpx defines `URLTypes`, `HeaderTypes`, `QueryParamTypes`, `TimeoutTypes` in
`_types.py` — and that module's `__all__` is only
`["AsyncByteStream", "SyncByteStream"]`. The unions were deliberately
un-exported (deprecated 0.26, removed 0.28). requests' new `_types.py` says it
outright: *"These types are not part of the public API and must not be relied
upon by external code."*

Why: a published union freezes every accepted input shape forever.

*For us:* still holds, in a weaker form. An exported alias becomes the thing
everyone annotates with, and then widening one input is a repo-wide edit instead
of a one-line one.

**Doesn't transfer — `_`-prefix as enforcement.**

httpx has *zero* public modules; every file is `_`-prefixed. It then rewrites
`__module__` on every export so even `repr()` and tracebacks teach the supported
path:

```python
__locals = locals()
for __name in __all__:
    if not __name.startswith("__"):
        setattr(__locals[__name], "__module__", "httpx")
```

That machinery buys the freedom to split, merge or rename `_client.py` without
breaking strangers. You don't have strangers. What survives is the weaker, still
real signal: a `_`-prefixed module tells the next reader *this is not
load-bearing, change it freely*. Use it for that, not as a wall.

## 3. Build three altitudes

**3.1 — The convenience layer is *implemented as* the layer below it.**

`requests.api.request` is nine lines:

```python
def request(method, url, **kwargs):
    with sessions.Session() as session:
        return session.request(method=method, url=url, **kwargs)
```

polars does the same thing across a language boundary. `DataFrame.filter`, in
full:

```python
return (
    self.lazy()
    .filter(*predicates, **constraints)
    ._collect_eager(optimizations=QueryOptFlags._eager())
)
```

That pattern repeats at ten sites in `frame.py`. SQLAlchemy's async layer is the
sync core behind greenlets, not a second implementation; the ORM is built *on*
Core, not beside it.

Why: two user-facing surfaces, one implementation — drift becomes structurally
impossible rather than merely tested against.

*For us:* the highest-leverage rule in this document. It costs nothing at design
time and a rewrite later.

**3.2 — Document the shortcut's cost where it's taken.**

requests, in a code comment rather than an FAQ:

> By using the `with` statement we are sure the session is closed, thus we avoid
> leaving sockets open which can trigger a ResourceWarning in some cases, and look
> like a memory leak in others.

*For us:* the reader who needs this is reading the source, not the docs.

**3.3 — The escape hatch goes in the docstring of the thing you'd subclass.**

`HTTPAdapter`'s docstring is a three-line runnable example of mounting it.
`BaseTransport.handle_request` opens with *"Developers shouldn't typically ever
need to call into this API directly"* — a public method labelled
implementer-facing rather than caller-facing.

**3.4 — Never overload `None` for "unset".**

Four packages, four sentinels: httpx `USE_CLIENT_DEFAULT`, rich `NoChange`,
attrs `NOTHING`, pydantic `PydanticUndefined`.

rich's reason is the clearest: `justify=None` already *means* something ("inherit
the console default"), so "not specified" needs its own value. attrs makes
`NOTHING` a `Literal[_Nothing.NOTHING]` enum member so it narrows in a type
checker. httpx exports its sentinel so it can appear in signatures, while the
docstring says *"user code shouldn't need to use the USE_CLIENT_DEFAULT
constant."*

*For us:* applies to any three-state parameter. The bug this prevents is silent
and gets found in production.

## 4. Put the seam where the change happens

**4.1 — A seam is a narrow interface over a large implementation.**

- requests `BaseAdapter`: **two methods** (`send`, `close`). `HTTPAdapter` behind
  it is 700+ lines. Every mocking library, caching layer and request-signing shim
  in the ecosystem plugs in here.
- FastAPI `Depends` is a frozen dataclass with **three fields**. Behind it:
  sub-dependency graph resolution, caching, generator lifecycle, `AsyncExitStack`
  teardown, sync-vs-async detection, thread offloading — ~1500 lines.
- rich `Segment` is a 3-field `NamedTuple`. Every renderable yields Segments;
  every output backend consumes them.

*For us:* measure a seam by that ratio. If the interface is as big as what's
behind it, it isn't a seam — it's a rename.

**4.2 — The package defines the Protocol; the host implements it.**

Flask declares its pluggable classes as *class attributes* — `json_provider_class`,
`response_class`, `url_rule_class`, and even `test_client_class`. Extension means
subclass and reassign. No registry, no config, no entry points.

*For us:* `nanzen_canonical` owns `CanonicalDataLoader`; the backend owns the
tables and the connection, so the backend implements it. The dependency arrow
points at the abstraction, which is why the data layer never imports the app.

**4.3 — Ship the test double next to the seam.**

httpx ships `MockTransport` as a first-party implementation of the *public*
transport interface — which is why `Client(transport=MockTransport(handler))` is
the sanctioned testing story rather than `unittest.mock.patch`. click ships
`click.testing.CliRunner`. polars ships `polars.testing.assert_frame_equal`.
SQLAlchemy ships `sqlalchemy/testing/` **in the wheel**, including
`testing/suite/` — a conformance suite third-party dialect authors run against
their driver. The extension point comes with a certification harness.

*For us:* `nanzen_canonical.loader.StaticLoader` is exactly this. Testability is
public API, not an afterthought for consumers to reinvent.

**4.4 — Extension is overriding one method, not registering a plugin.**

FastAPI: subclass `APIRoute`, override `get_route_handler()`, set
`router.route_class` — you can now wrap every handler without middleware. Its
testing seam, `app.dependency_overrides`, is a plain `dict[Callable, Callable]`
keyed by the original callable: zero API. rich requires no inheritance at all —
four dunders (`__rich__`, `__rich_console__`, `__rich_measure__`,
`__rich_repr__`) behind `runtime_checkable` Protocols.

*For us:* prefer a Protocol or one overridable method to a registry. A registry
earns its keep only when the implementations are genuinely unknown at build time.

Worth stealing verbatim if you ever dispatch on duck typing, from rich's
`protocol.py`:

```python
_GIBBERISH = """aihwerij235234ljsdnp34ksodfipwoe234234jlskjdf"""

def rich_cast(renderable: object) -> "RenderableType":
    rich_visited_set: Set[type] = set()   # prevent potential infinite loop
    while hasattr(renderable, "__rich__") and not isclass(renderable):
        if hasattr(renderable, _GIBBERISH):   # objects that claim every attribute
            return repr(renderable)
```

Duck-typed dispatch has two failure modes at scale — objects that answer
`hasattr` for everything (`Mock`, proxies), and `__rich__` chains that cycle.
Both handled explicitly, both in five lines.

## 5. Errors are an API

**5.1 — One root, multiply-inheriting the stdlib exception that fits.**

`RequestException(IOError)` — so pre-existing `except IOError` code keeps
working. Then diamonds, on purpose: `ConnectTimeout(ConnectionError, Timeout)` is
catchable as either; `MissingSchema(RequestException, ValueError)` and
`InvalidURL(RequestException, ValueError)` so naive `except ValueError` around
URL handling still fires.

attrs goes the *other* way and has no common root at all —
`FrozenInstanceError(AttributeError)`, because per its own docstring: *"It mirrors
the behavior of namedtuples by using the same error message and subclassing
AttributeError."* Substitutability beat catchability, deliberately.

*For us:* pick the tradeoff consciously and write the reason in the docstring.
Both are defensible. Silence is not.

**5.2 — Split "your data was bad" from "you used this wrong".**

- pydantic: `ValidationError(ValueError)` for data; `PydanticUserError(TypeError)`,
  `PydanticUndefinedAnnotation(NameError)`, `PydanticImportError(ImportError)` for
  misuse — each multiply-inheriting the semantically correct builtin.
- FastAPI: `HTTPException`'s docstring opens *"This is for client errors […] **Not
  for server errors in your code.**"* `FastAPIError(RuntimeError)` is the other
  tree.
- httpx puts programmer errors *outside* `HTTPError` entirely:
  `StreamError(RuntimeError)`, docstring *"The developer made an error in
  accessing the request stream in an invalid way."*

*For us:* this is the split that matters most. A retry loop that catches your own
misuse retries forever.

**5.3 — The message is failure + context + what to do.**

- `raise_for_status()` produces `"404 Client Error: Not Found for url: https://…"`
  — status, reason **and url**. It decodes the reason with a latin-1 fallback
  because some servers localize their reason strings.
- `ConnectTimeout`'s entire docstring: *"Requests that produced this error are
  safe to retry."* An operational instruction inside an exception class.
- httpx's `StreamConsumed` names the *causes*, not the symptom: *"this could be
  due to passing a generator as request content, and then receiving a redirect
  response or a secondary request as part of an authentication flow."*
- FastAPI renders validation context in Python traceback format, so terminals and
  editors linkify it and jump you to the endpoint:

```python
context = f'\n  File "{self.endpoint_file}", line {self.endpoint_line}, in {self.endpoint_function}'
```

- Flask's `@setupmethod` — nine lines of decorator on `route`, `add_url_rule`,
  `before_request`, `register_blueprint` — converts a whole class of silent,
  thread-timing-dependent bugs into a loud error naming the offending method:
  *"The setup method 'route' can no longer be called on the application. It has
  already handled its first request, any changes will not be applied
  consistently."* Making a temporal contract mechanically enforceable is the
  transferable move.

**5.4 — Put the context on the object, not only in the string.**

`RequestException.__init__` attaches `.request` / `.response`, and back-fills
`.request` from `response.request` when only the response was passed.

pydantic's `ValidationError` is the case study:

- **every** failure is reported, not the first;
- `loc` is a path tuple — `('b', 0)` renders as `b.0`, directly mappable to a form
  field or JSON pointer;
- `type` is a stable slug, documented as *"an identifier designed for programmatic
  use that will change rarely or never"* — you branch on `type`, never on `msg`;
- `msg` and `ctx` are separate, so the message can be re-rendered in another
  language;
- `input` is echoed back, so the error is self-contained;
- and it's privacy-tunable: `errors(include_url=…, include_context=…,
  include_input=…)`. `include_input=False` is how you keep PII out of logs.

*For us:* any error crossing a package boundary should be branchable without
string matching.

**Doesn't transfer — stable error-code URLs.** SQLAlchemy gives every exception a
four-character code (`IntegrityError` = `gkpj`, `MissingGreenlet` = `xd2s`)
rendered as `https://sqlalche.me/e/20/gkpj`, with the version token stamped in at
import so the link resolves to the docs for the version actually installed. It
exists so message text can be reworded without invalidating a decade of Stack
Overflow answers and users' alerting greps. You can reword yours and fix the greps
in the same commit.

## 6. Types are the contract

**6.1 — Ship `py.typed`.**

All ten exemplars do. requests shipped it in 2.34.0 after a decade of relying on
typeshed, and treated it as user-visible API — the type changes in 2.34.1 and
2.34.2 each got a changelog entry with migration advice.

*For us:* a package without `py.typed` is invisible to a type checker from the
outside. Every import from it is `Any`, silently. It's a one-byte file.

**6.2 — Keep the type vocabulary private and central.**

requests `_types.py`, httpx `_types.py`, Flask `typing.py` — the last being 94
lines with no runtime code, and where Flask's real contract actually lives:

```python
ResponseReturnValue = t.Union[
    ResponseValue,
    tuple[ResponseValue, HeadersValue],
    tuple[ResponseValue, int],
    tuple[ResponseValue, int, HeadersValue],
    "WSGIApplication",
]
```

Every non-obvious alias in that file carries a comment explaining the typing
limitation behind it, several with issue links.

**6.3 — Overloads for shape-varying calls; `ParamSpec` for decorators.**

click's `command` has four overloads, one per legal call shape, with
`CmdType = TypeVar("CmdType", bound=Command)` so `@command(cls=MyCommand)` yields
a `MyCommand`, not a `Command`. And `pass_context` models "the decorator eats the
first argument" exactly:

```python
def pass_context(f: t.Callable[te.Concatenate[Context, P], R]) -> t.Callable[P, R]:
```

SQLAlchemy's `select()` carries 23 overloads. `DBAPIError.instance()` has three,
so the non-wrapping paths (`KeyboardInterrupt`, `DontWrapMixin`) type-check as
themselves rather than as the wrapper.

**6.4 — Test your types.**

Flask keeps `tests/type_check/` — `.py` files that are **never executed**, checked
by mypy strict and pyright in CI. `typing_route.py` asserts that every legal
`ResponseReturnValue` shape compiles: `str`, `bytes`, `Response`, `dict`, a
`TypedDict`, a generator, `tuple[str, int]`, `async def`, `View.as_view`. click
has `tests/typing/` under both checkers. pydantic runs `tests/typechecking/`
under mypy *and* pyright, mandating `assert_type()` for positives and paired
ignore comments for negatives.

*For us:* free, fast, and it catches contract regressions no runtime test can.
The cheapest high-value item on this list.

**6.5 — When the type system loses, say so in the source.**

- attrs: `# NOTE: Factory lies about its return type to make this possible:
  x: List[int] = Factory(list)`.
- attrs: `def fields(cls) -> Any` — the return is a per-class generated tuple, so
  `Any` instead of a fiction.
- pydantic documents its own `TypeAdapter` mypy failure *in the class docstring*,
  with the exact `# type: ignore[arg-type]` workaround.
- rich sets `enable_error_code = ["ignore-without-code"]`, banning bare
  `# type: ignore`.

The deeper lesson is SQLAlchemy's: `ext/mypy/`, its 1.4-era type-checker plugin,
still ships but is deprecated. 2.0 reshaped the runtime API — `Mapped[T]`,
`mapped_column` — until PEP 484 could express it. **A type-checker plugin is a
signal your API isn't expressible.**

## 7. Docs the compiler can't check

**7.1 — Make the examples executable.**

- requests: `addopts = "--doctest-modules"`. Docstring examples are contracts.
- polars runs `tests/docs/run_doctest.py` over every docstring example, including
  the ones showing rendered box-drawn tables. Its exception classes are defined
  twice — once imported from Rust, once in pure Python under `except ImportError`
  *"for documentation purposes when there is no binary"* — so the docs build
  without a compiler and Sphinx still sees real docstrings.
- FastAPI: `[tool.coverage.run] source = ["docs_src", "tests", "fastapi"]`. **77
  directories of tutorial code, measured for coverage.** An uncovered tutorial
  shows up as a coverage gap.
- pydantic collects fenced snippets from inside docstrings and asserts their
  `#> output` blocks — which is why `AfterValidator`'s docstring can contain a
  full `ValidationError` dump that cannot rot.
- rich: 47 of its 78 modules end in a runnable `if __name__ == "__main__":` demo.
  `python -m rich.table` works.

*For us:* an example that isn't run is a comment with syntax highlighting.

**7.2 — Document the constraint the implementer will get wrong.**

click's `ParamType` docstring enumerates five obligations, and two of them are
precisely the ones people miss:

> - `convert` must accept values that are already the correct type.
> - It must be able to convert a value if the `ctx` and `param` arguments are
>   `None`. This can occur when converting prompt input.

**7.3 — Record *why*, especially where the code looks wrong.**

requests keeps a compatibility shim it dislikes, and labels it rather than hiding
it:

```python
# This code exists for backwards compatibility reasons.
# I don't like it either. Just look the other way. :)
```

FastAPI's ruff config encodes a domain constraint: `keep-runtime-typing = true`,
because FastAPI *evaluates annotations at runtime* — so the usual "modernize type
syntax" autofix would be a correctness bug.

SQLAlchemy's `util/preloaded.py` names the real reason its imports are hoisted,
which is not the obvious one:

> To avoid potential thread safety issues for imports that are deferred in a
> function, like https://bugs.python.org/issue38884, these modules are added to
> the system module cache by importing them after the packages has finished
> initialization.

Not "to break cycles" — a function-body import under contention can hand out a
half-initialized module.

**7.4 — Make import cost a tested contract, if you're the kind of package where it
matters.**

click's `tests/test_imports.py::test_light_imports` spawns a subprocess with
`builtins.__import__` monkeypatched, records every module click pulls in, and
asserts the set is a subset of a 26-name stdlib allowlist. pydantic runs a
subprocess test asserting `pydantic.deprecated`, `pydantic.fields` and
`pydantic.types` are **not** in `sys.modules` after `from pydantic import
BaseModel` — guarding its lazy `__getattr__` router against silent degradation.

*For us:* worth it when a package is imported by a CLI, a cold-start worker, or a
test suite that pays the cost thousands of times. Not otherwise. Know which you
are.

## 8. Changing a package others depend on

The graph, as of writing:

```
nanzen_canonical   ← (nothing)
nanzen_metrics     ← canonical
nanzen_audit       ← canonical
nanzen_tools       ← canonical, metrics, audit, common_tools, connectors
agent_core         ← common_tools, llm
nanzen_agents      ← agent_core, canonical, metrics, tools, common_tools
```

**8.1 — Know your dependents before you change a signature.**

```bash
grep -l '"nanzen-metrics"' packages/*/pyproject.toml apps/*/pyproject.toml
```

This is the whole reason we get to skip everything in 8.2. Use it.

**8.2 — A breaking change and every call site land in one PR. That is our
deprecation cycle.**

The published packages can't do that, and the cost is enormous:

- pydantic hand-catalogued **~190 removed symbols** in `_migration.py` so each one
  produces a specific `PydanticImportError` instead of a generic `AttributeError`
  — plus thirteen five-line tombstone modules that exist only so
  `import pydantic.utils` resolves and hits the router, plus `pydantic/v1/`: the
  entire previous major, vendored, with a script to re-sync it.
- SQLAlchemy used a whole release as a runway — `SQLALCHEMY_WARN_20=1` for
  *opt-in* deprecation noise, `create_engine(future=True)` to run 2.0 semantics
  inside 1.4, `__allow_unmapped__` to migrate file by file — and split its
  warnings into `RemovedIn20Warning` / `MovedIn20Warning` / `LegacyAPIWarning` so
  consumers could filter *removed*, *moved* and *legacy-but-supported* separately.
- click wrote a metaclass so a deprecated alias still passes `isinstance`:

```python
class _FakeSubclassCheck(type):
    def __subclasscheck__(cls, subclass): return issubclass(subclass, cls.__bases__[0])
    def __instancecheck__(cls, instance): return isinstance(instance, cls.__bases__[0])
```

None of that is a virtue. It is the price of unreachable callers. **Don't pay it
voluntarily.**

**8.3 — A cycle is a seam error, not an import problem.**

Both large exemplars needed real machinery and both treated it as structural:
polars added `_reexport.py` (23 lines — *"Re-export Polars functionality to avoid
cyclical imports"*); SQLAlchemy built a `preloaded` registry that imports ~55
modules *after* package init, declaring the same names under `if TYPE_CHECKING:`
so type checkers see through the indirection.

*For us:* at our scale a cycle means two packages are one package, or a Protocol
belongs in the lower one. Fix the design; don't reach for a function-body import.

**8.4 — If you do add runtime indirection, pay for it with a static mirror.**

polars guards its module-level `__getattr__` — which serves deprecated aliases —
behind a `TYPE_CHECKING` check, and says why:

```python
if not TYPE_CHECKING:
    # This causes typechecking to resolve any Polars module attribute
    # as Any regardless of existence so we check for TYPE_CHECKING, see #24334.
    def __getattr__(name: str) -> Any:
```

An unguarded module-level `__getattr__` makes *every* attribute on the package
type-check as `Any`, silently killing typo detection across every consumer.
SQLAlchemy pays the same tax the same way, declaring its ~55 preloaded modules as
real imports under `if TYPE_CHECKING:`.

**What we deliberately don't copy:** semver and changelogs (one `VERSION.txt`, one
repo), deprecation-warning vintages, tombstone modules, import shims, vendored
previous majors, stable error-code URLs, `_`-privacy as access control. Every one
of them exists to serve a caller you cannot reach.

---

# Part 2 — Applying it

## The short version

Starting a package, in order:

1. Write the `__init__.py` docstring **first**. State the boundary and the
   invariant — what this package is *not* responsible for is usually the more
   useful half.
2. Name modules after domain nouns. If you reach for `core.py`, the domain isn't
   named yet.
3. Decide what the front door exports. Default to *few*: the vocabulary, not the
   machinery. Use `from .x import Y as Y` or an explicit `__all__`.
4. Build the convenience API *on top of* the general one, in the same file, so the
   two can't drift.
5. Find the one thing that will change — the store, the transport, the model — and
   put a Protocol there. Ship a static/fake implementation of it beside the real
   one.
6. Decide the error split before you write the first `raise`: bad input vs. caller
   misuse. Attach context to the exception object.
7. Add `py.typed`. Keep type aliases in a private module. Add one non-executed
   type-check file asserting your public shapes.
8. Place the package in the dependency DAG before you add its first import. An
   edge that closes a cycle is a design error.

## Worked example — `nanzen_canonical`

Four modules and a facade. Small enough to walk end to end.

```
nanzen_canonical/
  __init__.py     vocabulary re-exports, 3 names
  vocabulary.py   Book, TransactionKind, ALL_COMPANIES
  data.py         CanonicalData — one company's four tables as a frozen value
  derive.py       the invoice ↔ settlement join
  loader.py       CanonicalDataLoader (the seam) + StaticLoader (the double)
```

**1 · Domain.** The docstring leads with the *boundary*, not the contents: the four
canonical sources are declared by the backend's Alembic chain, and *"that DDL is
the schema's only spelling."* The package holds *"everything about the model that
is not schema."* A reader who came here to add a column learns in the first
paragraph that they're in the wrong file — rule 7.2, applied to a package rather
than a Protocol.

**2 · Surface.** Three names: `Book`, `TransactionKind`, `ALL_COMPANIES`.
`CanonicalData` and `CanonicalDataLoader` are deliberately one import deeper.
That's rich's bet: the front door teaches the *vocabulary*, the module tree is the
index for the machinery. Most callers only ever need to name a book.

**3 · Altitudes.** `Book` / `TransactionKind` (closed value sets, no I/O) →
`CanonicalData` (a frozen value you can construct in a test with three lines) →
`CanonicalDataLoader` (the seam that touches a database). Each rung is optional;
a caller who only needs to name a book never meets a connection.

**4 · Seam.** `CanonicalDataLoader` is defined here, in the *lower* package, and
implemented by the backend — which owns the tables and the connection. The
dependency arrow points at the abstraction, so the data layer never imports the
app. `StaticLoader` ships beside it (rule 4.3), which is what makes every
downstream consumer — `nanzen_metrics`, `nanzen_audit`, `nanzen_tools`,
benchmarks — testable without a database.

**5 · Errors.** The package raises nothing of its own today, which is honest: it's
a vocabulary and a value type. When it needs to, rule 5.2 sets the split — *"the
frame you handed me is malformed"* (caller misuse) is a different exception from
*"there is no data for this company in this window"* (an expected data state), and
conflating them is how a retry loop spins forever.

**6 · Types.** The `Protocol` in `loader.py` *is* the type contract, and it's in
the right place — the consumer of an abstraction should own it. The natural next
step is rule 6.4: a five-line non-executed file asserting `StaticLoader` satisfies
`CanonicalDataLoader`, so an accidental signature change fails in CI rather than
at the first backend deploy.

**7 · Docs.** The docstring cross-references its own siblings with `:mod:` and
`:class:` roles, and names the migration that owns the schema
(`0049_canonical_tables`). Someone can navigate the whole package without opening
a second file.

**8 · Dependencies.** Zero. It is the root of the DAG, which is exactly why
`metrics`, `audit`, `tools` and `agents` can all depend on it without a cycle.
Adding a dependency here would be the most expensive edge in the repo — every
package above inherits it.
