---
name: clone-guard
description: Scan a freshly cloned repo BEFORE opening it, and report the surfaces that execute code on open, on install, or on git events. `clone-guard DIR` checks exactly: `.claude/settings.json` + `settings.local.json` hooks and `apiKeyHelper` (AUTO-RUN), `.vscode/tasks.json` `folderOpen` tasks (AUTO-RUN), `.vscode/settings.json` `task.allowAutomaticTasks: "on"` (PRESENT), `.devcontainer/devcontainer.json` initialize/postCreate/postStart/postAttach commands (AUTO-RUN), `package.json` preinstall/install/postinstall scripts (INSTALL-TIME), `.envrc` (AUTO-RUN), `.git/hooks/` non-sample files (PRESENT), `.husky/` hook files (PRESENT). Every finding shows the exact command verbatim. Config parsing is JSONC-lenient (comments + trailing commas); an unparseable checked file is an UNPARSEABLE finding, never a crash. Ordinary files (Makefile, setup.py, *.sh, flake.nix) are never flagged. Read-only — the scanner never executes, installs or modifies anything. Exit 0 = CLEAN (no findings), 1 = FINDINGS, 2 = error. Use on every repo before an editor or agent opens it.
---

# clone-guard — agent skill

## Purpose

The dangerous moment in a repo's life is the moment it is
opened: editors and AI coding agents auto-read config
files, and some of those files carry commands that run on
open — Claude Code hooks, VS Code `folderOpen` tasks,
devcontainer lifecycle commands — with no install step.
In August 2026 a worm campaign planted exactly those
files in public repos. clone-guard is the deterministic
pre-open check: one command, run on a clone nothing has
opened yet, that lists every open-time / install-time /
git-event execution surface with its command verbatim.

**The safety core of this tool:** clone-guard only reads.
It never executes, installs, sources or modifies anything
in the scanned folder, and its own test suite proves that
by hashing the scanned tree before and after. A scan is
always safe to run — including on a repo you already
suspect.

## When to use / not use

Use when:
- You (or an agent you direct) are about to open, explore
  or work in a repo you did not write: clone it to a
  scratch folder, run clone-guard, read the commands in
  the report, and only then open it.
- A pipeline adopts third-party tools: the scan report is
  the open-time half of the adoption safety re-check.
- You need a yes/no gate: exit 0 means none of the checked
  surfaces exist; exit 1 hands you the exact list.

Do NOT use when:
- You want a malware verdict. clone-guard reports
  *surfaces*, never guilt: legitimate projects use hooks
  and install scripts constantly. The command text is the
  evidence; a human (or a careful agent) judges it.
- The code itself needs review. clone-guard does not read
  source files, look for signatures, or check other
  agents' config formats (`.cursor/`, `.gemini/`, MCP
  server configs). CLEAN + unreviewed code is still
  unreviewed code.
- The project root is nested. Only the folder named on the
  command line is scanned; scan each subproject root
  separately.

## Commands

```bash
# Scan a clone before opening it (text report)
python3 src/clone_guard.py DIR

# Same facts as one JSON object — for scripts and agents
python3 src/clone_guard.py DIR --json
```

## Rules (exact)

Checked — and only these:

- **`.claude/settings.json`, `.claude/settings.local.json`**
  — every string `command` value anywhere under the
  `hooks` object, reported per event (`SessionStart`,
  `PreToolUse`, ...) as `claude-hook` (AUTO-RUN); plus a
  string `apiKeyHelper` as `claude-api-key-helper`
  (AUTO-RUN).
- **`.vscode/tasks.json`** — tasks whose
  `runOptions.runOn` is `folderOpen`, as
  `vscode-folderopen-task` (AUTO-RUN). The reported
  command is the task's `command` plus its string `args`
  joined with spaces; a task with no command of its own
  is still reported, with a `dependsOn` note in `detail`.
  Tasks without `folderOpen` are never flagged.
- **`.vscode/settings.json`** —
  `task.allowAutomaticTasks` equal to `"on"`, as
  `vscode-automatic-tasks` (PRESENT).
