# Designing a Python package

Guidelines for approaching a new package or module, distilled from ten libraries
that got it right: **flask**, **fastapi**, **requests**, **httpx**, **pydantic**,
**attrs**, **click**, **rich**, **sqlalchemy**, **polars**.

**These are guidelines, not rules.** Every one of them is a default worth
departing from with a reason. Several of the exemplars contradict each other —
requests gives its exceptions a single root, attrs deliberately gives them none —
and both are right for their situation. The value is in the reasoning, not the
verdict.

One caveat to carry throughout: a lot of published-package machinery exists
because those projects **cannot reach their callers**. Deprecation cycles,
tombstone modules, vendored previous majors, stable error-code URLs. If you know
every consumer of your package, weigh that ceremony against what it actually buys
you. Copying the reasoning is the point; copying the ritual is how a package gets
ornate for free.

The seven sections follow the order you actually make the decisions.

---

## 1. Name the domain first

**1.1 — Let the module docstring state the constraint, not the contents.**

httpx opens `_exceptions.py` by drawing its entire exception tree in ASCII,
indentation-nested. That drawing *is* the design doc, and it lives with the
classes rather than in a wiki that will rot.

The contents of a package are derivable from its file tree. The invariant is not
— and the invariant is the thing a future change will quietly violate.

**1.2 — Name modules after domain nouns, not layers.**

requests: `sessions`, `adapters`, `auth`, `cookies`, `models`, `status_codes`.
Flask: `blueprints`, `ctx`, `globals`, `templating`, `signals`, `views`.
polars goes further and makes `dataframe/`, `lazyframe/`, `expr/`, `series/` into
*packages*, because those objects run 10k–13k lines each.

Both requests and Flask do keep a `utils.py` / `helpers.py` — as leaves, never as
the spine. A `utils.py` is fine. A `utils.py` that grew into the biggest file in
the package is the package telling you it never had a domain.

**1.3 — One package, one vocabulary.**

SQLAlchemy splits `sql/` (Core expression language) from `orm/` (unit of work):
different vocabularies, so different subpackages, each with its own thin front
door — `orm/__init__.py` is 171 lines over a 4.8 MB subtree.

If reading the package requires two glossaries, consider whether it's two
packages.

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

`__init__.py` is what a reader loads first to learn the package. Fifteen names
teach the shape; two hundred hide it.

**2.2 — Re-export explicitly.**

```python
from .core import Command as Command   # PEP 484 explicit re-export
```

click's entire `__init__.py` is 62 lines of that form; Flask's is 62 lines of it
too. httpx instead hand-maintains an `__all__` of 67 names. Either form tells a
type checker the name is public API — a bare `from .x import Y` does not.

Avoid `from .x import *` at the front door.

**2.3 — Consider two layers of `__all__`.**

httpx's private modules each declare their own `__all__` (`_client.py`'s is
exactly `["USE_CLIENT_DEFAULT", "AsyncClient", "Client"]`), which is what makes
the package-level star-imports safe. Module `__all__` gates what leaks upward;
package `__all__` is the contract.

**2.4 — Think twice before exporting a type alias.**

httpx defines `URLTypes`, `HeaderTypes`, `QueryParamTypes`, `TimeoutTypes` in
`_types.py` — and that module's `__all__` is only
`["AsyncByteStream", "SyncByteStream"]`. The unions were deliberately
un-exported. requests' `_types.py` says it outright: *"These types are not part of
the public API and must not be relied upon by external code."*

An exported alias becomes the thing everyone annotates with. Widening one input
then turns into an edit everywhere instead of a one-line change.

**2.5 — Use `_` prefixes as a signal, not a wall.**

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
breaking anyone. Whether it's worth the effort depends entirely on who your
callers are. What's cheap and always worth it is the signal itself: a
`_`-prefixed module tells the next reader *this is not load-bearing, change it
freely*.

## 3. Build three altitudes

**3.1 — Implement the convenience layer *as* the layer below it.**

`requests.api.request` is nine lines:

```python
def request(method, url, **kwargs):
    with sessions.Session() as session:
        return session.request(method=method, url=url, **kwargs)
```

polars does the same across a language boundary. `DataFrame.filter`, in full:

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

