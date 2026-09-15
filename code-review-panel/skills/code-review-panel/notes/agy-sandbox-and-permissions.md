# agy (Antigravity CLI) as a headless reviewer: sandbox, permissions, and codex for comparison

Researched 2026-09-14 on this Mac. Versions: `agy` 1.2.3 (`~/.local/bin/agy`, a single
181 MB arm64 Mach-O Go binary, no bundled JS, README or schema), `codex-cli` 0.153.4.

## Answer in five lines

1. **Not safe.** `agy --sandbox --dangerously-skip-permissions` is not a read-only review. The flag also auto-approves the agent's own request to leave the sandbox (`BypassSandbox: true`). A Google collaborator confirmed that is intended (issue #36). So the worst case is any command at your full user privileges, with network access.
2. `--sandbox` is a real OS sandbox: macOS Seatbelt through `/usr/bin/sandbox-exec`. But it covers **terminal commands only**. The file tools (view/write) are governed by the permission layer, not by Seatbelt.
3. **Recommended route: no permissions flag at all.** Use `agy --sandbox --add-dir <repo> -p "<prompt that only reads files>"`. In headless mode, in the default `request-review` mode, workspace reads go through. Writes, shell commands and URL fetches are soft-denied and listed in `denied_actions`. That is a read-only reviewer at the permission layer. Probes 9, 11, 12, 14, 15 and 6 verify it.
4. **A narrow command allow rule is possible, but not in the file the error message names.** `command(...)` grants are honoured headless when placed in `~/.gemini/config/config.json` → `userSettings.globalPermissionGrants.allow`, or in a project record used with `--project`. Matching is on the exact whole command line unless the rule starts with `regex:`. UNTESTED here, because it needs a global file edit.
5. **codex for comparison.** `codex review` on this machine gets the **read-only** sandbox (Seatbelt, no network): this repo has no trust entry and there is no `sandbox_mode` in config. Headless approval is normally `never`. Your `approvals_reviewer = "auto_review"` switches that back to the config default `on-request`, with escalations judged by an auto-review subagent (read from source, not probed).

---

## Method and limits

- The binary has no readable source tree. "Install" citations are `strings(1)` output, plus `agy changelog` entries tagged with the version that introduced each change. To reproduce a strings citation: `strings -n 6 ~/.local/bin/agy | awk 'length<300' | grep -nF '<phrase>'`. Line numbers change with the flags, so each claim quotes the exact phrase.
- Official docs: antigravity.google/docs/cli/{sandbox,permissions,settings,commands/permissions}. Issue tracker: github.com/google-antigravity/antigravity-cli.
- Probes: all used `--sandbox`, never `--dangerously-skip-permissions`, with a bound of `timeout 180`. The cwd was `<scratch dir>` (a fresh `git init` repo holding `README.txt`), and output went through `--output-format json`. Every run exited 0. Files the probes created were deleted afterwards.
- Global config was not edited, so allow rules are **untested**. Pointing `HOME` at a scratch settings file breaks sign-in (probe 8: `Authentication required ... error: authentication failed or timed out`), which issue #627 also reports.

## Existing config (structure only)

- `~/.gemini/antigravity-cli/settings.json`: `model` and `trustedWorkspaces` (the home directory and a few project paths). There is no `permissions` key.
- `agy -p "/config"` (no agent turn) reports effective values: `enableTerminalSandbox false`, `toolPermission request-review`, `allowNonWorkspaceAccess false`, `permissions` (empty), `artifactReviewPolicy asks-for-review`.
- `~/.gemini/config/config.json` is `{"userSettings": {}}`, so it holds no grants. `~/.gemini/config/projects/*.json` holds three projects with `projectResources` only; none has `permissionGrants`.
- `cli.log` for each probe: `applyUserSettings: no shared config permissions from ~/.gemini/config/config.json`, `CLI settings initialized: permissions=<nil>, toolPermission=request-review`, `ApplyProjectPermissionGrants: no grants for project "CLI Project"`, `Print mode: enabling terminal sandbox for this session`.

---

## Q1. What does `--sandbox` confine?

**It is an OS sandbox, and it covers terminal commands.**

