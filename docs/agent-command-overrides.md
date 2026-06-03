# Agent Availability Commands

## Problem

Orca can miss CLIs that work in a user's terminal because detection runs from a different process environment. This is common on WSL when tools are installed through `nvm`, `asdf`, or similar shell initializers: an interactive WSL terminal can run `codex`, while a non-interactive probe such as `bash -lc 'command -v codex'` cannot see the same PATH.

The wrong fix is to make a detection path automatically become the terminal launch command. If a user enters `/home/axdr/.nvm/versions/node/v20.19.5/bin/codex` only to prove Codex exists, Orca should not start typing that full path into every terminal when plain `codex` already works.

## Goal

Make local agent availability runtime-aware without changing launch behavior by surprise:

1. Users can provide an availability command for an undetected local agent.
2. A valid availability command makes the agent appear in launch and default choices.
3. Availability commands are scoped to Windows/host, WSL default, or a named WSL distro.
4. WSL probes use an interactive shell so `.bashrc`-initialized tools such as `nvm` are detected as plain commands.
5. Launch commands remain clean and backward-compatible: Orca launches the catalog command, or the existing legacy launch override if one is configured.

## Non-Goals

- No filesystem scanning for agent installs.
- No package-manager-specific discovery.
- No SSH-specific availability command editor in this pass.
- No new launch override UX. Existing flat `agentCmdOverrides` launch behavior is preserved.

## Design

### Runtime-Scoped Availability

Add `agentCmdOverridesByRuntime?: Record<string, Partial<Record<TuiAgent, string>>>` to settings.

Runtime keys:

- `host`
- `wsl:default`
- `wsl:<distro>`

The existing flat `agentCmdOverrides` remains the legacy launch override map. Runtime-scoped values are used for availability detection and UI provenance, not for terminal startup command construction.

### Detection

Renderer computes the active local agent runtime and passes the effective availability commands to preflight detection. Main preflight checks:

- catalog detect commands from `TUI_AGENT_CONFIG`
- user-provided availability command executable tokens

The result carries provenance:

```ts
{ id: TuiAgent, catalogFound: boolean, overrideFound: boolean }
```

The renderer still derives the id list for existing launch/default consumers, but the Settings UI can distinguish:

- `Detected`: catalog command was found
- `Custom check`: availability command was found
- `Not found`: neither was found

### WSL Shell Mode

WSL probes use interactive bash (`bash -ic`) for command availability. This matches the user's WSL terminal better than `bash -lc`, because `nvm` and similar tools are commonly initialized from `.bashrc`.

This applies to:

- agent preflight detection
- the Codex WSL account availability check that previously produced "Codex CLI is not available in WSL ..."

The actual `codex login` WSL spawn remains the existing explicit login command.

### Launch Separation

Runtime-scoped availability commands do not replace launch commands.

Example:

- Availability command for WSL default: `/home/axdr/.nvm/versions/node/v20.19.5/bin/codex`
- `+` menu label: `Codex`
- terminal startup command: `codex`

If the user has an existing legacy launch override such as `codex --profile work`, launch continues to use that legacy override.

## UI

Every agent row can open an availability command editor, including rows under "Available to install".

Copy:

- `Availability command for Windows`
- `Availability command for WSL default`
- `Availability command for WSL <distro>`

Help text:

> Used for detection only. Launches still run the agent command unless a launch override is configured.

The compact row keeps showing the catalog launch command (`codex`, `claude`, etc.). If an availability command is configured, it is shown separately as an availability check so the UI does not imply launch replacement.

## Data Flow

1. Settings opens Agents pane.
2. Renderer computes local preflight context from Account/Agent location.
3. Renderer resolves effective availability commands for that context.
4. Renderer calls `preflight.detectAgents({ ...context, agentCmdOverrides })`.
5. Main preflight checks catalog commands plus availability executable tokens.
6. Store receives provenance and derives detected ids.
7. Agents pane renders `Detected`, `Custom check`, or `Not found`.
8. Launch paths continue to use legacy flat `agentCmdOverrides` only.

## Test Plan

- Shared helper tests for runtime keys and scoped availability resolution.
- Preflight tests for catalog-only, custom-check-only, both-found, and invalid custom-check cases.
- WSL preflight tests asserting `bash -ic` is used.
- Codex WSL account test asserting the availability check uses `bash -ic`.
- Renderer store tests for runtime-scoped detection cache keys.
- Agents pane tests for custom-check UI states.
- Launch tests proving runtime-scoped availability commands do not replace terminal startup commands.
- Typecheck, lint, format, and Electron UI validation.
