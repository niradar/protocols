# Claude Code Sandbox Setup Protocol (WSL2 on Windows)

**Author:** Nir Adar ([niradar@gmail.com](mailto:niradar@gmail.com))
**Date:** February 23, 2026

> **Disclaimer:** This protocol is provided as-is, for informational purposes only. It reflects my personal experience setting up Claude Code sandboxing and may not cover all edge cases or security scenarios. Use at your own risk. The author assumes no responsibility for any damage, data loss, or security issues resulting from following this guide. Always review the [official Claude Code documentation](https://code.claude.com/docs/en/sandboxing) for the most up-to-date information. Security configurations should be validated for your specific environment and threat model.

---

Placeholders — substitute with your values:

- `<WSL_USER>` = your Linux username (created on first Ubuntu launch)
- `<WIN_USER>` = your Windows username (for `/mnt/c/Users/<WIN_USER>/...`)
- `<PROJECT_DIR>` = your project folder path

---

# Part 1: First-Time Machine Setup (do once)

## Step 1: Install WSL2

Open **PowerShell as Administrator** and run:

```powershell
wsl --install
```

Restart your PC. Open "Ubuntu" from the Start menu. It will ask you to create a username and password — complete the setup (this is what makes `sudo` available).

Sanity check (in PowerShell):

```powershell
wsl -l -v
```

Make sure your distro shows **VERSION 2**. WSL1 is not supported (bubblewrap requires WSL2 kernel features).

## Step 2: Update Ubuntu and install basic tooling

Inside the Ubuntu terminal:

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y curl git ca-certificates jq
```

## Step 3: Install Node.js via nvm

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

Close and reopen the terminal (or run `source ~/.bashrc`), then:

```bash
nvm install --lts
```

Verify:

```bash
node --version
npm --version
```

If `nvm` is still "not found", fully restart WSL from PowerShell:

```powershell
wsl --shutdown
```

Then reopen Ubuntu and run `source ~/.bashrc` again.

## Step 4: Install Claude Code and sandbox dependencies

```bash
npm install -g @anthropic-ai/claude-code
npm install -g @anthropic-ai/sandbox-runtime
sudo apt-get install -y bubblewrap socat
```

## Step 5: Login to Claude Code

Launch Claude from any folder:

```bash
claude
```

Run `/login` and complete authentication. Then exit (`/exit`).

## Step 6: Optional — Set up user-wide failure logging

Add hooks to `~/.claude/settings.json` to log blocked Bash commands across all projects:

```json
{
  "hooks": {
    "PostToolUseFailure": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '{time: now | strftime(\"%Y-%m-%d %H:%M:%S\"), command: .tool_input.command, error: (.error | split(\"\\n\") | last | select(. != \"\") // .error[:200])}' >> ~/claude-bash-failures.log"
          }
        ]
      }
    ]
  }
}
```

Example log output:
```
{"time":"2026-02-23 14:30:00","command":"curl https://example.com","error":"curl: (56) CONNECT tunnel failed, response 403"}
```

---

# Part 2: Per-Project Setup (do for each new project)

## Step 1: Navigate to the project folder

**Critical**: The sandbox scopes write access to the directory you launch Claude from. Do **not** start Claude from `~`.

**Option A — Linux filesystem (recommended, faster I/O):**

```bash
mkdir -p ~/projects
cd ~/projects/<your-project>
```

**Option B — Windows-mounted folder:**

```bash
cd /mnt/c/<PROJECT_DIR>
```

## Step 2: Enable sandbox for this project

```bash
claude
```

Inside Claude, run:

```
/sandbox
```

Configure:
1. **Mode**: Select "Sandbox BashTool, with auto-allow"
2. **Overrides**: Select "Strict sandbox mode" — this disables the escape hatch that retries commands outside the sandbox

These settings are saved per-project. You need to do this for each new project.

## Step 3: Create project settings file

Create `.claude/settings.local.json` in your project root (this file is gitignored automatically):

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "allowUnsandboxedCommands": false,
    "excludedCommands": ["docker"],
    "network": {
      "allowedDomains": [
        "registry.npmjs.org",
        "*.npmjs.org",
        "pypi.org",
        "files.pythonhosted.org",
        "raw.githubusercontent.com",
        "objects.githubusercontent.com"
      ],
      "allowLocalBinding": false
    }
  },
  "permissions": {
    "defaultMode": "dontAsk",
    "allow": [
      "Read",
      "Edit",
      "Write",
      "WebFetch",
      "WebSearch",
      "Task"
    ],
    "deny": [
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)",
      "Read(~/.gnupg/**)",
      "Read(~/.config/**)",
      "Read(//mnt/c/Users/**)",
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Read(**/secrets/**)"
    ]
  }
}
```

