# Claktor CLI

Claktor finds repeated logic, even when it is written differently, and
measures how much smaller each change makes the program. Use it with a coding
agent to shorten a codebase while keeping its behavior.

This repository holds the signed releases of the `claktor` binary and takes
issues. The binary is licensed under the [PolyForm Shield License 1.0.0](LICENSE).

## Install

Tell your coding agent:

> Install Claktor with `curl -fsSL https://claktor.com/install | sh`, then run
> `~/.claktor/bin/claktor guide` and follow it on this repository.

Or install it yourself, on Linux or macOS, x86_64 or arm64:

```sh
curl -fsSL https://claktor.com/install | sh
```

The script downloads the binary for your platform, checks its SHA-256 and
prints `installed: <path>` when it is done. The same steps by hand:

```sh
os=$(uname -s | sed 's/Linux/linux/; s/Darwin/macos/')
cpu=$(uname -m | sed 's/aarch64/arm64/; s/amd64/x86_64/')
base=https://claktor.com/dl
mkdir -p ~/.claktor/bin && cd ~/.claktor/bin
curl -fsSL "$base/claktor-$os-$cpu" -o claktor.new
curl -fsSL "$base/claktor-$os-$cpu.sha256" -o claktor.sha256
got=$( (sha256sum claktor.new 2>/dev/null || shasum -a 256 claktor.new) | cut -d' ' -f1)
[ "$got" = "$(cut -d' ' -f1 claktor.sha256)" ] && chmod 755 claktor.new && mv claktor.new claktor
rm -f claktor.new claktor.sha256
~/.claktor/bin/claktor version
```

The binary lives at `~/.claktor/bin/claktor`. Run it by that path, or link it
into a directory on your `PATH`. `https://claktor.com/dl/<file>` redirects to
this repository's latest release and counts the download.

## Guide

`claktor guide` prints what follows. It is written for a coding agent.

Claktor finds repeated logic, even when it is written differently, and
measures how much smaller a change makes the program. Use it to shorten a
codebase while keeping its behavior. Every command takes `--json`.

Make the largest justified behavior-preserving refactor you can within the
available budget. Reduce the complete program's normalized token count. Choose
a coherent change suitable for one PR; you may explore alternatives and repair
your work.

### In short

1. `claktor map`, complete `.claktor/decisions.json`, then `claktor map compile`
   until `error_count` is 0. The map keeps tests and generated files out of
   the program and its rankings. A repository that already has `.claktor/`
   is mapped: `claktor map compile` prints its targets and their checks.
2. `claktor map target` names the largest production target, its check and
   where to search; then `claktor similar DIRECTORY --target TARGET --limit 20 --json`.
3. Run the target's check, make one coherent change, run the check again.
4. `claktor length diff --compare <start commit>`: keep the change at `delta`
   -50 or lower.

The short list gives the order. The sections below define each step; follow them.

### Map production source

Map the repository before choosing where to search. Reuse its saved decisions
on later runs; refresh the inventory and compile after the source layout changes.

1. `claktor map` writes the tracked files, parsed manifests and generated-file
   headers to `.claktor/` at the repository root, which git ignores, and seeds
   `.claktor/decisions.json`.
2. Complete `decisions.json`. Read test configuration, manifests and
   generation scripts, and search `inventory.json` for generated, vendored and
   static files (`generated/`, `vendor/`, `public/`, minified bundles); confirm
   what writes or serves each one.
   - `tests`: test files and test helpers, as directories ending `/`
     (`tests/`) or globs (`**/*.test.ts`).
   - `exclude`: exact tracked paths, or directories ending `/`, of generated,
     vendored or fixture source, each with the concrete evidence as `reason`.
     Keep the seeded entries; drop one when the source contradicts it.
   - `projects`: each manifest listed under `unresolved` in `facts.json`,
     mapped to its project root (`.` for the repository root).
   - `exceptions`: a path the compiler flags, mapped to the source-backed
     explanation of why its evidence misleads or why its directory is excluded.
   Everything else stays production, including scripts and examples.
   `claktor map --help` shows a complete `decisions.json`. `exclude` entries are
   objects: `{"path": "vendor/", "reason": "copied by scripts/vendor.sh"}`.
3. `claktor map compile` validates the decisions and prints the targets. Fix
   what it reports and rerun until `error_count` is 0.
   `.claktor/memberships.json` lists the targets of every file.
4. Record the repository's existing check for each production target (the
   command its CI or contributor documentation runs for that code) in
   `decisions.json` under `checks`, keyed by the printed target name, as
   `{"argv": ["pnpm", "typecheck"], "generated": []}`. Compile again until
   `production_targets_without_a_check` is empty, or at least omits the target
   you will change: a later session learns the check from `map compile`.
   Subsequent commands use the saved map; `.claktor/exclusions.md` records its
   evidence.

