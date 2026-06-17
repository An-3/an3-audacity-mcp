# Install protocol for AI agents

You are an AI coding agent (Claude Code, Codex, Cursor, or similar) installing the
**Audacity MCP server** for your user. Follow these steps in order. Stop and ask the
user only where this document explicitly says to.

The server controls a running Audacity instance over `mod-script-pipe` and exposes
Audacity's scripting commands as MCP tools. It uses `stdio` transport, so each MCP
client launches the server itself — there is no long-running daemon to manage.

## 0. Preconditions

- macOS or Linux (named pipes live in `/tmp`).
- Python 3.10+ available as `python3`.
- Audacity 3.x+ installed.
- `git` available.

If Python 3.10+ is missing, tell the user and stop.

## 1. Clone and set up

```bash
git clone https://github.com/An-3/an3-audacity-mcp.git
cd an3-audacity-mcp
./scripts/setup.sh
```

`setup.sh` creates `.venv` and installs `mcp[cli]` and `httpx`. Record the absolute
path to `.venv/bin/python` and to `audacity_mcp_server.py` — you need both for the
client config. Compute them now:

```bash
echo "$(pwd)/.venv/bin/python"
echo "$(pwd)/audacity_mcp_server.py"
```

## 2. Have the user enable mod-script-pipe

This step needs the GUI, so the user must do it. Tell them:

1. Open Audacity.
2. `Edit → Preferences → Modules` (macOS: `Audacity → Settings… → Modules`).
3. Set `mod-script-pipe` to **Enabled**.
4. **Restart Audacity** and open or create a project.

The pipes (`/tmp/audacity_script_pipe.to.*` and `.from.*`) only appear after the
restart. Wait for the user to confirm before continuing.

## 3. Verify connectivity

```bash
./scripts/smoke_test.sh
```

If it reports the pipes are missing, mod-script-pipe is not enabled or Audacity was
not restarted — send the user back to step 2. Do not proceed until the smoke test
talks to Audacity.

## 4. Register the MCP server with the user's agent

Use the absolute paths from step 1.

**Claude Code:**

```bash
claude mcp add audacity -- /ABS/PATH/.venv/bin/python /ABS/PATH/audacity_mcp_server.py
```

**Codex:**

```bash
codex mcp add audacity -- /ABS/PATH/.venv/bin/python /ABS/PATH/audacity_mcp_server.py
```

**Claude Desktop / other stdio clients** — add to the client's MCP config:

```json
{
  "mcpServers": {
    "audacity": {
      "command": "/ABS/PATH/.venv/bin/python",
      "args": ["/ABS/PATH/audacity_mcp_server.py"]
    }
  }
}
```

Replace `/ABS/PATH` with the real absolute path. Relative paths and `uv`/`python`
shorthands are the most common cause of `spawn ENOENT` — always use absolute paths.

## 5. Install the agent skill

The repo ships an agent skill at
`.claude/skills/audacity-control/SKILL.md` documenting the tool-to-command mapping,
the no-argument limitation, recommended workflows, and troubleshooting.

- **Claude Code:** if the user opens this repo as a project, the skill is picked up
  automatically. To make it available everywhere, copy it into the user's skills
  directory:

  ```bash
  mkdir -p ~/.claude/skills/audacity-control
  cp .claude/skills/audacity-control/SKILL.md ~/.claude/skills/audacity-control/
  ```

- **Other agents:** read `.claude/skills/audacity-control/SKILL.md` into your context
  before driving Audacity; it tells you how the tools behave.

## 6. Verify end to end

Restart the user's agent so it loads the new MCP server, then confirm the `audacity`
tools (`cmd_*`) are listed. As a live check, call `cmd_GetInfo` and confirm Audacity
returns project information ending in `BatchCommand finished`.

Report success with: the install path, which client(s) you registered, and the
result of the `cmd_GetInfo` check. If any step failed, report which one and the
exact error.
