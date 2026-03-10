# Claude Code Dotfiles — Auto-Sync Your Config Across Machines

![Claude Code](https://img.shields.io/badge/Claude_Code-5C4EE5?logo=anthropic&logoColor=white)
![Shell](https://img.shields.io/badge/Shell-zsh%20%7C%20bash-89E051?logo=gnubash&logoColor=white)
![Git](https://img.shields.io/badge/Sync-Git-F05032?logo=git&logoColor=white)

> **Stop re-configuring Claude Code every time you switch computers.**
> This guide shows you how to version your Claude Code dotfiles and keep them automatically in sync between your work laptop, personal laptop, and any future machine — using nothing but Git and a shell function.

---

## What This Does

A **dotfiles setup for Claude Code** — version your `~/.claude` directory in Git so your AI coding environment follows you everywhere.

---

## Why This Exists

If you use [Claude Code](https://docs.anthropic.com/en/docs/claude-code) as your AI coding assistant, you probably have:

- A carefully crafted `CLAUDE.md` with instructions, preferences, and context
- Custom slash commands in `commands/` you built over time
- A `settings.json` tuned to your model and preferences
- Maybe hooks, agents, or rules that make Claude behave exactly the way you want

The problem? **All of that lives in `~/.claude` and goes nowhere.** Switch to your work machine and you're starting from zero. Get a new laptop and you lose everything.

This dotfiles repo solves that with a simple, no-dependency approach: treat `~/.claude` as a Git repo, sync it to GitHub, and add a one-function shell wrapper that pulls on open and pushes on close.

---

## What Gets Synced

| File / Folder | What It Is |
|---------------|------------|
| `CLAUDE.md` | Your global instructions, preferences, and context for Claude |
| `settings.json` | Model selection, plugins, and environment config |
| `commands/` | Custom slash commands |
| `hooks/` | Shell hooks that run before/after Claude tool calls |
| `agents/` | Custom sub-agents with specialized behaviors |
| `rules/` | Reusable rule sets for specific workflows |

**What does NOT get synced** (by design):

| Ignored | Why |
|---------|-----|
| `cache/`, `backups/`, `file-history/` | Local ephemeral data, irrelevant on other machines |
| `history.jsonl` | Conversation history — private and machine-specific |
| `projects/`, `shell-snapshots/`, `paste-cache/` | Machine-specific runtime data |
| `plugins/`, `plans/`, `session-env/` | Local runtime state |
| `*.log`, `sessions/`, `transcripts/` | Temporary files |
| `*.key`, `*.pem`, `credentials.json`, `*.token` | **Never version secrets** |

---

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed: `npm install -g @anthropic-ai/claude-code`
- [Git](https://git-scm.com/) installed
- [GitHub CLI](https://cli.github.com/) (`gh`) — optional but recommended
- A GitHub account

---

## Part 1 — First-Time Setup

Do this once, on your main machine.

### Step 1 — Create the GitHub repo

```bash
# Option A: using GitHub CLI
gh repo create claude-code-dotfiles --public

# Option B: go to github.com/new and create it manually
```

> 💡 **Public or private?** Public repos let the community learn from your config. Private keeps it just for you. Either works — just never put secrets in `CLAUDE.md`.

### Step 2 — Initialize git in `~/.claude`

```bash
cd ~/.claude
git init
git remote add origin git@github.com:YOUR-USERNAME/claude-code-dotfiles.git
```

### Step 3 — Create the `.gitignore`

This is the most important step. Use an explicit allowlist so only the files you choose get versioned.

```bash
cat > ~/.claude/.gitignore << 'EOF'
# Never version these
.api_key
credentials.json
secrets/
*.key
*.pem
auth.json
*.token

# Claude Code runtime / cache (machine-specific, not portable)
cache/
backups/
file-history/
history.jsonl
paste-cache/
plugins/
projects/
shell-snapshots/
plans/
session-env/
transcripts/
sessions/
*.log
clipboard/
project_clones/
.cache/

# Always version these (explicit allowlist)
!CLAUDE.md
!settings.json
!commands/
!skills/
!hooks/
!agents/
!rules/
EOF
```

### Step 4 — First commit and push

```bash
git add CLAUDE.md settings.json commands/
git commit -m "Initial Claude Code config"
git push -u origin main
```

Your config is now on GitHub.

---

## Part 2 — Setting Up on a New Machine

Do this on every other computer where you want your config.

### Step 1 — Back up any existing config

```bash
mv ~/.claude ~/.claude.bak
```

### Step 2 — Clone the repo as `~/.claude`

```bash
git clone git@github.com:YOUR-USERNAME/claude-code-dotfiles.git ~/.claude
```

Your `CLAUDE.md`, commands, and settings are now in place instantly.

### Step 3 — Add the auto-sync function to your shell

Open `~/.zshrc` (or `~/.bashrc`) and add:

```bash
# Auto-sync Claude Code config across machines
claude() {
  echo "🔄 Syncing Claude config..."
  git -C ~/.claude pull origin main --quiet

  command claude "$@"

  echo "💾 Saving config changes..."
  git -C ~/.claude add CLAUDE.md settings.json commands/ hooks/ agents/ rules/ 2>/dev/null
  if ! git -C ~/.claude diff --cached --quiet; then
    git -C ~/.claude commit -m "chore: sync config $(date '+%Y-%m-%d %H:%M')"
    git -C ~/.claude push origin HEAD:main --quiet
    echo "✅ Config updated on GitHub"
  else
    echo "✅ No changes"
  fi
}
```

Reload your shell:

```bash
source ~/.zshrc
```

### How it works from here

Every time you run `claude`:

1. It **pulls** the latest config from GitHub before the session starts
2. Opens Claude Code normally
3. When you exit, it **commits and pushes** any changes automatically

---

## Part 3 — Customizing Your Config

### CLAUDE.md — Your global instructions

This is the most powerful file. Claude reads it at the start of every session. Use it to define:

- Your preferred coding style and conventions
- How you want Claude to communicate (tone, verbosity)
- Rules that apply to every project
- Patterns and preferences you never want to repeat

Example:

```markdown
## Code Style
- Always use snake_case for Python variables
- Prefer explicit over implicit
- Never add comments unless the logic is non-obvious

## Communication
- Be concise. No preamble.
- No emojis unless I ask.
- When in doubt, ask before making irreversible changes.
```

> ⚠️ **Security reminder**: If your repo is public, `CLAUDE.md` is public too. Never put account IDs, API keys, internal URLs, or personal information in it.

### Custom slash commands

Place `.md` files in `~/.claude/commands/`. Each file becomes a `/command-name` you can invoke during any Claude session.

Example — `commands/summarize-pr.md`:

```markdown
You are a senior engineer reviewing a pull request.
Summarize the changes in 3 bullet points, focusing on what changed and why.
Then give one concern and one thing done well.
```

Now `/summarize-pr` is available in every session, on every machine.

---

## Optional — Sync Commands to Kiro (macOS only)

> This section is optional and only applies if you use [Kiro](https://kiro.dev), AWS's AI IDE for macOS.

If you use both Claude Code and Kiro, you can reuse the same slash commands in both tools. Claude Code commands live in `~/.claude/commands/` as `.md` files with a Claude-specific YAML frontmatter. Kiro reads steering files from `~/.kiro/steering/` with a different frontmatter format.

The script below converts your Claude commands to Kiro steering files automatically — stripping Claude's frontmatter and adding Kiro's `inclusion: manual` header.

### Setup

**Step 1 — Save the script**

```bash
mkdir -p ~/.claude/hooks
cat > ~/.claude/hooks/sync-to-kiro.sh << 'EOF'
#!/bin/bash
# Syncs ~/.claude/commands/*.md → ~/.kiro/steering/
# Strips Claude frontmatter (name/description) and adds Kiro's inclusion: manual

CLAUDE_COMMANDS="$HOME/.claude/commands"
KIRO_STEERING="$HOME/.kiro/steering"

mkdir -p "$KIRO_STEERING"

synced=0
deleted=0

# Sync new and modified files
for src in "$CLAUDE_COMMANDS"/*.md; do
    [ -f "$src" ] || continue

    filename=$(basename "$src")
    dst="$KIRO_STEERING/$filename"

    # Extract content after Claude's frontmatter (after the second ---)
    content=$(awk 'BEGIN{count=0} /^---/{count++; next} count>=2{print}' "$src")

    new_content="---
inclusion: manual
---
$content"

    # Only write if content changed
    if [ ! -f "$dst" ] || [ "$new_content" != "$(cat "$dst")" ]; then
        printf '%s\n' "$new_content" > "$dst"
        echo "[sync-to-kiro] Updated: $filename"
        ((synced++))
    fi
done

# Remove steering files that no longer have a matching command
for dst in "$KIRO_STEERING"/*.md; do
    [ -f "$dst" ] || continue
    filename=$(basename "$dst")
    if [ ! -f "$CLAUDE_COMMANDS/$filename" ]; then
        rm "$dst"
        echo "[sync-to-kiro] Removed: $filename"
        ((deleted++))
    fi
done

if [ "$synced" -eq 0 ] && [ "$deleted" -eq 0 ]; then
    echo "[sync-to-kiro] Nothing to sync."
fi
EOF
chmod +x ~/.claude/hooks/sync-to-kiro.sh
```

**Step 2 — Run it manually or hook it to your workflow**

```bash
bash ~/.claude/hooks/sync-to-kiro.sh
```

Or add it to your auto-sync shell function so it runs every time you exit Claude:

```bash
claude() {
  git -C ~/.claude pull origin main --quiet
  command claude "$@"
  bash ~/.claude/hooks/sync-to-kiro.sh
  # ... rest of your sync logic
}
```

### How it works

- Reads every `.md` file in `~/.claude/commands/`
- Strips the Claude YAML frontmatter block (`name`, `description`)
- Writes the file to `~/.kiro/steering/` with `inclusion: manual` as the new header
- Deletes any Kiro steering file whose source command no longer exists
- Skips files that haven't changed (content diff before writing)

> **Note:** The `inclusion: manual` setting means the steering file is only applied when you explicitly reference it in Kiro — it won't be injected into every conversation automatically.

---

## Troubleshooting

**`git push` fails with "non-fast-forward"**

Another machine pushed changes first. Pull and rebase:

```bash
git -C ~/.claude pull --rebase origin main
git -C ~/.claude push origin HEAD:main
```

**Claude doesn't pick up `CLAUDE.md` changes**

Claude reads `CLAUDE.md` at session start. Exit and reopen Claude for changes to take effect.

**I accidentally committed a secret**

Remove it immediately:

```bash
git -C ~/.claude rm --cached path/to/secret
git -C ~/.claude commit -m "remove secret"
git -C ~/.claude push origin HEAD:main
```

Then rotate the secret — pushing to GitHub exposes it even after deletion unless you rewrite history.

---

## Security Best Practices

- ✅ Use an explicit allowlist in `.gitignore` rather than a blocklist
- ✅ Never put credentials, API keys, or account IDs in `CLAUDE.md` or `settings.json`
- ✅ Keep cloud credentials in their respective CLI config files (`~/.aws/credentials`, etc.), not here
- ✅ If using a public repo, review every commit before pushing
- ✅ Use SSH (`git@github.com:...`) instead of HTTPS for the remote
