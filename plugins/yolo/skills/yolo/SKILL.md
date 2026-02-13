---
name: yolo
description: Set up PermissionRequest hooks for the current project or user
argument-hint: [approve-all|approve-websearch|review|off] [--global]
disable-model-invocation: true
---

Set up a PermissionRequest hook that auto-handles permission prompts.

## Arguments

- `/yolo approve-all` — auto-approve everything (no security review)
- `/yolo approve-websearch` — auto-approve WebSearch and WebFetch only; all other tools fall through to the normal permission dialog
- `/yolo review` — route each permission request to Claude for security review via `claude -p`
- `/yolo off` — remove the PermissionRequest hook
- Add `--global` to any command to apply to `~/.claude/settings.json` instead of the project's `.claude/settings.local.json`

If no argument is given, default to `review`.

## Instructions

1. Determine the target settings file:
   - If `--global` is in the arguments: `~/.claude/settings.json`
   - Otherwise: `.claude/settings.local.json` in the project root

2. Read the target settings file. If it doesn't exist, start with `{}`.

3. Based on the mode:

### `approve-all`

Set `hooks.PermissionRequest` in the settings to:

```json
[
  {
    "hooks": [
      {
        "type": "command",
        "command": "INPUT=$(cat); TOOL=$(echo \"$INPUT\" | grep -o '\"tool_name\":\"[^\"]*\"' | head -1 | sed 's/\"tool_name\":\"//;s/\"//'); if [ \"$TOOL\" = \"AskUserQuestion\" ]; then echo '{}'; else echo '{\"hookSpecificOutput\":{\"hookEventName\":\"PermissionRequest\",\"decision\":{\"behavior\":\"allow\"}}}'; fi"
      }
    ]
  }
]
```

### `approve-websearch`

Set `hooks.PermissionRequest` in the settings to:

```json
[
  {
    "hooks": [
      {
        "type": "command",
        "command": "INPUT=$(cat); TOOL=$(echo \"$INPUT\" | grep -o '\"tool_name\":\"[^\"]*\"' | head -1 | sed 's/\"tool_name\":\"//;s/\"//'); if [ \"$TOOL\" = \"AskUserQuestion\" ]; then echo '{}'; elif [ \"$TOOL\" = \"WebSearch\" ] || [ \"$TOOL\" = \"WebFetch\" ]; then echo '{\"hookSpecificOutput\":{\"hookEventName\":\"PermissionRequest\",\"decision\":{\"behavior\":\"allow\"}}}'; else echo '{}'; fi"
      }
    ]
  }
]
```

### `review`

First, create `.claude/hooks/permission-review.sh` in the target location (project root or `~/.claude/hooks/` for global) with the contents of [permission-review.sh](permission-review.sh). Make it executable.

Then set `hooks.PermissionRequest` in the settings to:

```json
[
  {
    "hooks": [
      {
        "type": "command",
        "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/permission-review.sh",
        "timeout": 30
      }
    ]
  }
]
```

For `--global`, use the absolute path `~/.claude/hooks/permission-review.sh` instead of `$CLAUDE_PROJECT_DIR`.

### `off`

Remove the `PermissionRequest` key from `hooks` in the settings. If `hooks` is then empty, remove `hooks` too.

4. Write the updated settings file, preserving all other keys (permissions, etc.).

5. Tell the user to restart their Claude Code session for hooks to take effect.