Two user-facing surfaces over one implementation means drift is structurally
impossible rather than merely tested against. This is probably the
highest-leverage idea in this document — it costs nothing at design time and a
rewrite later.

**3.2 — Document the shortcut's cost where it's taken.**

requests, in a code comment rather than an FAQ:

> By using the `with` statement we are sure the session is closed, thus we avoid
> leaving sockets open which can trigger a ResourceWarning in some cases, and look
> like a memory leak in others.

The reader who needs this is reading the source, not the docs.

**3.3 — Put the escape hatch in the docstring of the thing you'd subclass.**

`HTTPAdapter`'s docstring is a three-line runnable example of mounting it.
`BaseTransport.handle_request` opens with *"Developers shouldn't typically ever
need to call into this API directly"* — a public method labelled
implementer-facing rather than caller-facing.

**3.4 — Don't overload `None` for "unset".**

Four packages, four sentinels: httpx `USE_CLIENT_DEFAULT`, rich `NoChange`,
attrs `NOTHING`, pydantic `PydanticUndefined`.

rich's reason is the clearest: `justify=None` already *means* something ("inherit
the console default"), so "not specified" needs its own value. attrs makes
`NOTHING` a `Literal[_Nothing.NOTHING]` enum member so it narrows in a type
checker. httpx exports its sentinel so it can appear in signatures, while the
docstring says *"user code shouldn't need to use the USE_CLIENT_DEFAULT
constant."*

Applies to any three-state parameter. The bug it prevents is silent and gets
found in production.

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

Measure a candidate seam by that ratio. If the interface is as big as what sits
behind it, it isn't a seam — it's a rename.

**4.2 — Prefer an abstract base class; reach for `Protocol` only for duck typing.**

An ABC is the better default. It's discoverable (`help()` shows it, IDEs jump to
it), it gives you a place to put shared behaviour and docstrings, `abstractmethod`
fails loudly at instantiation rather than silently at the first call, and
`isinstance` works without ceremony.

requests' `BaseAdapter` is a plain base class with two `NotImplementedError`
stubs. Flask's `JSONProvider` is an ABC with `dumps` / `loads` / `response`, and
`DefaultJSONProvider` subclasses it. Both are the shape to copy: a small base
class, a first-party implementation of it, and a docstring on each method saying
what an implementer must guarantee.

`Protocol` earns its place when you genuinely cannot require inheritance —
third-party types you don't control, or objects that should participate by shape
alone. rich is the clean example: `__rich__`, `__rich_console__`,
`__rich_measure__`, `__rich_repr__` behind `runtime_checkable` Protocols, so any
object can render without importing rich at all. That's duck typing, and it's the
right tool there. It is not the right tool for "my package needs a storage
backend."

**4.3 — Ship the test double next to the seam.**

httpx ships `MockTransport` as a first-party implementation of the *public*
transport interface — which is why `Client(transport=MockTransport(handler))` is
the sanctioned testing story rather than `unittest.mock.patch`. click ships
`click.testing.CliRunner`. polars ships `polars.testing.assert_frame_equal`.
SQLAlchemy ships `sqlalchemy/testing/` **in the wheel**, including
`testing/suite/` — a conformance suite third-party dialect authors run against
their driver. The extension point comes with a certification harness.

If you define an interface, define a fake for it in the same module. Testability
is public API, not an afterthought for consumers to reinvent.

**4.4 — Extension should be overriding one method, not registering a plugin.**

Flask declares its pluggable classes as *class attributes* —
`json_provider_class`, `response_class`, `url_rule_class`, and even
`test_client_class`. Extension means subclass and reassign. No registry, no
config, no entry points.

FastAPI: subclass `APIRoute`, override `get_route_handler()`, set
`router.route_class` — you can now wrap every handler without middleware. Its
testing seam, `app.dependency_overrides`, is a plain `dict[Callable, Callable]`
keyed by the original callable: zero API.

A registry earns its keep when the implementations are genuinely unknown at build
time. Otherwise a subclass or an overridable method is less to learn and less to
maintain.

If you do dispatch on duck typing, rich's `protocol.py` is worth stealing
verbatim:

