# Python Toolset

An editor-only Unreal Engine plugin that adds a **Python escape-hatch toolset**
to the [Unreal MCP](https://dev.epicgames.com/documentation/unreal-engine/unreal-mcp-in-unreal-editor)
(Model Context Protocol) server. It lets an MCP client run arbitrary editor
Python to cover tasks that no dedicated toolset exposes, and ships an agent
skill describing when that fallback is appropriate.

## What it is

The Unreal MCP plugin exposes the editor to AI agents through a registry of
*toolsets* - curated, typed tools for assets, Blueprints, UMG, PCG, Niagara,
GAS, Sequencer, and so on. Those toolsets are deliberately narrow.

This plugin adds one more toolset, `PythonBridgeToolset`, whose single job is to
run a Python script inside the running editor with the **full `unreal` module
available and no sandbox**. It is the gap-filler for the cases the curated
toolsets do not reach.

It also registers an agent skill (`PythonBridgeSkill`) that tells the agent to
treat the bridge as a *fallback* - to prefer a dedicated toolset whenever one
fits, and only reach for raw Python when nothing else does.

## What it does

The toolset exposes two MCP tools:

- **`get_python_environment`** - returns the script contract (the required
  `run()` entrypoint, how return values and stdout are encoded, and the
  surrounding transaction behaviour). Call this once before the first script.
- **`execute_python`** - compiles and runs a submitted script.

### Script contract

A submitted script must define a `run()` function:

```python
import unreal

def run():
    actors = unreal.EditorActorSubsystem().get_all_level_actors()
    return {"actor_count": len(actors)}
```

- Whatever `run()` returns is JSON-encoded and sent back under the `result`
  key. Values that are not natively JSON-serializable are stringified, so
  prefer returning plain `dict` / `list` / `str` / `int` / `float` / `bool` /
  `None`.
- Anything printed to stdout is captured and returned under the `stdout` key.
- The whole execution is wrapped in a **single editor transaction**. On success
  the transaction is kept, so the script's changes undo as one step (Ctrl+Z).
  If `run()` raises, the transaction is rolled back so partial mutations do not
  persist, and a script-scoped traceback is returned as the tool error.

Scripts run **synchronously on the game thread** (where MCP tool calls are
dispatched), so they can call the `unreal` API directly without marshalling to
another thread.

### How it differs from the built-in `ProgrammaticToolset`

The engine's `EditorToolset` ships a `ProgrammaticToolset` that also executes
Python, but it is a sandbox: scripts may only import a small standard-library
allowlist and may only *orchestrate other registered tools*. This plugin is the
opposite by design - full, unrestricted `unreal` access - because its purpose is
to reach engine APIs that no registered tool wraps. Prefer `ProgrammaticToolset`
when you only need to batch existing tools; use this when you need an API that
isn't otherwise exposed.

## Requirements

- Unreal Engine with the **Model Context Protocol** and **Toolset Registry**
  plugins available (they ship with the engine under
  `Engine/Plugins/Experimental/`).
- The **Python Editor Script Plugin** (`PythonScriptPlugin`) enabled.

## How to enable it

1. **Copy the plugin** into your project's `Plugins/` directory, so the layout
   is `Plugins/PythonToolset/PythonToolset.uplugin`.

2. **Enable the plugin** either from the editor (Edit -> Plugins -> search
   "Python Toolset" -> tick Enabled) or by adding it to your `.uproject`:

   ```json
   {
       "Name": "PythonToolset",
       "Enabled": true
   }
   ```

   Make sure `ModelContextProtocol` and `PythonScriptPlugin` are enabled too;
   the plugin depends on the Toolset Registry, which the MCP server reads from.

3. **Restart the editor.** Python toolsets register at startup, and a *new* MCP
   tool only appears after a full restart - the `ModelContextProtocol.RefreshTools`
   console command re-polls the registry but does not re-run plugin startup
   scripts.

4. **Verify.** Once the editor is running with the MCP server up, the toolset
   appears in the client's tool list as:

   - `python_toolset.toolsets.python_bridge.PythonBridgeToolset.get_python_environment`
   - `python_toolset.toolsets.python_bridge.PythonBridgeToolset.execute_python`

   and the skill `PythonBridgeSkill` appears under the Agent Skill toolset's
   skill list.

## Layout

```
PythonToolset/
├── PythonToolset.uplugin          Editor-only; depends on ToolsetRegistry + PythonScriptPlugin
├── README.md
└── Content/Python/
    ├── init_unreal.py             Runs at startup: imports skills, registers toolsets
    └── python_toolset/
        ├── toolsets/
        │   └── python_bridge.py   PythonBridgeToolset (get_python_environment, execute_python)
        └── skills/
            └── python_bridge.py   PythonBridgeSkill (fallback guidance)
```

The plugin is content-only (no C++ module), so it requires no compilation - just
enable it and restart.

## Security note

`execute_python` runs **unrestricted** Python with full editor and filesystem
access. It is as powerful as anything you can type into the editor's Python
console. Only expose the MCP server to trusted clients, and review scripts that
mutate the project before running them - the transaction rollback protects
against partially-applied *failures*, not against a script that succeeds at
doing something destructive.