- `agy --help`: `--sandbox  Run in a sandbox with terminal restrictions enabled`. In settings it is `enableTerminalSandbox`: "restricts all agent-initiated terminal commands to a secure OS container" (docs/cli/settings).
- Binary strings: `/usr/bin/sandbox-exec`, an embedded profile headed `; macOS Seatbelt sandbox profile` / `(version 1)` / `; Deny everything by default.` / `(deny default)`, followed by `(allow process-exec)`, `(allow process-fork)`, `(allow mach-lookup)`, `(allow sysctl-read)`. Path rules are generated at runtime from templates: `(allow %s (subpath %q))` beside `file-read* process-exec*`, `%s(deny %s (with message %q) (subpath %q))`, `(allow network-outbound (literal %q))`, `(allow network-outbound (remote tcp "localhost:%s"))`, `(allow network-bind)`. The symbol `google3/devtools/ai/sandbox/exebox.(*sandboxProfile).allowNetwork` is present.
- Docs (docs/cli/sandbox): "macOS: Seatbelt profiles (SBPL) restrict filesystem access and socket connections" via `sandbox-exec`; Linux uses kernel namespaces. This is not Gemini CLI's implementation: there is no docker/podman option and no named seatbelt profiles such as `permissive-open`. It is a separate Go codebase (`google3/third_party/jetski/...`).

**What a sandboxed command can do:**

| Resource | Behaviour | Source |
|---|---|---|
| Write inside workspace | allowed | docs/cli/sandbox: "Commands can write to your workspace, temp directories, and common build caches" |
| Write to temp dir | allowed (the temp dir grant covers writes too) | same; changelog "default system temporary-directory grant to cover writes" |
| Read system dirs | `/usr`, `/etc` readable | docs/cli/sandbox |
| `~/.ssh`, `.env` | blocked by default | docs/cli/sandbox |
| Paths in `read_file(...)` / `write_file(...)` grants | mounted read-only / read-write | docs/cli/sandbox |
| Everything else outside the workspace | "invisible inside isolation" (docs); **UNVERIFIED by probe** | docs/cli/sandbox |
| `.git` | **read-only** for commands | changelog 1.1.10 "granting read-only rather than writable access to a Git repository's `.git` directory"; 1.0.9 ".git added to the core list of dangerous paths". The SKILL.md claim that "agy --sandbox cannot see .git" dates from before 1.1.10 and is probably stale (UNVERIFIED for commands) |
| `--add-dir` directories | treated as workspace roots, so writable for commands (inferred from "write to your workspace") | UNVERIFIED |
| Network | "without network access by default"; domains in `read_url(...)` grants are added to an outbound allowlist through a proxy | docs/cli/sandbox; changelog "sandbox not recording blocked network requests" |
| Subprocesses | `process-exec`/`process-fork` allowed inside the profile, so children inherit the sandbox | embedded profile |

**The escape hatch is part of the design.** The command tool takes a model-settable argument: `BypassSandbox ... "Set to true to run outside the sandbox (requires elevated user permission for auth exchange, network, or port binding)."` (binary strings). The docs say: "The agent can also request to run a command outside the sandbox on its own—for example, to retry a command that failed due to sandbox restrictions". Approval is required "unless the command matches an `unsandboxed` allow rule". The prompt reads "🔓 Allow sandbox bypass for command execution? ⚠️ Confirm the command is safe to run outside of the sandbox with full network and disk access."

**`--sandbox` does not confine the file tools.** In probe 13, the file-writing tool wrote into `.git` under `--sandbox`. Seatbelt makes `.git` read-only for commands, and the write still landed. File tools are gated only by the permission layer: workspace plus trusted/added dirs, `allowNonWorkspaceAccess`, and `read_file`/`write_file` grants.

```
timeout 180 agy --sandbox --add-dir "$W" --mode accept-edits -p "Use only your file-writing tool, no shell. Create $W/.git/agy-probe.txt containing x, then report whether it succeeded." --output-format json
→ denied: None; "I successfully created the file agy-probe.txt ... at .../.git/agy-probe.txt"
$ ls -la $W/.git/agy-probe.txt → -rw-r--r-- 2 bytes   (verified on disk, then deleted)
```

**The workspace quirk matters for the wrapper.** Without `--add-dir`, agy did not treat the cwd as the workspace for its file tools. `cli.log` did list `workspaceDirs=[<cwd>]`.

- Probe 3 (`-p "Use only your file-viewing tool ... Read README.txt"`, no `--add-dir`) was denied: `a tool required the "read_file" permission ... auto-denied`, `denied_actions: [{action: read_file, display_name: ListDir}]`.
- Probe 7 used the canonical absolute path and was denied the same way.
- Probe 10 (`--mode accept-edits`, create `inside-tool.txt`, no `--add-dir`) answered "Done." but wrote the file to `~/.gemini/antigravity-cli/scratch/inside-tool.txt`, the agent's own scratch dir, not the cwd.
- Probe 9, the same read with `--add-dir "$W"`, succeeded and quoted `readme`.