```python
_GIBBERISH = """aihwerij235234ljsdnp34ksodfipwoe234234jlskjdf"""

def rich_cast(renderable: object) -> "RenderableType":
    rich_visited_set: Set[type] = set()   # prevent potential infinite loop
    while hasattr(renderable, "__rich__") and not isclass(renderable):
        if hasattr(renderable, _GIBBERISH):   # objects that claim every attribute
            return repr(renderable)
```

Duck-typed dispatch has two failure modes at scale — objects that answer
`hasattr` for everything (`Mock`, proxies), and chains that cycle. Both handled
explicitly, in five lines.

## 5. Errors are an API

**5.1 — Decide on a root, and inherit the stdlib exception that fits.**

`RequestException(IOError)` — so pre-existing `except IOError` code keeps
working. Then diamonds, on purpose: `ConnectTimeout(ConnectionError, Timeout)` is
catchable as either; `MissingSchema(RequestException, ValueError)` and
`InvalidURL(RequestException, ValueError)` so naive `except ValueError` around
URL handling still fires.

attrs goes the *other* way and has no common root at all —
`FrozenInstanceError(AttributeError)`, because per its own docstring: *"It mirrors
the behavior of namedtuples by using the same error message and subclassing
AttributeError."* Substitutability beat catchability, deliberately.

Both are defensible. Pick the tradeoff consciously and write the reason in the
docstring; silence is the only wrong answer here.

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

This is the split that matters most. A retry loop that catches your own misuse
retries forever.

**5.3 — Make the message failure + context + what to do.**

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
  `before_request`, `register_blueprint` — converts a class of silent,
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

Any error crossing a package boundary should be branchable without string
matching.

## 6. Types are the contract

**6.1 — Ship `py.typed`.**

All ten exemplars do. requests shipped it in 2.34.0 after a decade of relying on
typeshed, and treated it as user-visible API — the type changes in the two
following patch releases each got a changelog entry with migration advice.

A package without `py.typed` is invisible to a type checker from the outside.
Every import from it is `Any`, silently. It's a one-byte file.

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

Cheap, fast, and it catches contract regressions no runtime test can.

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
`mapped_column` — until PEP 484 could express it. A type-checker plugin is often
a signal that the API isn't expressible.

## 7. Docs the compiler can't check

**7.1 — Make the examples real code.**

FastAPI keeps **77 directories** of tutorial code under `docs_src/` as importable
modules, imported by the test suite and measured for coverage:
`[tool.coverage.run] source = ["docs_src", "tests", "fastapi"]`. An uncovered
tutorial shows up as a coverage gap.

rich takes the lighter version: 47 of its 78 modules end in a runnable
`if __name__ == "__main__":` demo, so `python -m rich.table` renders a table.
Executable documentation that doubles as a smoke test.

An example that is never run is a comment with syntax highlighting.

**7.2 — Document the constraint the implementer will get wrong.**

click's `ParamType` docstring enumerates five obligations, and two of them are
precisely the ones people miss:

> - `convert` must accept values that are already the correct type.
> - It must be able to convert a value if the `ctx` and `param` arguments are
>   `None`. This can occur when converting prompt input.

When you write an ABC, this is what its method docstrings are for.

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

---

## Worked example — `requests`, end to end

`requests` is small enough to walk through all seven sections, and it makes most
of the choices above visibly.

```
requests/
  __init__.py      21 curated names
  api.py           get/post/put/… — 158 lines, ~90% docstring
  sessions.py      Session: cookies, pooling, defaults
  adapters.py      BaseAdapter (the seam) + HTTPAdapter (700+ lines)
  models.py        Request, PreparedRequest, Response
  auth.py          AuthBase — callable, one method
  cookies.py  structures.py  status_codes.py  hooks.py
  _types.py  _internal_utils.py       (private)
```

**1 · Domain.** Twenty flat modules, zero nesting, every name a noun from HTTP:
`sessions`, `adapters`, `auth`, `cookies`, `status_codes`. Nothing is called
`core` or `base` or `manager`. `utils.py` exists but is a leaf, not the spine.

**2 · Surface.** Twenty-one exports out of a codebase with hundreds of callables
— and only ten of the ~24 exception classes, chosen as the ones you'd realistically
catch. `requests.utils`, `requests.adapters` and `requests.auth` are documented
public modules, so "no underscore" genuinely means public; `_internal_utils.py`
and `_types.py` carry the prefix and mean it.

