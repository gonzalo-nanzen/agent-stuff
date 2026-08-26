# Designing a command-line program

Guidelines for approaching a new CLI, adapted from **[Command Line Interface
Guidelines](https://clig.dev/)** by Aanand Prasad, Ben Firshman, Carl Tashian and
Eva Parish — the best single reference on the subject. Read the original; this is
a working condensation with the examples kept concrete.

**These are guidelines, not rules.** The authors say so themselves: *"if you've
thought about it and determined that a rule is wrong for your program, that's
fine."* Several of the recommendations here deliberately contradict forty years of
UNIX tradition, and a few contradict each other. The value is in knowing which
convention you're breaking and why.

Scope: ordinary command-line programs. Not full-screen terminal applications
(vim, emacs, k9s) — those are a different discipline.

The sections follow the order you actually make the decisions.

---

## 1. Decide who it's for

Everything downstream follows from this one question, and the honest answer for
most modern tools is *humans*.

**Human-first, when humans are the primary audience.** Traditional UNIX commands
were written assuming other *programs* would call them — they had more in common
with functions than with applications. Most CLIs written today are used
overwhelmingly by people, and still carry that baggage: terse by default, silent
on success, cryptic on failure. If a command is going to be used primarily by
humans, design it for humans first.

**It will be scripted anyway.** From the original:

> Whatever software you're building, you can be absolutely certain that people
> will use it in ways you didn't anticipate. Your software *will* become a part in
> a larger system — your only choice is over whether it will be a well-behaved
> part.

The two goals are less opposed than they look. Most of this guide is about
serving both: detect whether you're talking to a terminal or a pipe, and behave
accordingly.

**Treat it as a conversation, because it already is one.** Beyond trivial
commands, using a CLI means multiple invocations: type, get an error, adjust, get
a different error, succeed. Or `git add` several times then `git commit`. Or `cd`
and `ls` repeatedly to build a mental map. Once you accept that, the design moves
follow — suggest corrections on bad input, make intermediate state visible,
confirm before something scary, say what to run next.

> The user is conversing with your software, whether you intended it or not. At
> worst, it's a hostile conversation which makes them feel stupid and resentful.

**Say just enough.** A command that hangs for four minutes in silence is saying
too little; one that dumps pages of debug output is saying too much. Both leave
the user unsure whether it worked. Getting this balance right is most of the
craft.

## 2. The basics

Short list, and getting any of it wrong makes the program a bad citizen.

**Use an argument-parsing library.** Your language's built-in one or a good third
party — argparse/click/typer, Cobra, clap, picocli, clikt, oclif,
swift-argument-parser. They handle flag parsing, help text and spelling
suggestions in ways you will not reproduce by hand, and they make your program
conventional for free.

**Exit zero on success, non-zero on failure.** This is how every script, CI step
and `&&` chain decides what happened. If your program has several distinct failure
modes worth acting on, map them to distinct non-zero codes and document them.

**Send primary output to `stdout`. Send everything else to `stderr`.**

Results go to stdout — that's what gets piped. Log lines, progress, prompts and
errors go to stderr, so they reach the user without corrupting the pipeline. The
test: `mycmd | grep foo` should still show the user what's happening, and
`mycmd > out.txt` should put only real output in the file.

## 3. Help is the interface

A GUI shows you everything it can do. A CLI has to be asked. Help text is
therefore not documentation *about* the interface — for most users it *is* the
interface.

**`-h`, `--help` and bare invocation all work.** Users add `-h` to whatever they
have already typed, so it should win over other flags and arguments and never
have side effects. For a multi-command tool, support `mycmd help`,
`mycmd help sub` and `mycmd sub --help` too.

**Full help on `--help`; concise help on bare invocation.** If a command needs
arguments and gets none, don't error — show a short version: what the program is,
one or two examples, the most important flags, and how to get the rest. `jq` is
the reference implementation:

```
$ jq
jq - commandline JSON processor [version 1.6]

Usage: jq [options] <jq filter> [file...]

jq is a tool for processing JSON inputs, applying the given filter to
its JSON text inputs and producing the filter's results as JSON on
standard output.

Example:

    $ echo '{"foo": 0}' | jq .
    {
        "foo": 0
    }

For a listing of options, use jq --help.
```

**Lead with examples.** People reach for an example before they read a flag table.
Order them from simple to complex, and include output when it's short enough to
help. Push exhaustive examples to a cheat sheet or a web page rather than bloating
`--help`.

**Put common things first.** Not alphabetical — frequency order. `git` with no
arguments groups its commands by task ("start a working area", "work on the
current change", "examine the history and state") rather than listing all 150.

**Format for scanning.** Bold headings and clear sections beat a wall of text.
`heroku` is the pattern worth copying:

```
$ heroku apps --help
list your apps

USAGE
  $ heroku apps

OPTIONS
  -A, --all          include apps in all teams
  -p, --personal     list apps in personal account
  -s, --space=space  filter by space

EXAMPLES
  $ heroku apps
  === My Apps
  example
```

**Suggest, don't correct.** When you can guess what they meant, offer it — don't
silently run something else:

```
$ heroku pss
 ›   Warning: pss is not a heroku command.
Did you mean ps? [y/n]:
```

**Link out, and give a support path.** Top-level help should carry a URL for the
docs and a place to report bugs. Link specific pages or anchors, not just the
homepage.

**Don't hang waiting for stdin that isn't coming.** If the program expects piped
input and stdin is an interactive terminal, print help and exit rather than
sitting there looking broken.

## 4. Arguments and flags

**Prefer flags to positional arguments.** Flags are self-documenting at the call
site, they're order-independent, and they let you add options later without
breaking anyone. Positional arguments are worth it when the command has one
obvious subject (`cp <source> <destination>`, `rm file1 file2 file3`) — beyond
that, ambiguity sets in fast.

**Always provide the long form.** `-h` and `--help`, `-f` and `--force`. Scripts
read better with long flags, and users don't have to look them up. Keep
single-letter flags for the genuinely common options; the namespace is small and
you'll want it later.

**Use the conventional names.** These are hardwired into people's fingers:

| flag | meaning |
|---|---|
| `-a`, `--all` | all |
| `-d`, `--debug` | debugging output |
| `-f`, `--force` | force the operation |
| `-h`, `--help` | help — **and nothing else, ever** |
| `-n`, `--dry-run` | describe, don't do |
| `-o`, `--output` | output file |
| `-p`, `--port` | port |
| `-q`, `--quiet` | less output |
| `-u`, `--user` | user |
| `-v` | verbose *or* version — pick one and be unambiguous |
| `--json` | machine-readable output |
| `--no-input` | never prompt |
| `--version` | version |

**Make the defaults right for the majority.** Most users will never discover a
flag, let alone remember it. Many UNIX defaults are terse because the assumed
caller was a script — `ls` being the classic. That reasoning may not apply to you.

**Accept `-` for stdin/stdout.** It removes the need for temporary files:

```
$ curl https://example.com/something.tar.gz | tar xvf -
```

**Be order-independent where you can.** People edit the previous line and append
a flag. Requiring flags before subcommands, or in a fixed order, breaks that
habit constantly.

**Never accept secrets in flags.** Flag values land in `ps` output, in shell
history, and in CI logs. Take a `--password-file`, or read from stdin. (See §8 —
environment variables are barely better.)

## 5. Output

**Human-readable first, machine-readable on request.** Detect whether stdout is a
TTY: format for a person when it is, keep it plain and line-based when it isn't.

**Offer `--json` when the data has structure**, and `--plain` when your
human-friendly formatting (columns, wrapping, truncation) would break line-based
tools. Once you publish either, treat it as a contract — see §10.

**Say something on success, briefly.** Silence-on-success made sense when the
caller was a script. For a human it's indistinguishable from a no-op. Provide
`-q` for the script case.

**Explain state changes.** If the program changed something — files, a remote, a
database — say what changed. `git push` narrates the whole thing and ends with
the ref update. Anything that crosses a boundary the user might not expect —
reading a file elsewhere on disk, contacting a remote server — should be
announced.

**Make current state easy to inspect.** `git status` exists because the index is
invisible otherwise. If your tool has internal state, give it a command that
prints that state in full.

**Suggest the next command.** This is the cheapest discoverability win available.
`git status` does it inline:

```
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   cli/pkg/cli/rm.go
```

**Use color intentionally, and let it be turned off.** Red for errors, color for
the thing that matters — not everywhere, or it stops meaning anything. Disable it
when: stdout/stderr is not a TTY, `NO_COLOR` is set and non-empty, `TERM=dumb`,
or `--no-color` is passed.

**No animations when nobody's watching.** Progress bars and spinners in a
non-TTY context produce thousands of lines of garbage in CI logs. Check first.

**Don't treat stderr as a log file.** `[2026-08-26T14:22:01Z] WARN` prefixes are
for your debugging, not the user's. Keep them behind `--verbose`, and keep
tracebacks behind it too — until something genuinely unexpected happens, at which
point show the traceback *and* tell them how to report it.

**Page long output.** If you're about to emit more than a screenful, send it
through `less` (`-FIRX` is the usual set — `git` uses `FRX`), and only when stdout
is a TTY. Respect `$PAGER`.

## 6. Errors

Error messages are the part of your program people read most carefully, usually
while frustrated.

**Write them for humans.** Catch the errors you can anticipate and translate
them. Not `EACCES: permission denied, open 'file.txt'` but *"Can't write to
file.txt. You might need `chmod +w file.txt`."* The pattern is: what failed, why,
and the most likely fix.

**Protect the signal-to-noise ratio.** If the same problem occurs 400 times, print
a header and a count, not 400 identical lines. Cut anything that doesn't help.

**Put the important part last.** It's what's on screen when the command stops
scrolling. Use red sparingly enough that it still draws the eye.

**For genuinely unexpected errors, make reporting effortless.** Include the
traceback, or write it to a file and say where. Give a URL — pre-populated with
the version, the platform and the error, if you can manage it.

## 7. Interactivity and dangerous operations

**Only prompt when stdin is a TTY.** That's the reliable test for "a human is
here." In a script or CI, don't prompt — fail with a message naming the flag they
should have passed.

**Honour `--no-input`.** Same behaviour, requested explicitly.

**Never *require* a prompt.** Everything promptable must also be settable by flag
or argument, or your tool cannot be automated.

**Turn off echo for passwords.** Table stakes.

**Make it obvious how to get out.** Ctrl-C should work, including during network
I/O. If you wrap a session (SSH-style), document the escape sequence.

**Scale confirmation to the blast radius.** Roughly:

- *Mild* — deleting a file the user named. Confirmation optional; `-f` to skip.
- *Moderate* — deleting a directory, changing something remote, bulk edits.
  Prompt by default, and offer `--dry-run` so they can look first.
- *Severe* — destroying a remote resource that can't be recreated. Make it
  deliberately hard: require typing the resource's name. Provide
  `--confirm="name"` so it stays scriptable.

The `--dry-run` flag deserves special mention: for anything destructive or
sweeping, it's the difference between a tool people trust and one they route
around.

## 8. Configuration

Three kinds of configuration, and the kind determines the mechanism:

| what it is | example | where it goes |
|---|---|---|
| Varies per invocation | `--dry-run`, `--verbose` | **flags** |
| Stable per user or machine | proxy, color, non-default paths | **flags + environment variables** |
| Stable per project, belongs in git | `docker-compose.yml`, `package.json` | **a config file in the repo** |

**Precedence, highest to lowest:** flags → environment variables → project config
→ user config → system config. This is what people expect; deviating from it
produces bugs nobody can reproduce.

**Follow the XDG Base Directory Specification.** User config belongs in
`$XDG_CONFIG_HOME` (`~/.config/<yourtool>/`), not another dotfile in `$HOME`.
Widely adopted — fish, yarn, neovim, tmux, wireshark, and git all support it.

**Read the general-purpose environment variables** rather than inventing your own:
`NO_COLOR`, `DEBUG`, `EDITOR`, `PAGER`, `TMPDIR`, `HOME`, `SHELL`, `TERM`,
`HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY`, `LINES` / `COLUMNS`. For your own,
use `UPPERCASE_WITH_UNDERSCORES`, prefix them with your tool's name, keep values
single-line, and don't collide with anything in POSIX.

**Ask before editing files you don't own.** Prefer writing a new config file to
modifying an existing one. If you must touch a shared file, delimit your addition
with dated comments so a human can find and remove it.

**Never read secrets from environment variables.** They're exported to every child
process, visible in `docker inspect` and `systemctl show`, and trivially leaked by
a shell substitution. Use a credentials file with tight permissions, a pipe, a
UNIX socket, or a real secret service. `.env` files are fine for development
configuration and a bad place for production secrets — they're rarely
version-controlled, have no history, and hold everything as strings.

## 9. Robustness

Robust means both *being* robust and *feeling* robust — they need different work.

**Validate input early**, before you've half-done something.

**Respond within about 100ms**, even if the work takes minutes. Print what you're
about to do before you do it — especially before a network request, which is
where a program most often looks hung.

**Show progress for anything slow**, with enough motion to prove it's alive. If
you run work in parallel, use a library that handles multiple progress bars
properly rather than interleaving writes yourself. When a step fails, print the
logs the progress display was hiding.

**Set timeouts on network calls**, with sensible defaults and a way to change
them.

**Make interrupted runs resumable.** Re-invoking after a Ctrl-C should pick up
rather than start over or refuse to proceed.

**Exit fast on Ctrl-C.** Print something so it's clear you heard it, put a timeout
on cleanup, and let a second Ctrl-C skip cleanup entirely — `docker compose up`
is the model:

```
^CGracefully stopping... (press Ctrl+C again to force)
```

The corollary is that cleanup will sometimes not run, so **design for messy
state at startup**. Crash-only is easier to get right than orderly shutdown:
if the next invocation can repair the state, it doesn't need to be repaired now.

## 10. Naming, subcommands, and not painting yourself in

**Naming:** short, lowercase, letters and dashes only, memorable, easy to type,
and not already taken by something common. (ImageMagick and Windows both shipped a
`convert`; it went badly for years.) Very short names are reserved for utilities
of near-universal use.

**Subcommands:** be consistent. Same flag names across subcommands, same output
shape, same verbs applied to different nouns. For multi-level commands prefer
`noun verb` (`docker container create`). Avoid pairs nobody can distinguish —
`update` and `upgrade` in the same tool is a support burden forever.

**Keep changes additive.** Adding a flag is safe. Changing what an existing flag
does is not. When you must, warn first, say what to use instead, and stop warning
once the user has switched.

**Changing human-readable output is usually fine** — that's the price of improving
it. This only works if there's a stable alternative for scripts, which is what
`--json` and `--plain` are for. Say clearly which one you consider a contract.

**Two traps that close doors permanently:**

- *Catch-all subcommands.* If bare `mycmd foo.txt` means `mycmd run foo.txt`, you
  can never add a subcommand named `foo.txt`, and any new subcommand risks
  colliding with a filename.
- *Automatic abbreviation.* If `mycmd i` resolves to `install` by prefix match,
  you can never add another command starting with `i` without breaking scripts.
  Explicit, declared aliases are stable; inferred ones are a liability.

**Avoid time bombs.** Assume the binary someone installs today gets run in twenty
years. Anything that depends on a server you maintain, a certificate that expires,
or an API that will be retired is a scheduled failure.

**Don't collect telemetry without consent.** People find out, and they take it
personally. If you must, prefer opt-in; if you go opt-out, say so prominently and
make disabling it one obvious step. Downloads, docs analytics and simply asking
users will answer most of the questions telemetry would have.

---

## Worked example — `git`, end to end

`git` is worth walking because it's on every machine you own and it demonstrates
most of this guide — including, honestly, a couple of the mistakes.

**Help.** Bare `git` prints concise help with commands grouped by task, not
alphabetically (§3). `git help`, `git help commit` and `git commit --help` all
work, the latter two opening the man page. `git stauts` produces *"git: 'stauts'
is not a git command… The most similar command is status"* — suggest, don't
correct (§3), and `help.autocorrect` lets you opt into having it run automatically.

**Flags.** Long forms everywhere, short forms only for the common ones. `-n` is
`--dry-run` on `git add`, matching convention (§4). `git apply -` reads a patch
from stdin. Aliases exist but you have to declare them — git will not guess that
`git comm` means `commit` (§10).

**Output.** `git status` prints state that would otherwise be invisible, with the
next command inline in parentheses (§5). `git push` narrates progress on stderr
and ends with the ref update, so `git push 2>/dev/null` still gives you the
summary. Colour is on for a terminal and off for a pipe, controlled by
`color.ui`. `git diff` pages automatically through `less` — respecting
`$GIT_PAGER`, then `core.pager`, then `$PAGER` — and only when stdout is a TTY
(§5).

**Exit codes.** `git diff --quiet` exits 1 when there are differences and 0 when
there aren't, which makes it directly usable in a shell conditional (§2). This is
the "map failure modes to codes" advice taken seriously.

**Interactivity.** `git commit` with no `-m` opens `$EDITOR` (§8), and aborts if
you leave the message empty. Credential prompts appear only on a TTY; in CI it
fails with a message instead of hanging (§7).

**Configuration.** Precedence runs system → global → local → `-c` on the command
line, exactly the order in §8. Global config lives at `~/.gitconfig` *or*
`$XDG_CONFIG_HOME/git/config` — XDG support added without breaking the old path,
which is §10's additive-change rule applied to a file location.

**Where it departs from the guide — instructively.** `git checkout` did two
unrelated jobs (switch branches, restore files) for fifteen years, which is
exactly the "ambiguous command" problem in §10. The fix in 2.23 is the right
pattern: they did not change `checkout`, they *added* `git switch` and `git
restore` and left the old behaviour alone. Additive, no broken scripts, no
deprecation deadline.

And git's error messages remain the standing counter-example to §6 — *"fatal:
refusing to merge unrelated histories"* tells you the rule that fired, not what
to do about it. Even careful, long-lived tools leave this one on the table, which
is a decent argument for writing your errors early rather than last.