Add more domains to `allowedDomains` as needed for your project's tools.

### What each section does

| Setting | Purpose |
|---------|---------|
| `allowUnsandboxedCommands: false` | No escape hatch — commands that can't run sandboxed just fail |
| `excludedCommands: ["docker"]` | Docker is incompatible with bubblewrap — runs outside sandbox with normal permissions |
| `allowedDomains` | Only these domains reachable from Bash (npm, PyPI, GitHub raw files) |
| `allowLocalBinding: false` | Prevents Bash from binding to local ports (blocks exfiltration via localhost listeners) |
| `defaultMode: "dontAsk"` | Auto-denies any tool not in the `allow` list — no prompts, fully autonomous |
| `allow` list | Tools Claude can use without prompting (file ops, web research, subagents) |
| `deny` list | Sensitive paths blocked from reading even though `Read` is allowed globally |

## Step 4: Restart Claude and verify

```bash
claude
```

Verification checklist:

1. **Read denied secrets**: Ask "Read ~/.ssh/config" — should be **blocked**
2. **Read Windows user folder**: Ask "List files in /mnt/c/Users/<WIN_USER>/Desktop" — should be **blocked**
3. **Write outside project**: Ask "Create a file in ~/tmp-test" — should **fail**
4. **Network restriction**: Ask Claude to run `curl https://example.com` — should **fail** (not in allowedDomains)
5. **Normal project work**: Ask "Create a file in the project folder" — should **work**
6. **Check permissions**: Run `/permissions` to confirm rules and which settings file they came from

---

# Reference

## What the sandbox protects

| Layer | What it does |
|-------|-------------|
| Filesystem (writes) | Only your project folder is writable |
| Filesystem (reads) | Entire machine readable by default — use `permissions.deny` to block sensitive paths |
| Network (Bash) | Only `allowedDomains` reachable from Bash commands |
| Network (WebFetch/WebSearch) | Separate from sandbox, open for research — safe because these tools can't access local files |

## Permission path patterns (important!)

When adding paths to `allow` or `deny` rules, the prefix determines how the path is resolved:

| Pattern | Meaning | Example |
|---------|---------|---------|
| `//path` | **Absolute** path from filesystem root | `Read(//mnt/c/Users/**)` |
| `~/path` | Path from **home directory** | `Read(~/.ssh/**)` |
| `/path` | Path relative to **settings file** | `Read(/src/**)` → `<project>/.claude/src/**` |
| `path` or `./path` | Path relative to **current directory** | `Read(*.env)` → `<cwd>/*.env` |

**Common mistake**: Using `/mnt/c/Users/**` (single slash) — this is treated as relative to the settings file, not as an absolute path. Always use `//` for absolute paths.

**Wildcards**: `*` matches files in a single directory, `**` matches recursively across directories.

## Adding new domains

If a tool fails due to network restrictions, add the needed domain to `sandbox.network.allowedDomains` in your settings file. Only add what you actually need. Be cautious with broad domains like `github.com` — they could be used for data exfiltration.

## dontAsk mode

`dontAsk` auto-denies any tool not in your `allow` list. Combined with sandbox auto-allow, this means:
- Sandboxed Bash → runs automatically
- Allowed tools (Read, Edit, etc.) → run automatically
- Everything else → silently denied

This is what enables fully unattended operation.

## Common Gotchas

- **"Strict mode didn't change anything"** — usually means either: you launched Claude from `~` (so the entire home dir is the project), or `allowUnsandboxedCommands` is still `true`
- **WSL "restart" is not a full Linux restart** — closing the terminal doesn't restart WSL. Use `wsl --shutdown` from PowerShell
- **Sandbox alone doesn't block reads** — the sandbox only restricts writes. Use `permissions.deny` rules for read restrictions
- **`allowedDomains` nesting** — must be under `sandbox.network.allowedDomains`, NOT `sandbox.allowedDomains`
- **Docker in sandbox** — docker can't run inside bubblewrap. Add it to `excludedCommands` so it runs outside with normal permissions

---

Reference docs:
- [Sandboxing](https://code.claude.com/docs/en/sandboxing)
- [Permissions](https://code.claude.com/docs/en/permissions)
- [Settings](https://code.claude.com/docs/en/settings)
- [Hooks](https://code.claude.com/docs/en/hooks-guide)