### Select the largest production target

1. From the repository root, run `claktor map target`. It measures every
   production target, selects the one with the most normalized tokens (ties
   go to the first name) and prints its name, its recorded `check` and its
   retrieval `directories`: `packages/NAME` for files under `packages/NAME/`,
   the first directory for other nested files and `.` for root files, ordered
   by the source bytes each holds. `partial: true` means some files were
   skipped, so the totals are lower bounds.
2. The selection is saved in `.claktor/search.json` with its commit, so a
   later session resumes from the same scope. Start retrieval in the first
   directory. The complete production target remains available for the change.
3. Run `claktor similar DIRECTORY --target TARGET --limit 20 --peers 3 --json`.
   Follow `nextCommand` until you have twenty ranked leads or the listing ends.
   When the leads fall short of a worthwhile reduction, inspect the target
   directly or continue through the ordered directories. Use directory-sized queries.
   The first query can take minutes; later pages and `--family` reuse it.
   Exit code 2 means some oversized code was skipped; the listing is still usable.

### The loop

1. Ranked leads estimate net savings after extraction costs. They are starting
   hypotheses: `length diff` measures the saving and the check tests the
   behavior. A lead's families are disjoint, so one change may cover them all.
   Separate leads may overlap, so treat them as alternatives. `--family ID`
   pages every occurrence; follow `nextCommand`.
2. Read repository guidance and run the selected target's recorded check before
   changing anything. Use the repository's pinned runtime and dependencies.
3. Read every cited site and its callers. Work out what must stay the same:
   outputs, errors, side effects, evaluation order, public interfaces, types.
4. Create a new branch and pursue the largest justified reduction in one
   coherent change, within twenty production files in the selected target.
   Keep tests, their expectations, manifests and check configuration unchanged.
   Keep helpers and the rest of the implementation in eligible production
   source inside the measured scope, on the existing dependencies.
5. Inspect the complete diff, run the recorded check and repair any issues your
   changes cause. Refresh similarity after edits when exploring further savings.
6. Run `claktor length diff --compare <start commit> --summary`. It measures
   the complete change, including new helpers and callers; renaming,
   formatting and erasing types leave the count as it is. Its `changed` row
   sums the changed files alone. Remeasure after repairs and review every
   outside-scope or unmeasured change.
7. Aim for the largest justified reduction within the available resources.
   Keep the final change when the check passes and `delta` is -50 or lower. If
   it falls short, keep seeking a worthwhile reduction; once the search is
   exhausted, discard your candidate and say so.
8. Write the pull request title and body: what repetition the change removes,
   what behavior it keeps, the `delta`, and the checks you ran.

Other commands: `claktor length` totals a program, `claktor version` prints
the build, and `claktor update` installs the newest signed release.

### Paid version

The paid version adds a sub-agent that you, the coding agent, run as a
process. It comes with more features and workflows built for refactoring: a
model, prompts and leads tuned on our refactoring evaluations, a dead-code
finder, and a memory of what earlier runs tried. Together they refactor code
for less than a general-purpose agent costs.

Refactored code is cheaper to work on, because there is less of it to
maintain, test and, above all, read. Every line that stays in the repository
is one the next engineer and the next agent read before they can act. A
smaller program means less reading for the agents that review pull requests,
the agents that develop and the agents that verify.

It is in development. Run `claktor interest` to register interest; add
`--email ADDRESS` if the person wants to hear when it is ready.

### Usage data

An installed copy records one anonymous event per command: command name, exit
code, duration, version, platform, driver, the languages of the mapped
repository and a random install id. It sends them together, once twenty are
waiting or a minute after the first. Arguments, paths, repository names and
code stay on the machine.

## Updates

After you run a command, an installed copy checks for updates at most once an
hour. It installs a newer release when its `manifest.json` carries a valid
minisign signature from the key below. `claktor update` checks now.

To verify a release by hand:

```sh
minisign -Vm manifest.json -P RWTMeGrmjBuO+Zqs2P6TD4tncfAX5P81i3/1mfvRyf1tr3p8CCEu0I+q
```

Public key: `RWTMeGrmjBuO+Zqs2P6TD4tncfAX5P81i3/1mfvRyf1tr3p8CCEu0I+q`

## Usage data settings

`claktor telemetry off` stops the usage events and deletes the ones still
queued. `claktor telemetry on` resumes them and `claktor telemetry status`
shows the setting. `DO_NOT_TRACK=1` in the environment does the same for the
commands that run with it.
