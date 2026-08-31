# Python style

Guidelines for writing Python that reads the same way across a codebase, adapted
from the **[Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)**
— the most widely copied of the published Python styles, and the only one that
explains its reasoning rule by rule (each entry is argued as Definition / Pros /
Cons / Decision).

**These are guidelines, not rules.** The original closes with *"BE CONSISTENT"*
and immediately qualifies it: consistency applies most heavily locally, and is
not a justification for staying in an old style forever. Where a rule below is
worth departing from, the departure should be visible and explained, not silent.

One caveat to carry throughout: **a lot of this exists because Google is one
enormous monorepo with mixed-vintage code and no way to reach every caller.**
Full-path imports, lint-suppression conventions, license boilerplate, TODOs
that must link a tracked bug, `absl` and `pytype`. If your repo is small enough
to grep in full, weigh that ceremony against what it buys you. Copy the
reasoning; the ritual is optional.

Where the original names a Google tool, this version names ours: **[ruff](https://docs.astral.sh/ruff/)**
for both formatting and linting, **[ty](https://github.com/astral-sh/ty)** for
type checking.

Scope: how code is written, not how a package is designed — the decisions here
start after you know what you're building.

---

## 1. Settle the mechanics before writing a line

Nearly a third of this guide describes things a tool should decide for you. Set
the tools up once and the arguments stop.

**1.1 — `ruff format` so formatting is never reviewed.**

Google's own note is that many teams run an autoformatter *"to avoid arguing over
formatting."* `ruff format` is Black-compatible, so section 8 below is mostly a
description of its output. Read it to recognize good output, not to hand-apply
it.

**1.2 — `ruff check`, and suppress with a reason.**

Suppress narrowly, by rule code, on the line:

```python
def do_PUT(self):  # noqa: N802  # WSGI name.
```

The rule code is the point: suppressions become greppable and revisitable. A bare
`# noqa` suppresses everything on the line and hides the next problem — always
name the rule, and add the reason when the code doesn't explain itself.

For an argument you must accept but don't use, `del` it — don't rename it:

```python
def viking_cafe_order(spam: str, beans: str, eggs: str | None = None) -> str:
    del beans, eggs  # Unused by vikings.
    return spam + spam + spam
```

`_` and `unused_` prefixes are allowed but no longer encouraged: they break
callers who pass by keyword, and nothing enforces that the argument is actually
unused.

**1.3 — Run `ty` in CI, not by hand.**

Type annotations that nothing verifies rot. The position worth keeping: adding or
modifying a public API means annotating it *and* having the check run. Where a
project can't turn checking on yet, leave a TODO linking the blocker rather than
silence.

Suppress the same way you suppress lint — narrowly and by name, `# ty: ignore[rule]`
rather than a blanket `# type: ignore`.

## 2. Imports

**2.1 — Use full package paths. Never relative imports.**

```python
from doctor.who import jodie    # yes
import jodie                    # no — which jodie? depends on sys.path
```

Assume `import jodie` means a third-party top-level package, not the `jodie.py`
sitting next to you. Relative imports also risk importing a module twice under
two names.

**2.2 — Alias only for a reason.**

A genuine collision, a name that clashes with a public parameter, an
inconveniently long name, or a name too generic in context
(`from storage.file_system import options as fs_options`). Bare `import x as y`
only when the abbreviation is universal — `import numpy as np`.

**2.3 — `typing` and `collections.abc` names read as keywords.**

Import those symbols directly, several per line:

```python
from collections.abc import Mapping, Sequence
from typing import Any, Generic, cast, TYPE_CHECKING
```

The flip side of treating them as keywords: don't define your own `Sequence` or
`Any`. On a real collision, `from typing import Any as AnyType`.

**2.4 — One import per line, grouped generic → specific.**

`__future__`, then standard library, then third-party, then your repo's
sub-packages. Sorted lexicographically by full package path, ignoring case.
Blank lines between groups are optional.

Modern note: Google explicitly **deprecated** the old separate group for
same-top-level-package imports. New code treats its own sub-packages like any
other sub-package import.

## 3. Errors

**3.1 — Raise built-ins when a built-in fits.**

`ValueError` for a violated precondition is the canonical case. Reach for a
custom exception when callers need to catch *that* thing specifically.

**3.2 — Custom exceptions inherit, end in `Error`, and don't stutter.**

`ConnectionError`, not `ConnectionException`; `foo.ConnectionError`, not
`foo.FooError`.

**3.3 — No `assert` outside tests.**

Google allows `assert` for documenting invariants the code doesn't depend on. We
don't: **in non-test code, every check is a real conditional that raises.**

Assertions are stripped by `python -O`, so an `assert` is a check that may or may
not exist at runtime — and a reader can't tell from the call site which kind it
is. The rule removes that question.

```python
# Yes
if minimum < 1024:
    raise ValueError(f"Min. port must be at least 1024, not {minimum}.")
port = self._find_next_open_port(minimum)
if port is None:
    raise ConnectionError(f"Could not connect on port {minimum} or higher.")
return port
```

```python
# No — vanishes under -O, and the code below depends on it,
# including the return type.
assert minimum >= 1024, "Minimum port must be at least 1024."
port = self._find_next_open_port(minimum)
assert port is not None
return port
```

If the condition is "this should be impossible", raise anyway — `AssertionError`
or a project-specific `InvariantError` — so the failure is guaranteed to surface.

In pytest-based tests, `assert` *is* the idiom. That exemption doesn't travel
back into the code under test.

**3.4 — Never catch everything, with two exceptions.**

A bare `except:` or `except Exception:` is allowed only when you immediately
re-raise, or when you are deliberately building an isolation point — the
outermost block of a thread, a plugin boundary — where the exception is recorded
and suppressed on purpose. Otherwise you have also caught `sys.exit()`,
`KeyboardInterrupt`, misspelled names and your own test failures.

**3.5 — Keep the `try` body small.**

The bigger the `try`, the more likely it swallows an error from a line you never
thought could raise. Use `finally` for cleanup that must happen either way.

**3.6 — Write error messages that survive being grepped.**

Three requirements: the message must precisely match the actual condition,
interpolated pieces must be visibly interpolated, and the string must be easy to
search for.

```python
# Yes
if not 0 <= p <= 1:
    raise ValueError(f"Not a probability: {p=}")
except OSError as error:
    logging.warning(f"Could not remove directory {workdir!r}: {error!r}")
```

```python
# No
if p < 0 or p > 1:            # also false for float("nan")
except OSError:
    logging.warning(f"Directory already was deleted: {workdir}")   # assumes a cause
    logging.warning(f"The {workdir} directory could not be deleted.")  # hard to grep
```

That last one has a failure mode worth keeping: called with `workdir="deleted"`,
it logs *"The deleted directory could not be deleted."*

## 4. State, scope, and cleverness

**4.1 — Avoid mutable global state.**

Two costs, both concrete: it breaks encapsulation in ways that surface late (a
globally-held database connection makes "connect to two databases at once during
a migration" a rewrite), and assignments run at *import* time, so importing the
module changes behavior.

Where it's genuinely warranted, declare it at module level or as a class
attribute, prefix `_`, expose it only through functions or methods, and **write
down why** in a comment. Module-level constants are fine and encouraged —
`_MAX_HOLY_HANDGRENADE_COUNT = 3` internal, `SIR_LANCELOTS_FAVORITE_COLOR` public.

**4.2 — Nest a function to close over a value, not to hide it.**

Nested functions and classes can't be tested directly and make the enclosing
function longer. Closing over a local (other than `self`/`cls`) justifies it.
Hiding does not: prefix the name with `_` at module level so tests can still
reach it.

**4.3 — Lexical scoping is fine; know the trap.**

```python
i = 4
def foo(x):
    def bar():
        print(i, end="")
    for i in x:      # i is local to foo, so this is what bar sees
        print(i, end="")
    bar()
```

`foo([1, 2, 3])` prints `1 2 3 3`, not `1 2 3 4`. Any assignment to a name makes
it local for the whole block, including uses that precede the assignment.

**4.4 — Decorators: judiciously, and test them.**

A decorator runs at definition time — import time for anything module-level — and
a failure there is essentially unrecoverable. So: no external dependencies inside
a decorator (no files, sockets, database connections); it may run under `pydoc`.
Say in the docstring that it *is* a decorator. Write unit tests for it.

**4.5 — Never `staticmethod`. Rarely `classmethod`.**

`staticmethod` only to satisfy an existing library's API — otherwise write a
module-level function. `classmethod` for named constructors, or a class-specific
routine that touches necessary global state such as a process-wide cache.

**4.6 — Properties must behave like attributes.**

Cheap, straightforward, unsurprising. A property that just reads and writes an
internal attribute isn't earning its keep — make the attribute public. A property
that controls access or computes a *trivially* derived value is fine. Don't use
properties for computation a subclass might want to override; inheritance with
properties is non-obvious.

**4.7 — Getters and setters when the operation is non-trivial.**

The function-call syntax is the signal: it warns the reader that something more
than a load is happening (state invalidated, cache rebuilt, cost incurred). If a
getter/setter pair is a pure passthrough, make the attribute public. When
migrating a property to real getters/setters, *don't* keep the property bound —
let old access break visibly so callers learn the cost changed.

**4.8 — Avoid power features.**

Custom metaclasses, bytecode access, dynamic inheritance, object reparenting,
import hacks, `__del__`-based cleanup, reflection tricks. They compress code
today and cost disproportionately at debugging time. Standard-library machinery
built on them — `abc.ABCMeta`, `dataclasses`, `enum` — is fine to use.

**4.9 — Don't rely on atomicity in threads.**

Built-in types only *look* atomic; a Python-level `__hash__` or `__eq__` breaks
the illusion, and variable assignment depends on dicts anyway. Prefer
`queue.Queue` for passing data between threads; otherwise `threading` primitives,
preferring `threading.Condition` over raw locks.

## 5. Everyday expressions

**5.1 — Comprehensions: one `for`, and optimize for readability.**

Multiple `for` clauses or multiple filters are not permitted — write the loop.

```python
# Yes
result = [mapping_expr for value in iterable if filter_expr]
return {x: transform(x) for x in generate(param) if x is not None}

result = []
for x in range(10):
    for y in range(5):
        if x * y > 10:
            result.append((x, y))
```

```python
# No
result = [(x, y) for x in range(10) for y in range(5) if x * y > 10]
```

**5.2 — Use the default iterators and operators.**

`for key in adict`, `if obj in alist`, `for line in afile`, `for k, v in
adict.items()` — not `.keys()`, not `.readlines()`. They're simpler, faster, and
generic over anything supporting the protocol. Never mutate a container while
iterating it.

**5.3 — Generators: document with `Yields:`, and force cleanup.**

`Yields:` rather than `Returns:` in the docstring, describing what `next()`
returns — not the generator object. Locals in a generator aren't collected until
it's exhausted or itself collected, so a generator holding an expensive resource
should be wrapped in a context manager ([PEP 533](https://peps.python.org/pep-0533/)).

**5.4 — Lambdas for one-liners only.**

Over ~60–80 characters or more than one line, make it a named nested function —
anonymous frames make stack traces harder to read. For common operations use the
`operator` module: `operator.mul`, not `lambda x, y: x * y`. Prefer a generator
expression to `map()`/`filter()` with a lambda.

**5.5 — Conditional expressions: every part on one line.**

True-expression, condition, else-expression each fit on their own line, or use a
real `if` statement.

```python
one_line = "yes" if predicate(value) else "no"
the_longest_ternary_style_that_can_be_done = (
    "yes, true, affirmative, confirmed, correct"
    if predicate(value)
    else "no, false, negative, nay")
```

**5.6 — No mutable default arguments — and no computed ones.**

Defaults are evaluated once, at function definition time.

```python
def foo(a, b: Sequence | None = None):   # yes
    if b is None:
        b = []
def foo(a, b: Sequence = ()):            # yes — tuples are immutable
def foo(a, b=[]):                        # no
def foo(a, b=time.time()):               # no — time of module load
def foo(a, b=_FLAG.value):               # no — argv not parsed yet
```

**5.7 — Implicit false, with four caveats.**

Prefer `if foo:` over `if foo != []:` and `if seq:` over `if len(seq):`. But:

- `None` checks are always `is None` / `is not None`. The alternative value might
  itself be falsy.
- Never `== False`. Use `not x`; to distinguish `False` from `None`, chain:
  `if not x and x is not None:`.
- For integers, implicit false risks silently treating `None` as `0`. Compare
  against `0` explicitly: `if i % 10 == 0`, not `if not i % 10`.
- `"0"` is true. NumPy arrays raise in boolean context — test `if not users.size`.

And `x = x or []` is called out specifically as the wrong way to default an
argument.

**5.8 — Format strings; never build them with `+` in a loop.**

**Prefer f-strings.** Google leaves the choice between f-string, `%` and
`.format()` to judgment; we don't — f-strings put the value next to its
placeholder, which is the whole readability argument. Reach for `%` or `.format()`
only when the template genuinely has to exist separately from its arguments
(a translated string, a user-supplied template).

A single `a + b` join is fine; formatting with `+` is not. Accumulating with `+=`
in a loop can be quadratic, and CPython's optimization for it is an
implementation detail that may not apply. Append to a list and `"".join()`, or
write to an `io.StringIO`.

`"""` for multi-line strings; `textwrap.dedent()` when the indentation would leak
into the value. Quote characters are `ruff format`'s job, not a per-file choice.

**5.9 — f-strings in logging too.**

```python
logging.error(f"Cannot write to home directory, $HOME={homedir!r}")
```

This is a deliberate departure from Google, who require the `%s`-pattern form:

```python
logging.info("Current $PAGER is: %s", os.getenv("PAGER", default=""))
```

Their two reasons are real — some backends store the unexpanded pattern as a
queryable field, and interpolation is skipped when no logger will emit the
record. Neither pays off unless you actually query on the pattern, and the
readability cost is paid at every call site. Use the pattern form where a
specific sink depends on it (and where structured logging is the point, pass
fields as `extra=` rather than interpolating at all).

Ruff's `G004` rule flags f-strings in logging — leave it off.

**5.10 — Close what you open, with `with`.**

Files, sockets, database connections, `mmap` mappings, h5py files, matplotlib
figures. Open handles exhaust file descriptors, block moves/deletes/unmounts, and
can be read from after they're logically dead. Don't lean on `__del__`: there's
no guarantee when it runs, and a stray reference in a global or a traceback keeps
the object alive indefinitely. For objects that don't support `with`, use
`contextlib.closing()`. If context management is genuinely infeasible, the
docs must explain how the resource's lifetime is managed.

## 6. Documentation

**6.1 — A docstring is mandatory when any of three things is true:** the function
is part of the public API, it is nontrivial in size, or its logic is non-obvious.

**6.2 — The bar: enough to write a call without reading the body.**

Describe calling syntax and semantics, not implementation — *unless* the
implementation detail affects use. A function that mutates an argument as a side
effect must say so. Details irrelevant to the caller belong in comments beside
the code.

Descriptive (`"""Fetches rows from a Bigtable."""`) or imperative
(`"""Fetch rows..."""`) — consistent within a file. A `@property` docstring reads
like an attribute: `"""The Bigtable path."""`, not `"""Returns the path."""`.

**6.3 — `Args:` / `Returns:` / `Raises:`.**

```python
def fetch_smalltable_rows(
    table_handle: smalltable.Table,
    keys: Sequence[bytes | str],
    require_all_keys: bool = False,
) -> Mapping[bytes, tuple[str, ...]]:
    """Fetches rows from a Smalltable.

    Retrieves rows pertaining to the given keys from the Table instance
    represented by table_handle.  String keys will be UTF-8 encoded.

    Args:
        table_handle: An open smalltable.Table instance.
        keys: A sequence of strings representing the key of each table
          row to fetch.  String keys will be UTF-8 encoded.
        require_all_keys: If True only rows with values set for all keys will
          be returned.

    Returns:
        A dict mapping keys to the corresponding table row data fetched.
        Each row is represented as a tuple of strings. Returned keys are
        always bytes.

    Raises:
        IOError: An error occurred accessing the smalltable.
    """
```

Three details that are easy to get wrong:

- `Returns:` may be omitted if the function returns only `None`, or if the
  summary line already starts with "Returns"/"Yields" and says enough.
- A tuple return is *one* value: *"Returns: A tuple (mat_a, mat_b), where mat_a
  is ..."* — not old NumPy style, which listed the elements as if they were
  several returns and never mentioned the tuple.
- `Raises:` lists exceptions **relevant to the interface**. Do not document what
  gets raised when the caller violates the documented API — that would make
  misuse behavior part of the API.

Include types in prose only where an annotation doesn't already carry them.

**6.4 — Classes document their public attributes.**

An `Attributes:` section, formatted like `Args:`, excluding properties. The
summary says what an instance *represents*:

```python
# Yes
class CheeseShopAddress:
    """The address of a cheese shop."""

class OutOfCheeseError(Exception):
    """No more cheese is available."""
```

```python
# No
class CheeseShopAddress:
    """Class that describes the address of a cheese shop."""

class OutOfCheeseError(Exception):
    """Raised when no more cheese is available."""
```

Exception classes describe the condition, not the circumstance of raising.

**6.5 — `@override` replaces a "see base class" docstring.**

A method decorated with `@override` (from `typing` / `typing_extensions`) needs
no docstring — unless it materially refines the base contract or adds side
effects, in which case document at least the differences. Without the decorator,
a docstring is required.

**6.6 — Comment the tricky parts; never narrate the code.**

*"If you're going to have to explain it at the next code review, you should
comment it now."* Inline comments start at least two spaces from the code.

```python
if i & (i-1) == 0:  # True if i is 0 or a power of 2.
```

```python
# BAD COMMENT: Now go through the b array and make sure whenever i occurs
# the next element is i+1
```

Assume the reader knows Python better than you do — but doesn't know what you're
trying to do.

Punctuation, spelling and grammar matter here; comments should read as narrative
text. Complete sentences are usually clearer than fragments.

**6.7 — TODOs point at a tracked resource, not a person.**

```python
# TODO: crbug.com/192795 - Investigate cpufreq optimizations.
```

The old `TODO(username):` form is discouraged for new code, and TODOs whose
context is an individual or team are called out as something to avoid — people
move, issues persist. "At a future date do X" needs a specific date or a specific
event ("Remove this when all clients handle XML responses").

**6.8 — Module docstrings: a summary, a blank line, then usage.**

A typical-usage example is encouraged. Test modules don't need one unless there's
something real to say — how to run it, an unusual fixture, an external dependency.
`"""Tests for foo.bar."""` adds nothing and shouldn't be written.

## 7. Type annotations

**7.1 — Annotate public APIs first; you don't have to annotate everything.**

Then, in priority order: code prone to type errors, code that's hard to
understand, and code that has stabilized. Mature modules can usually be annotated
in full without losing flexibility.

Don't annotate `self`/`cls` (use `Self` only when the type information genuinely
requires it) or `__init__`'s return.

**7.2 — Nullability is explicit.**

```python
def f(a: str | int | None, b: str | None = None) -> str:   # yes, 3.10+
def f(a: Union[str, int, None], b: Optional[str] = None):  # yes, older syntax
def f(a: str = None):                                      # no — implicit Optional
```

**7.3 — Prefer abstract containers in signatures, built-ins for concrete types.**

`collections.abc.Sequence` over `list` in a parameter; `tuple[float, float]`
over `typing.Tuple[...]` when the concrete type is what you mean.

**7.4 — Always parameterize generics.**

A bare `Sequence` means `Sequence[Any]`. If `Any` is truly right, say `Any` — but
a `TypeVar` is often what you actually wanted:

```python
_T = TypeVar("_T")
def get_names(employee_ids: Sequence[_T]) -> Mapping[_T, str]:
```

**7.5 — Type variables need descriptive names unless private *and* unconstrained.**

`_T`, `_P` are fine. A constrained or exported one is not: `AddableType =
TypeVar("AddableType", int, float, str)`, `AnyFunction = TypeVar("AnyFunction",
bound=Callable)`.

**7.6 — Aliases are CapWorded; private ones get `_`.**

```python
_LossAndGradient: TypeAlias = tuple[tf.Tensor, tf.Tensor]
ComplexTFMap: TypeAlias = Mapping[str, _LossAndGradient]
```

**7.7 — Forward references: `from __future__ import annotations`, or quote it.**

**7.8 — `str` for text, `bytes` for binary, `AnyStr` when they must match.**
Never `typing.Text` in new code.

**7.9 — Conditional imports are a last resort; circular typing imports are a smell.**

`if TYPE_CHECKING:` is explicitly discouraged — refactoring to allow a top-level
import is preferred. Where it's unavoidable: right after the normal imports, no
blank lines inside, sorted, and the types referenced as strings.

A circular dependency that exists only because of typing is a signal to
restructure. The escape hatch (`some_mod = Any` with a meaningful alias name) is
documented, not recommended.

**7.10 — Annotated assignments, not type comments.**

`a: Foo = SomeUndecoratedFunction()`. The trailing `# type: Foo` form was
necessary before 3.6 and shouldn't be added to.

## 8. Formatting and naming

Most of this is the formatter's job. It's here so you recognize correct output.

**8.1 — 4 spaces, never tabs. 80 columns.**

Explicit exceptions to 80: long imports, URLs/paths/long flags in comments, long
module-level string constants without whitespace, and `# noqa` comments.
Docstring summary lines have no exception. Where the formatter can't get a line
under the limit, exceeding it is allowed.

*(Note: much of Google's own published code and the guide's own examples use a
2-space indent, a legacy of their internal C++ style. The stated rule for Python
is 4.)*

**8.2 — No backslash continuations.**

Use implicit joining inside brackets, adding parentheses if needed. Break at the
highest syntactic level available, and if you break twice, break at the same
level both times.

```python
# Yes
bridgekeeper.answer(
    name="Arthur", quest=questlib.find(owner="Arthur", perilous=True))

# No
bridgekeeper.answer(name="Arthur", quest=questlib.find(
    owner="Arthur", perilous=True))
```

**8.3 — Trailing commas only when the closing bracket is on its own line.**

That comma is also the signal `ruff format` reads to explode a container to one
item per line — so it's a formatting instruction, not decoration.

**8.4 — No vertical alignment of `=`, `:` or `#` across lines.**

It's a maintenance burden: one long name later and every line in the block
churns.

**8.5 — Spaces around `=` only when a parameter has both an annotation and a
default.**

```python
def complex(real, imag=0.0): ...
def complex(real, imag: float = 0.0): ...
```

**8.6 — Two blank lines between top-level definitions, one between methods.**
None after a `def` line. Parentheses sparingly — not around `return` values or
`if` conditions unless they're carrying a line break or marking a tuple.

**8.7 — Naming.**

| Type | Public | Internal |
| --- | --- | --- |
| Packages, modules | `lower_with_under` | `_lower_with_under` (modules) |
| Classes, exceptions | `CapWords` | `_CapWords` |
| Functions, methods | `lower_with_under()` | `_lower_with_under()` |
| Constants | `CAPS_WITH_UNDER` | `_CAPS_WITH_UNDER` |
| Variables, parameters | `lower_with_under` | `_lower_with_under` |

Avoid: single-character names (except counters, `e` for an exception, `f` for a
file handle, private unconstrained TypeVars, and established mathematical
notation); dashes anywhere in a module or package name; `__dunder__` names, which
Python reserves; names that encode the type, like `id_to_name_dict`.

Descriptiveness should be **proportional to scope** — `i` is fine in a five-line
block and too vague three scopes deep. Don't abbreviate by deleting letters
inside a word.

Single leading underscore for internal; double underscore is discouraged — name
mangling hurts readability and testability and isn't really private anyway. Tests
may access protected constants of the module under test.

`.py` extension, always; no dashes in filenames, or the file can't be imported or
unit-tested. Related classes and top-level functions live together in one module
— there's no one-class-per-file rule.

**8.8 — Mathematical notation is an explicit exemption.**

Short names matching a reference paper are *preferred* over style-compliant ones
in math-heavy code — provided you cite the source (ideally a link) in a comment
or docstring, still use descriptive names for public APIs, and scope the
`# noqa: N806` narrowly.

**8.9 — `main()` behind `if __name__ == "__main__":`.**

Everything at module top level runs on import — including under `pydoc` and in
test collection. Keep the work in `main()` so the module stays importable.

**8.10 — Prefer short functions; 40 lines is the prompt, not the limit.**

No hard cap. Past ~40 lines, ask whether it splits without damaging the
structure. The stated reason is future-facing: the function that works perfectly
today is the one someone extends in three months.

---

## Parting words

> **BE CONSISTENT.** If you're editing code, take a few minutes to look at the
> code around you and determine its style. [...] The point of having style
> guidelines is to have a common vocabulary of coding so people can concentrate
> on what you're saying rather than on how you're saying it.

With the limit the guide places on itself immediately afterward: consistency
applies most heavily *locally*, and to choices the global style leaves open. It
is not a reason to keep writing an old style without weighing the new one, or to
fight a codebase's drift toward newer conventions.
