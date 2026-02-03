# Auto-approving Claude Code permission requests with a hook

Inspired by [Boris Cherny's suggestion](https://x.com/bcherny/status/2017742755737555434) to route permission requests to a model via a hook.

Docs: [PermissionRequest hooks](https://code.claude.com/docs/en/hooks#permissionrequest)

## The problem

Claude Code prompts you for permission before running commands, editing files, or fetching URLs that aren't pre-allowed. This is good for safety, but interrupts flow when you're actively working. Your `settings.local.json` accumulates an ever-growing allowlist of one-off commands.

## Two approaches

### Option 1: Auto-approve everything

The simplest version. A command hook that responds to every permission request with "allow". Equivalent to `--dangerously-skip-permissions` but scoped to `PermissionRequest` (the permission dialog) rather than bypassing the entire permission system.

Add to `.claude/settings.local.json`:

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"hookSpecificOutput\":{\"hookEventName\":\"PermissionRequest\",\"decision\":{\"behavior\":\"allow\"}}}'"
          }
        ]
      }
    ]
  }
}
```

Pros: zero latency, no API calls.
Cons: no security review -- everything gets approved.

### Option 2: Route to Claude for review

A command hook that pipes the permission request to `claude -p` (non-interactive mode) for security evaluation. Uses your existing Claude Code auth -- no separate API key needed.

The script sends the permission request JSON to Opus 4.5 which responds ALLOW or DENY. Safe operations (build, test, git, file edits) get auto-approved. Dangerous operations (destructive commands, data exfiltration, sensitive file writes) get denied and you see the permission dialog as usual.

You can swap the model by changing `--model claude-opus-4-5-20251101` in the script.

## Install as a plugin

```
/plugin marketplace add recursechat/agent-workflow
/plugin install yolo@recursechat-agent-workflow
```

This installs the review hook and the `/yolo` skill for toggling modes.

## Install manually

**Step 1.** Save `plugins/yolo/scripts/permission-review.sh` to `.claude/hooks/` and make it executable:

```bash
cp plugins/yolo/scripts/permission-review.sh .claude/hooks/
chmod +x .claude/hooks/permission-review.sh
```

**Step 2.** Add to `.claude/settings.local.json`:

```json
{
  "hooks": {
    "PermissionRequest": [
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
  }
}
```

## Testing

Test the script standalone before hooking it up:

```bash
# Should return allow
echo '{"tool_name":"Bash","tool_input":{"command":"npm test"}}' | .claude/hooks/permission-review.sh

# Should return deny
echo '{"tool_name":"Bash","tool_input":{"command":"rm -rf /"}}' | .claude/hooks/permission-review.sh
```

Then restart your Claude Code session and trigger a real permission request by asking Claude to do something not in your allowlist.

## What doesn't work

The docs list `type: "prompt"` hooks as supporting `PermissionRequest`, but in practice they don't fire. The prompt hook returns `{"ok": true}` and Claude Code is supposed to translate that to the `hookSpecificOutput` format, but the translation doesn't happen for this event. Only `type: "command"` hooks work with `PermissionRequest`. Tested on Claude Code 2.1.29.

## Notes

- Restart your session after editing hooks -- they're snapshotted at startup
- `PermissionRequest` hooks don't fire in non-interactive mode (`-p`)
- Use `claude --debug` and `Ctrl+O` (verbose mode) to see hook execution
- Check `/hooks` in Claude Code to verify hooks are loaded