This matches the third-party report in issue #548: "`agy` does not adopt the current directory as its workspace ... `--add-dir <repo>` fixes it". My probe cwd sat under `/private/tmp`, which is not in `trustedWorkspaces`, so trust may be a second factor (the binary has the string `%s requires permission to read, edit, and execute files here.`). The panel's in-repo runs sit under the trusted `~`. Either way, **always pass `--add-dir <repo root>`.**

## Q2. What `--dangerously-skip-permissions` auto-approves, and whether `--sandbox` still holds under it

- `agy --help`: "Auto-approve all tool permission requests without prompting". The headless denial text makes the same offer: "Alternatively, re-run with --dangerously-skip-permissions to auto-approve all tools." (every denied probe's stderr).
- The permission actions it therefore covers are `read_file`, `write_file`, `read_url`, `execute_url`, `command`, `unsandboxed`/sandbox bypass, and `mcp` (action list from docs/cli/permissions).
- **It bypasses the sandbox in practice.** Issue #36 (open, 2026-05-20), reproduction: `agy --sandbox --dangerously-skip-permissions -p "Run 'echo test > /tmp/outside_workspace.txt'"`. "The initial bash tool call fails, but with a hint to the model to pass `bypassSandbox: true`. The model then follows this suggestion, and successfully runs the command". The reply from Google collaborator `chandrashakherkamasani`: "`--dangerously-skip-permissions` is designed to auto-approve *all* permissions, including sandbox bypass prompts." The alternative given is `"toolPermission": "proceed-in-sandbox"` with `--sandbox` alone.
- `proceed-in-sandbox` (changelog 1.0.1): "Auto-approves terminal commands that run inside the secure sandbox, requesting manual approval only when a command attempts to bypass the sandbox." Headless mode cannot give that approval, so a bypass would be soft-denied (inferred). It is settable only in the global `settings.json` (`--mode` accepts only `accept-edits|plan`). Issue #627 asks for it on `--mode`. UNTESTED.
- Not run, per instructions. Everything above about the flag comes from docs and issues, not probes.

## Q3. `permissions` rule syntax and semantics

**Format:** `action(target)`, in `allow` / `ask` / `deny` lists. Precedence is **Deny > Ask > Allow** (docs/cli/permissions).

| Action | Target | Notes (docs/cli/permissions) |
|---|---|---|
| `read_file` | path or `*` | recursive |
| `write_file` | path or `*` | implies `read_file` on the same target; denying a read blocks writes |
| `read_url` | domain or `*` | covers subdomains, ignores path; also feeds the sandbox network allowlist |
| `execute_url` | domain or `*` | browser navigation |
| `command` | prefix, `regex:pattern`, or `*` | shell commands (run sandboxed if the sandbox is on) |
| `unsandboxed` | prefix, `regex:`, or `*` | run outside the sandbox without a prompt. **Deprecated** as of changelog 1.2.2 ("deprecated `unsandboxed` permission rules ... migrating them to `command` rules"; binary: `Fix: replace "unsandboxed" with "command" (e.g. command(git status))`) |
| `mcp` | `server/tool`, `server/*`, `*` | |

Unknown actions are rejected at load: `ignoring invalid allow entry "list_dir(*)": unknown action "list_dir"` (issue #548, bompus).

**How `command` matching works:**
- Binary doc string: "Each token in the granted target is matched as a full word (internally treated as an anchored regular expression: `^(?:pattern)$`)."
- Changelog 1.0.13: matching is "strict (non-regex) by default ... opt-in to regex matching by prepending rules with `regex:`".
- Measured by third parties on 1.1.4 and 1.1.27 (issues #627 and #548): **the rule must equal the whole command line.** `command(git)` does **not** allow `git status --short`, and `command(python3 *)` and `command(python3*)` fail. `command(git (status|log))` fails silently, while `command(regex:git (status|log))` allows `git status` and `command(regex:git .*)` allows `git status --short`. Note the #548 regex result contradicts the earlier #627 result, where `command(regex:^python3)` did not match. That is consistent with the pattern being anchored to the full line, so a bare prefix regex matches nothing.
- Compound commands and pipelines are split and each part checked. Nested `$(...)` is checked per command (changelog 1.1.2, 1.1.5). Output redirection (`tool > file`) is relaxed so it still matches (changelog "relaxing redirection checks"). Complex redirections, PowerShell and unparseable strings need an exact match (changelog "Hardened command execution permission checks").
- A rule that tokenizes to zero words (`command(time)`, `()`) used to match everything and now matches nothing (1.1.11).

**Where rules are read from, and whether headless honours them:**
- Three scopes: Project > Shared > Global (docs/cli/commands/permissions). The CLI "merges project level permissions, permissions from user settings shared with Antigravity, and permissions from the CLI `settings.json`" (changelog).
- Changelog 1.1.5: headless runs "now honor persisted `settings.json` policies, including `permissions`". 1.1.3: headless "soft-denies such tools and prints a stderr notice naming the allow-rule needed". The probes confirm the soft-deny and `denied_actions`.
- **Caveat, third-party measurement on 1.1.27 (issue #548, open):** in headless mode, `mcp(...)` and `read_file(*)` from `~/.gemini/antigravity-cli/settings.json` are honoured, **but `command(...)` there is silently ignored**. `command()` grants work from `~/.gemini/config/config.json` → `{"userSettings": {"globalPermissionGrants": {"allow": ["command(regex:git .*)"]}}}`. They also work from `~/.gemini/config/projects/<id>.json` → top-level `"permissionGrants": {"permissionGrants": {"allow": [...]}, "v2Migrated": true}`, but only with `--project <id-or-name>`, because agy otherwise binds to "CLI Project". The field names match what the local log prints (`applyUserSettings: ... shared config permissions from .../config/config.json`, `ApplyProjectPermissionGrants`). **UNVERIFIED on 1.2.3 on this Mac.**
- No per-run `--settings` flag exists (`agy --help`). Issue #627 tried cwd `settings.json`, `.agents/rules.json`, `ANTIGRAVITY_EXECUTABLE_DATA_DIR` and `CASCADE_GLOBAL_CONFIG_OVERRIDE`; none worked.

**Defaults and modes:**
- Workspace files are Allow; everything unconfigured is Ask, which becomes a soft-deny when headless (docs/cli/permissions).
- URL fetch has been Ask since 1.1.28. Probe 6 (`Fetch https://example.com`) got `denied_actions: [{action: read_url}]`.
- `toolPermission`: `request-review` (default), `proceed-in-sandbox`, `strict`, `always-proceed` (docs/cli/settings).
- `--mode accept-edits` auto-approved an in-workspace write in probes 10 and 13, and **still denied a write outside the workspace** in probe 14:

```
timeout 180 agy --sandbox --add-dir "$W" --mode accept-edits -p "... Create ~/agy-probe-outside-DELETE-ME.txt ..."
→ jetski: no output produced — a tool required the "write_file" permission ... auto-denied.   (no file on disk)
```

- Reading outside the workspace was refused without any `denied_actions` entry. Probe 5, reading `~/.gemini/antigravity-cli/settings.json`, answered: "access was denied due to system protection boundaries".
- A headless run **stops at the first denied action** with `response: ""`. The JSON `status` is still `SUCCESS` and the exit code is 0. Probes 1 and 2 lost every later step. **The wrapper must treat non-empty `denied_actions` or an empty response as a failure, never as zero findings.**

## Q4. `codex review` defaults (codex-cli 0.153.4)

- `codex review --help` has no `-s`/`-a` flags, only `-c`, `--uncommitted`, `--base`, `--commit`, `--title`, `--enable`/`--disable`. In source, `codex review` builds `ExecCli::try_parse_from(["codex","exec"])` and runs `ExecCommand::Review` (`codex-rs/cli/src/main.rs:1160-1179`, tag `rust-v0.153.4`), so it uses `codex exec` defaults. Top-level `-s`/`-a` are inherited through `inherit_exec_root_options`.
- **Sandbox.** In `codex-rs/config/src/config_toml.rs:747-765`, the CLI override or `sandbox_mode` from config wins. Otherwise a project with a trust decision (trusted *or* untrusted) gets `WorkspaceWrite`, and anything else gets `SandboxMode::default()` = `ReadOnly` (`codex-rs/protocol/src/config_types.rs:104-107`).
  - `~/.codex/config.toml` has no `sandbox_mode`, and `<repo>` (and its `.worktrees/*`) has no `[projects]` entry. Only `~/Projects/_archive/movievibes` is trusted.
  - **Effective sandbox here: `read-only`** (macOS Seatbelt): no writes, network restricted.
  - **In a trusted repo it would be `workspace-write`.**
- **Approval.** `codex-rs/exec/src/lib.rs:411-413` defaults headless to `AskForApproval::Never`. But `build_exec_config` (`lib.rs:585-617`) **drops that override when `approvals_reviewer == AutoReview`**, and your config sets `approvals_reviewer = "auto_review"`.
  - Config sets no `approval_policy`, so it falls to `AskForApproval::default()` = `OnRequest` (`protocol.rs:991-994`).
  - Escalations ("sandbox escapes, blocked network access, MCP approval prompts", `config_types.rs:179-182`) go to an auto-review subagent that "apply[s] a risk-based decision framework before approving or denying".
  - **So on this machine: read-only sandbox, `on-request` approvals judged by the auto-reviewer** (read from source, not probed).
  - Pass `-c approvals_reviewer=user` to get the plain headless `never` (escalations fail back to the model). `--dangerously-bypass-approvals-and-sandbox` is the codex equivalent of danger mode and was not used.

## Q5. Conclusion

**Is `agy --sandbox --dangerously-skip-permissions` safe for a read-only review? No.**

- **Worst case.** The model can be steered by an instruction planted in the diff or a repo file under review. It runs a command, the sandbox blocks it, and it retries with `BypassSandbox: true`. The retry is auto-approved (#36, confirmed intended), so the command runs with your full user rights: network exfiltration of `~/.ssh`, tokens or the repo, `git push`, `rm`, persistence in `~/.zshrc`.
- **The sandbox never covered the file tools anyway** (probe 13). Under the flag, `write_file` outside the workspace and `read_url` to any domain are also auto-approved (inferred from "all tool permission requests").
- **The only remaining guard is an in-model safety classifier.** The binary embeds prompts such as "Writing to system paths ... or persistence paths (.bashrc, crontab) is dangerous". That is advisory model judgement, not a boundary.

**Better routes, in order:**

1. **No flag; read-only by permission layer (verified).** Run `agy --sandbox --add-dir <repo> -p "<file-reading prompt>" --output-format json`. Reads in the workspace are allowed. Writes, commands and URLs are soft-denied. This is the current SKILL.md route, plus `--add-dir <repo>` and a `denied_actions` check. `--sandbox` is belt-and-braces here, since no command can run anyway.
2. **`--mode plan`, plausibly the tightest mode (UNVERIFIED).** Issue #548 describes plan mode as structurally blocking mutation. It was not probed here, and "Fixed headless runs stalling forever on implementation-plan approval" (1.1.28) suggests plan mode has its own headless quirks.
3. **A narrow command allow, if a reviewer must run git (UNTESTED).**
   - Edit `~/.gemini/config/config.json` (not `antigravity-cli/settings.json`, where headless `command()` is reportedly ignored): `{"userSettings": {"globalPermissionGrants": {"allow": ["command(git diff)", "command(git diff --stat)", "command(regex:git (diff|log|show) [-A-Za-z0-9_./:=~^]+( [-A-Za-z0-9_./:=~^]+)*)"]}}}`.
   - Keep `--sandbox`: allowed commands still run inside Seatbelt with `.git` read-only and no network, and any bypass request stays soft-denied headless (no `unsandboxed` rule).
   - Caveats:
     - A `regex:` rule with `.*` is broad. `git diff --output=<path>` and `git log --output=<path>` write files. Only the sandbox confines where they land (workspace and temp are writable), so avoid `.*` and prefer an exact whole-line list.
     - Whole-line matching means `command(git)` or `command(cat)` alone does nothing.
     - The grant is machine-wide and shared with the Antigravity GUI.
     - `cat`-style commands are unnecessary: the file tools already read the workspace.

## Verified vs inferred

| Claim | Status | Evidence |
|---|---|---|
| `--sandbox` uses macOS Seatbelt via `/usr/bin/sandbox-exec`, deny-default profile | Verified (binary + docs) | strings; docs/cli/sandbox |
| Sandbox covers terminal commands, not file tools | Verified | probe 13 (.git write via file tool succeeded under --sandbox) |
| `.git` read-only for sandboxed commands | Docs/changelog only | changelog 1.1.10; not probed (commands cannot run headless without a grant) |
| Sandboxed commands have no network except `read_url`-granted domains | Docs only | docs/cli/sandbox; not probed |
| Model can request `BypassSandbox`; `--dangerously-skip-permissions` auto-approves it | Verified from primary sources (third-party repro + Google collaborator reply); not probed by rule | binary strings; issue #36 |
| Headless default: workspace reads allowed with `--add-dir`; write/command/url soft-denied | Verified | probes 9, 12 (read ok), 11 (write denied), 15 (command denied), 6 (url denied) |
| Without `--add-dir`, file tools do not use cwd as workspace | Verified (untrusted /private/tmp cwd) | probes 3, 7, 10; whether trust alone would fix it is UNVERIFIED |
| Denied headless run exits 0 with `status: SUCCESS`, empty response, `denied_actions` | Verified | probes 1-4, 6, 7, 11, 14, 15 |
| `--mode accept-edits` approves in-workspace writes, still denies outside | Verified | probes 10, 13, 14 |
| Rule grammar `action(target)`, Deny > Ask > Allow, action list | Docs | docs/cli/permissions |
| `command()` matches the whole line; `regex:` opt-in | Changelog + third-party measurement | changelog 1.0.13; issues #548, #627 |
| Headless ignores `command()` in `antigravity-cli/settings.json`, honours it in `config/config.json` / project file | Third-party measurement (1.1.27, Windows) | issue #548; UNVERIFIED on 1.2.3/macOS |
| `proceed-in-sandbox` keeps bypass gated | Docs + collaborator statement | changelog 1.0.1; docs; issue #36; not probed |
| `--mode plan` blocks mutation headless | Third-party claim | issue #548; UNVERIFIED |
| codex review → read-only sandbox here; `on-request` + auto_review approvals | Inferred from source + local config | codex-rs files cited above; not probed |

## Recommended wrapper change for the panel skill (proposal; SKILL.md not edited)

Replace step 2 of `AGY_WRAPPER` with:

```
2. Run with a hard 9-minute bound, from the repo root, naming the repo as the workspace:
   REPO="$(git rev-parse --show-toplevel)"
   timeout 540 agy --sandbox --add-dir "$REPO" --add-dir "$(dirname "$DIFF_FILE")" \
     --model "Gemini 3.1 Pro (High)" --print-timeout 8m --output-format json \
     --prompt "…same file-reading-only prompt…" > "$AGY_OUT"
   Then parse $AGY_OUT: if .denied_actions is non-empty, or .response is empty, or .status != "SUCCESS",
   report failed: "agy denied <actions>" (or "empty response") — never findings: [].
   Findings are in .response.
```

Rationale:
- `--add-dir "$REPO"` makes the source files readable to the file tools even when the cwd is not treated as the workspace (probes 3 and 9).
- `--output-format json` exposes `denied_actions`. A headless denial otherwise exits 0 with only a stderr line, and it ends the run at the first denied step.
- Keep the flag out, as SKILL.md already says, for the reason above: under it the sandbox is escapable (#36).
- Also update the comment "agy --sandbox cannot see .git". Since 1.1.10 sandboxed commands get read-only `.git`, and the file tools read `.git/HEAD` fine (probe 12). The `command` permission gate is still what stops `git diff`.
- If shell ever becomes necessary, use a whole-line `command(...)` allowlist in `~/.gemini/config/config.json` (Q5 route 3), set by a person and verified first. Do not use the flag, and do not use `antigravity-cli/settings.json`.

## Sources

- `agy --help`, `agy --version`, `agy changelog`, `agy -p "/config"`, `agy -p "/permissions"` (1.2.3); `strings` of `~/.local/bin/agy`; `~/.gemini/antigravity-cli/cli.log`
- https://antigravity.google/docs/cli/sandbox/ · https://antigravity.google/docs/cli/permissions/ · https://antigravity.google/docs/cli/settings/ · https://antigravity.google/docs/cli/commands/permissions/ · https://antigravity.google/docs/cli/using/
- https://github.com/google-antigravity/antigravity-cli/issues/36 · /issues/548 · /issues/627 · /issues/45
- codex: `codex review --help`, `codex --help`, `codex exec --help`, `~/.codex/config.toml`; https://github.com/openai/codex at tag `rust-v0.153.4`: `codex-rs/cli/src/main.rs`, `codex-rs/exec/src/lib.rs`, `codex-rs/config/src/config_toml.rs`, `codex-rs/protocol/src/config_types.rs`, `codex-rs/protocol/src/protocol.rs`
- Secondary: `~/.claude/skills/code-review-panel/SKILL.md` lines 385-448