**3 · Altitudes.** Four rungs, each a strict superset, and the top one is
*implemented as* the next:

| rung | entry | buys you |
|---|---|---|
| 1 | `requests.get(url)` | works; closes the socket |
| 2 | `Session()` | cookie persistence, pooling, defaults |
| 3 | `Session.mount(prefix, HTTPAdapter(max_retries=…))` | retries, pool sizing, per-host routing |
| 4 | subclass `BaseAdapter` / override `HTTPAdapter.init_poolmanager` | replace the transport |

Rung 1 is nine lines that open a `Session` and delegate — the shortcut cannot
behave differently from the thing it wraps. The cost of rung 1 is stated in a code
comment right where it's paid (§3.2). `head()` differs from the generic path by
exactly one line, `kwargs.setdefault("allow_redirects", False)`, and the docstring
calls the divergence out rather than letting it surprise you.

**4 · Seam.** `BaseAdapter` is two methods, `send()` and `close()`. Seven hundred
lines of `HTTPAdapter` sit behind it, and the whole ecosystem — mocking,
caching, request signing — plugs in there. It's a base class with
`NotImplementedError` stubs, not a Protocol, and `HTTPAdapter` ships as the
reference implementation of its own interface (§4.2).

`Session.mount()` keeps adapters in an `OrderedDict` sorted by prefix length, so
`mount("http://api.example.com", …)` wins over `mount("http://", …)` — the
routing rule is a property of the data structure rather than a special case.

`AuthBase.__call__(r: PreparedRequest) -> PreparedRequest` is duck typing done
right: any callable works, and it predates `Protocol` entirely.

The `prepare_request()` / `send()` split is public, so you can inspect or mutate a
fully-realized request before it goes out. Most HTTP clients hide that boundary.

Restraint worth noting: `hooks.py` is 33 lines and `HOOKS = ["response"]`, with a
candid `# TODO: response is the only one`. They shipped a one-event hook system
instead of an extensible-in-principle registry nobody would use (§4.4).

**5 · Errors.** One root, `RequestException(IOError)`, so old `except IOError`
code survives. Diamonds where they help the caller:
`ConnectTimeout(ConnectionError, Timeout)`, `MissingSchema(RequestException,
ValueError)`. `__init__` attaches `.request` and `.response` and back-fills one
from the other (§5.4). Every leaf docstring is one line describing the
*situation*, not the class — `ConnectTimeout`: *"Requests that produced this error
are safe to retry."* Warnings get their own parallel root, `RequestsWarning`.

`JSONDecodeError` multiply-inherits `InvalidJSONError` and the stdlib
`JSONDecodeError`, and overrides `__reduce__` so pickling picks the right parent's
reducer — with a six-line comment explaining the MRO trap. That comment is §7.3
in action.

**6 · Types.** `py.typed` plus a private `_types.py` whose docstring says it is not
public API. The facade problem — how do you type seven verbs that all forward
`**kwargs` — is solved with PEP 692 rather than duplicating a fifteen-parameter
signature seven times:

```python
def request(method: str, url: _t.UriType, **kwargs: Unpack[_t.RequestKwargs]) -> Response: ...
def get(url: _t.UriType, params: _t.ParamsType = None, **kwargs: Unpack[_t.GetKwargs]) -> Response: ...
```

`BaseRequestKwargs(TypedDict, total=False)` holds the shared keys; `GetKwargs`,
`PostKwargs` and friends add only what that verb accepts — so the per-verb
divergence that used to live in prose now lives in the type system. Duck-typed
inputs get `runtime_checkable` Protocols (`SupportsRead`, `SupportsItems`), and
`is_prepared()` returns `TypeIs[_ValidatedRequest]` to narrow a lifecycle, with a
docstring that admits the Liskov violation instead of hiding it (§6.5).

**7 · Docs.** Docstrings are Sphinx field-lists with cross-references, so
`requests.api.request`'s docstring *is* the API reference page. `api.py` is ~90%
docstring by volume, which is the right ratio for a facade module. Tests run
against a real HTTP server and a real TLS server with a purpose-built certificate
that has a `commonName` but no `subjectAltName` — because you cannot test an HTTP
library's certificate semantics with mocks, and they don't pretend otherwise.
