# CodeBuddy Hook

## Overview

CodeBuddy integration uses a Rust binary hook (`rtk hook codebuddy`) that intercepts `PreToolUse` events and rewrites Bash commands for token optimization.

## Installation

```bash
# Global installation (recommended)
rtk init --global --codebuddy

# Project-local installation
rtk init --codebuddy
```

This creates:
- `~/.codebuddy/settings.json` (global) or `<project>/.codebuddy/settings.json` (local)
- `~/.codebuddy/CODEBUDDY_INSTRUCTIONS.md` (global) or `<project>/.codebuddy/CODEBUDDY_INSTRUCTIONS.md` (local)

## JSON Format

### Input (stdin)

CodeBuddy sends the following JSON payload to the hook via stdin:

```json
{
  "session_id": "abc123",
  "transcript_path": "/Users/.../.codebuddy/projects/.../session.jsonl",
  "cwd": "/Users/cloudxiao/myproject",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "git status",
    "timeout": 30000,
    "description": "Check repo status"
  },
  "call_id": "call_abc123",
  "tool_use_id": "call_abc123"
}
```

**All fields**:

| Field | Type | Description |
|-------|------|-------------|
| `session_id` | string | Current session identifier |
| `transcript_path` | string | Path to the session transcript JSONL file |
| `cwd` | string | Current working directory |
| `hook_event_name` | string | Event type — only `PreToolUse` is processed |
| `tool_name` | string | Tool name — only `Bash` and `execute_command` trigger rewrites |
| `tool_input` | object | Complete tool input; all fields are preserved in `updatedInput` |
| `call_id` | string | Tool call identifier |
| `tool_use_id` | string | Same as `call_id` |

### Output (stdout)

The hook writes a single JSON line to stdout.

**On rewrite** (token-optimized command available):
```json
{
  "continue": true,
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "RTK auto-rewrite",
    "updatedInput": {
      "command": "rtk git status",
      "timeout": 30000,
      "description": "Check repo status"
    }
  }
}
```

**No rewrite** (pass through unchanged):
```json
{"continue": true}
```

**Permission denied**:
```json
{
  "continue": false,
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Blocked by RTK permission rule"
  }
}
```

#### Top-level output fields

| Field | Type | Description |
|-------|------|-------------|
| `continue` | boolean | `false` blocks tool execution entirely |
| `hookSpecificOutput` | object | Structured output (see below) |
| `stopReason` | string | Reason shown when execution is stopped |
| `reason` | string | Alias for `stopReason` |
| `systemMessage` | string | Message appended to the system context |
| `suppressOutput` | boolean | If `true`, hides hook output from the user |
| `decision` | `"block"` \| `"approve"` | **Deprecated** — use `hookSpecificOutput.permissionDecision` |

#### `hookSpecificOutput` fields

| Field | Type | Description |
|-------|------|-------------|
| `hookEventName` | string | Fixed value: `"PreToolUse"` |
| `permissionDecision` | `"allow"` \| `"deny"` \| `"ask"` | Access control decision |
| `permissionDecisionReason` | string | Reason shown in the permission dialog |
| `updatedInput` | object | Rewritten tool input. **Must be `"allow"` for this to take effect.** All original `tool_input` fields are preserved and `command` is replaced. |
| `additionalContext` | string | Extra context injected into the conversation |

> **⚠️ Field name pitfall**: The official CodeBuddy documentation states `modifiedInput` as the field name for command rewriting. The actual implementation reads `updatedInput`. Always use `updatedInput`.

## How It Works

```
CodeBuddy runs command (e.g., "git status")
  ↓
PreToolUse hook triggered
  ↓
`rtk hook codebuddy` reads JSON from stdin
  ↓
Extracts command from `tool_input.command`
  ↓
Calls `rtk rewrite "git status"`
  ↓
Registry matches pattern, returns "rtk git status"
  ↓
Returns `updatedInput` with rewritten command (requires permissionDecision: "allow")
  ↓
CodeBuddy executes "rtk git status"
  ↓
Filtered output reaches LLM (~80% fewer tokens)
```

## Debugging

Set `RTK_DEBUG=1` to enable verbose stderr output (prefixed with `[rtk/hook/codebuddy]`):

```bash
export RTK_DEBUG=1
```

Example output:
```
[rtk/hook/codebuddy] Received input (first 500 chars): {...}
[rtk/hook/codebuddy] Event: PreToolUse
[rtk/hook/codebuddy] Tool: Bash
[rtk/hook/codebuddy] Original command: git status
[rtk/hook/codebuddy] Permission verdict: Allow
[rtk/hook/codebuddy] Rewritten command: rtk git status
[rtk/hook/codebuddy] Final decision: allow, updatedInput: {...}
```

CodeBuddy also logs hook activity to `~/.codebuddy/debug/latest` (symlink to the most recent session log). Look for:
```
[DEBUG] tool-permission: PreToolUse hook returned permissionDecision=allow for Bash
```

## Configuration File Locations

| Scope | Config File | Instructions File |
|-------|-------------|-------------------|
| Global | `~/.codebuddy/settings.json` | `~/.codebuddy/CODEBUDDY_INSTRUCTIONS.md` |
| Project-local | `<project>/.codebuddy/settings.json` | `<project>/.codebuddy/CODEBUDDY_INSTRUCTIONS.md` |

## Testing

```bash
# Test the hook manually
echo '{"hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"git status"}}' | rtk hook codebuddy

# Expected output:
# {"continue":true,"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"allow","permissionDecisionReason":"RTK auto-rewrite","updatedInput":{"command":"rtk git status"}}}
```

## Uninstallation

```bash
rtk init --codebuddy --uninstall
```

This removes:
- Hook entry from `settings.json`
- `CODEBUDDY_INSTRUCTIONS.md` file
