---
name: audacity-control
description: >
  Control Audacity from an agent through the Audacity MCP server (mod-script-pipe).
  Use when the user wants to record, edit, apply effects to, generate, analyze, or
  export audio in Audacity — e.g. "import this wav", "normalize the track",
  "fade out the end", "export as mp3", "add a label", "reduce noise". Covers the
  prerequisite setup, how MCP tools map to Audacity scripting commands, the
  no-argument limitation of the generated tools, recommended workflows, and
  troubleshooting.
---

# Audacity control via MCP

This server exposes Audacity's scripting commands (the same ones available through
`mod-script-pipe` / the Scripting reference) as MCP tools. Each tool is named
`cmd_<CommandId>` (e.g. `cmd_Normalize`, `cmd_ImportAudio`, `cmd_ExportAudio`) and
sends `<CommandId>:` over the named pipe to a running Audacity instance.

## Before anything works

1. **Audacity must be running** with a project open.
2. **`mod-script-pipe` must be enabled**: `Edit → Preferences → Modules` (macOS:
   `Audacity → Settings… → Modules`), set `mod-script-pipe` to **Enabled**, then
   **restart Audacity**. The named pipes only appear in `/tmp` after a restart.
3. The server auto-detects pipe suffixes in `/tmp`
   (`audacity_script_pipe.to.*` / `.from.*`), so the macOS UID suffix like `.501`
   is handled for you.

If the pipes are missing, every tool returns a connection error — fix setup first,
don't retry blindly.

## Key behavior to plan around

- **Tools take no arguments.** Every generated tool sends the bare command with no
  parameters, so commands run with Audacity's **default or last-used settings**.
  For example `cmd_Amplify` applies amplify with whatever values were last used in
  the dialog. There is currently no way to pass numeric parameters (amount, dB,
  frequency, file path) through these tools.
- **Effects act on the current selection.** Audacity applies most effects and edits
  to the selected region of the focused track. If nothing is selected, the command
  may do nothing or error. Use `cmd_SelectAll` to operate on the whole track before
  applying an effect.
- **One-shot, fire-and-forget.** Each call returns Audacity's text response up to
  `BatchCommand finished`. Inspect that string to confirm success.

## Common workflows

**Apply an effect to a whole track**
1. `cmd_SelectAll`
2. effect tool, e.g. `cmd_Normalize`, `cmd_NoiseReduction`, `cmd_FadeOut`,
   `cmd_Amplify`

**Import → process → export**
1. `cmd_ImportAudio` (imports a file as a new track — uses Audacity's import dialog
   defaults)
2. `cmd_SelectAll`
3. effect tool(s)
4. `cmd_ExportAudio` (exports using last-used format/path)

**Record**
- `cmd_Record1stChoice` (current track) or `cmd_Record2ndChoice` (new track),
  `cmd_Pause`, `cmd_PlayStop`.

**Generate audio**
- `cmd_Tone`, `cmd_Noise`, `cmd_Chirp`, `cmd_Pluck`, `cmd_RhythmTrack` —
  again with default/last-used parameters.

**Inspect project state**
- `cmd_GetInfo` returns project information (tracks, clips, labels, commands) as
  text — useful to verify what happened after an edit.

## Tool categories

File, Import, Edit, Label, Select, View, Transport, Effect, Generate, Analyze,
Tools, Device, Selection, Timeline, Focus, Cursor, Track, Scriptables I/II, Help,
Diagnostics. Each Audacity command id in those categories is registered as a
`cmd_<id>` tool. When unsure of an id, `cmd_Help` and `cmd_GetInfo` report what the
running Audacity build supports.

## Troubleshooting

- `Could not connect to Audacity mod-script-pipe` → Audacity not running, or
  `mod-script-pipe` not enabled / not restarted, or pipes absent from `/tmp`.
- Effect "succeeds" but nothing audible changes → nothing was selected, or the
  effect used stale dialog defaults. Select first; verify with `cmd_GetInfo`.
- Need parameterized control (exact dB, exact export path/format) → not supported by
  the current no-argument tools; do that step manually in Audacity, or extend the
  server to forward parameters in the `Command: key=value` form.
