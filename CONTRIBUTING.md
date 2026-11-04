# Contributing

Contributions are welcome — bug reports, fixes and small focused features.

## Ground rules

- **Standard library only.** clone-guard's promise is
  "clone and run". Pull requests that add a third-party
  dependency will not be merged.
- **Local only.** No network calls, no telemetry, no uploads
  of user files. The tool reads a fixed list of config
  files in one folder and sends nothing anywhere.
- **Read-only is the product — never weaken it.** The
  scanner must never execute, install, source, or write
  to anything in the scanned folder (or anywhere else).
  A PR that "verifies" a finding by running the command,
  or that writes a report file into the scanned tree,
  will not be merged. The suite hashes the scanned tree
  around scans — keep those tests green.
- **False-positive discipline is the other product.** The
  flagged set is config that other tools execute
  *automatically*. Ordinary files (`Makefile`, `setup.py`,
  shell scripts, source, docs) are never flagged, however
  scary their contents look — a human chooses to run
  those; nobody chooses a `SessionStart` hook. A PR that
  flags a new surface needs to name the tool that executes
  it automatically and the moment it fires.
- **Verdicts say when, never whether.** New rules get a
  verdict from the existing four (AUTO-RUN, INSTALL-TIME,
  PRESENT, UNPARSEABLE) and a trigger that states the
  firing moment. Verdicts like "malicious" or "safe" will
  not be merged — the tool reports surfaces and verbatim
  commands; people judge.
- **Unparseable is a finding, never a crash and never a
  skip.** Any new parser or checked file must route its
  failures to the UNPARSEABLE finding. A traceback in a
  scan is a release-blocking bug.
- **Both directions tested.** A behaviour change needs a
  test where the surface fires and one where it does not
  (a normal file, a hook-free config, a task without
  `folderOpen`) — not just one side. Expected findings
  need hand-computed values, not output copied from the
  tool itself.

## Getting started

```bash
git clone https://github.com/jasveerai88/clone-guard
cd clone-guard
python3 -m unittest discover -s tests -v
python3 examples/make_sample.py /tmp/clone-guard-demo
python3 src/clone_guard.py /tmp/clone-guard-demo/suspicious-repo
```

## Pull requests

1. Fork, branch, and keep the change to one topic.
2. Add or update tests under `tests/`; the full suite must pass.
3. Note the change in `CHANGELOG.md` under an "Unreleased" heading.
4. Describe the behaviour change with a real before/after run if
   the CLI output changes.
