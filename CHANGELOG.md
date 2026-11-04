# Changelog

## v0.1.0 — 2026-11-04

First release.

- **Scan a repo before you — or your agent — opens it.**
  Born from the August 2026 worm campaign that planted
  autostart hooks in `.claude/settings.json` and
  `folderOpen` tasks in `.vscode/tasks.json`: files that
  execute when a developer or an AI coding agent opens a
  cloned repository, with no install required.
  clone-guard reports every such surface with its command
  printed verbatim, grouped by when it runs.
- **Nine checked locations, ten rules** — Claude Code
  hooks (any `command` string under `hooks`, per event)
  and `apiKeyHelper` in `.claude/settings.json` /
  `settings.local.json`; VS Code `folderOpen` tasks and
  `task.allowAutomaticTasks: "on"`; devcontainer
  `initializeCommand` / `postCreateCommand` /
  `postStartCommand` / `postAttachCommand` (string, argv
  and object forms); npm `preinstall` / `install` /
  `postinstall` scripts; `.envrc`; non-sample files in
  `.git/hooks/`; hook files in `.husky/` (internals
  skipped). Verdicts: AUTO-RUN, INSTALL-TIME, PRESENT,
  UNPARSEABLE — when it runs, never whether it is evil.
- **False-positive discipline** — ordinary project files
  (`Makefile`, `setup.py`, `*.sh`, `flake.nix`, source,
  docs) are never flagged, no matter what they contain;
  VS Code tasks without `folderOpen` and npm scripts
  other than the three install hooks are never flagged;
  sample git hooks are ignored. The flagged set is config
  that other tools execute automatically.
- **Provably read-only** — the scanner opens the checked
  files and reads them; it never executes, installs,
  sources or writes anything. The test suite hashes the
  entire scanned tree (every path + content, `.git`
  included) before and after scans in text and `--json`
  modes and requires exact equality.
- **JSONC-tolerant parsing** — strict JSON first, then a
  string-aware pass that strips `//` / `/* */` comments
  and trailing commas (a `//` inside a URL string is data
  and survives). A checked file that still cannot be
  parsed — or is not a JSON object — becomes an
  UNPARSEABLE finding ("could not be cleared"), never a
  crash and never a silent skip.
- **`--json` on the scan** — one JSON object with the
  same facts and exit codes as the text view (findings
  with file / rule / verdict / trigger / verbatim command
  / detail, a per-verdict summary, `clean`); errors are
  `{"error": ...}` objects, argparse usage errors
  included. Exit codes: 0 CLEAN · 1 FINDINGS · 2 ERROR.
- **`SKILL.md`** at the repo root for AI coding agents:
  the rule set at its exact edges, the JSON schema, exit
  codes, and the honesty rules (CLEAN means "none of the
  checked surfaces were found", never "this repo is
  safe"; scan before opening, never after; quote commands
  verbatim).
- Bundled sample generator (`examples/make_sample.py`): a
  planted-bad repo hitting every rule family (8 findings:
  5 AUTO-RUN · 1 INSTALL-TIME · 2 PRESENT, worked out by
  hand in the test-suite header) next to an ordinary clean
  repo whose hook-free settings and normal npm scripts
  must come back CLEAN. All "hostile" commands point at
  the reserved `example.invalid` domain — inert text
  fixtures, read and reported, never run.
- 56 stdlib-unittest tests, temp dirs only: every rule in
  both directions, the parser at its edges (comments in
  strings, escaped quotes, trailing commas, BOM), the
  exact bundled-sample counts, malformed configs as
  findings, normal files never flagged, and the
  hash-proven read-only guarantee.
- Pure Python standard library; local only; no network
  code; MIT licence.