- **`.devcontainer/devcontainer.json`** —
  `initializeCommand`, `postCreateCommand`,
  `postStartCommand`, `postAttachCommand`, as
  `devcontainer-command` (AUTO-RUN). String, argv-list
  (joined with spaces) and object (one finding per value)
  forms are all flattened to command strings.
- **`package.json`** — `scripts` entries `preinstall`,
  `install`, `postinstall`, as `npm-install-script`
  (INSTALL-TIME). All other scripts are never flagged.
- **`.envrc`** — presence, as `direnv-envrc` (AUTO-RUN),
  with the file's line count in `detail`.
- **`.git/hooks/`** — files not ending in `.sample`, as
  `git-hook` (PRESENT), with line count and executability
  in `detail`. Subdirectories are skipped.
- **`.husky/`** — files not starting with `.` or `_`
  (the `_/` internals and dotfiles are skipped), as
  `husky-hook` (PRESENT), with line count in `detail`.

Never flagged: `Makefile`, `setup.py`, `*.sh`, `flake.nix`,
`shell.nix`, README files, source code — no matter what
they contain. The flagged set is config that *other tools
execute automatically*, not files a human might choose to
run.

Parsing: strict JSON first, then a string-aware JSONC pass
(`//` and `/* */` comments and trailing commas stripped;
a `//` inside a string — e.g. a URL — is preserved). A
checked file that still fails to parse, is not UTF-8, or
whose top level is not a JSON object yields exactly one
`unparseable` finding (UNPARSEABLE) naming the file —
never a traceback, never a silent skip.

Findings are sorted: by verdict severity (AUTO-RUN,
INSTALL-TIME, PRESENT, UNPARSEABLE), then file, then
trigger. Duplicate (event, command) pairs within one
Claude settings file are reported once.

## Output JSON schema (`--json`)

Common fields: `tool` (`"clone-guard"`), `version`,
`command` (`"scan"`), `dir`, `surfaces_probed` (the count
of known surface locations checked — 9), `findings`,
`summary`, `clean`.

Each finding: `file` (path relative to DIR), `rule`
(`claude-hook` | `claude-api-key-helper` |
`vscode-folderopen-task` | `vscode-automatic-tasks` |
`devcontainer-command` | `npm-install-script` |
`direnv-envrc` | `git-hook` | `husky-hook` |
`unparseable`), `verdict` (`AUTO-RUN` | `INSTALL-TIME` |
`PRESENT` | `UNPARSEABLE`), `trigger` (when it runs),
`command` (the exact command string, or `null` when the
surface carries no single command), `detail` (extra facts
— line counts, executability, parse errors — or `null`).

`summary`: `{total, auto_run, install_time, present,
unparseable}`. `clean` is true exactly when `findings` is
empty.

On error, stdout is a single object: `{"error": "<message>"}`
with exit code 2 — including argparse usage errors in
`--json` mode. Never plain text, never a traceback.

## Exit codes

- `0` — CLEAN: no findings on any checked surface.
- `1` — FINDINGS: at least one finding of any verdict,
  including UNPARSEABLE. The scanned folder is unchanged
  (it is always unchanged — the tool only reads).
- `2` — ERROR: folder missing or not a folder, an invalid
  flag. No output facts may be quoted from a failed run.

## Limits & honesty rules the agent must respect

- **CLEAN never means "safe".** It means: none of the
  nine checked surface locations held an auto-execution
  surface. Say exactly that. The project's code can still
  do anything; clone-guard never read it.
- **Scan BEFORE opening, never after.** The surfaces this
  tool finds fire when an editor or agent opens the
  folder. Running clone-guard on a folder you have already
  been working in is archaeology, not protection.
- **A finding is not an accusation.** Report the verdict
  and the verbatim command; do not summarise a project as
  "malicious" from a finding list. `postinstall` scripts
  and husky hooks are normal in honest projects — the
  point of the report is that the commands are now
  *visible before they run*.
- **Quote commands verbatim.** Never paraphrase, decode
  or "clean up" a reported command in your summary; an
  obfuscated command is itself the signal.
- **UNPARSEABLE means unknown, not clean.** A config that
  could not be parsed could not be cleared; treat the
  repo as unscanned for that surface until a human reads
  the file.
- Local only: the tool reads only the listed config files
  in the one folder named on the command line, and sends
  nothing anywhere.
