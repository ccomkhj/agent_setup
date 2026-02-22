# Skills Repo

Source of truth for my skills and Cursor command files.

## Structure

- `skills/`: shared skills (for Cursor + other agents/tools).
- `commands/`: Cursor-only command files.

## Universal setup

Use this repo as the single source, then link each client to it.

```bash
# 1) clone
git clone <repo-url> ~/.agents
```

### Cursor

```bash
mkdir -p ~/.cursor
ln -sfn ~/.agents/commands ~/.cursor/commands
```

### Other clients (Codex, etc.)

Point each client skill path to this repo's `skills/`:

```bash
ln -sfn ~/.agents/skills <client_skills_dir>
```

If a client already has built-in/default skills, link `~/.agents/skills` as a subfolder instead of replacing the whole directory.

## Preference

Use skills first.  
Use commands only for Cursor edge cases where commands are more practical.
