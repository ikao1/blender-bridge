# blender-bridge

Run Python inside a live Blender session from an MCP client such as
[opencode](https://opencode.ai). `blender-bridge` has two halves:

- **`blender_bridge.py`** — a small TCP server that runs *inside* Blender,
  executes requests on Blender's main thread (via `bpy.app.timers` in the GUI),
  and returns stdout, the value of the last expression, and any traceback. It is
  auto-started by a startup hook.
- **`mcp_server.py`** — a dependency-free MCP stdio server that forwards tool
  calls to the in-Blender bridge and exposes them as the `blender_run` and
  `blender_status` tools.

```
MCP client  <--stdin/stdout-->  mcp_server.py  <--TCP 127.0.0.1:7788-->  Blender (blender_bridge.py)
```

## Requirements

- Blender (developed against 5.2 LTS; works on 3.x/4.x since it only uses
  `bpy.app.timers` and the standard library).
- Python 3 for the MCP server — standard library only, no `pip install` needed.

## Install

### 1. Blender side

Copy `blender_bridge.py` into Blender's user `modules` directory and the startup
hook into `startup`:

```
~/Library/Application Support/Blender/<version>/scripts/modules/blender_bridge.py
~/Library/Application Support/Blender/<version>/scripts/startup/opencode_blender_bridge.py
```

The included script does this for you (macOS, default version 5.2):

```sh
./install.sh 5.2
```

Then restart Blender. The bridge starts automatically and prints to the system
console:

```
blender-bridge listening on 127.0.0.1:7788
```

To start it manually instead, run this in the **Scripting** workspace:

```python
import blender_bridge
blender_bridge.start()
```

### 2. Headless / `--background`

`blender --background` has no UI event loop for the main-thread timer to use, so
drive the blocking pump instead (this keeps the process alive):

```sh
blender --background --python-expr "import blender_bridge; blender_bridge.run()"
```

### 3. MCP client side

Add the server to your opencode config (`~/.config/opencode/opencode.jsonc`),
pointing at the absolute path of `mcp_server.py`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "blender": {
      "type": "local",
      "command": ["python3", "/ABSOLUTE/PATH/TO/blender-bridge/mcp_server.py"],
      "environment": { "BLENDER_BRIDGE_PORT": "7788" },
      "timeout": 300000,
      "enabled": true
    }
  },
  "experimental": { "mcp_timeout": 300000 }
}
```

See `opencode.jsonc.example`. Any MCP client that speaks the `initialize`,
`tools/list`, and `tools/call` methods works.

## Tools

### `blender_status`

Confirms the bridge is reachable and reports Blender version, platform, current
`.blend` file, scene, object count, mode, and the current selection.

### `blender_run`

| argument          | type    | notes                                                    |
| ----------------- | ------- | -------------------------------------------------------- |
| `code`            | string  | Python source to execute (required).                     |
| `timeout_seconds` | number  | Default 120, max 290. Blender keeps running after timeout.|

The script runs on Blender's **main thread**, so the UI blocks while it runs.
Python globals persist between calls (a persistent REPL session) and `bpy`,
`bmesh`, and `mathutils` are pre-imported. The value of the last expression is
returned as `result`.

## Configuration

| variable               | default | used by     |
| ---------------------- | ------- | ----------- |
| `BLENDER_BRIDGE_PORT`  | `7788`  | both halves |

Set the same port on both sides if you change it.

## Security

The bridge listens on `127.0.0.1` only and executes arbitrary code on your
machine by design. Do not expose the port to a network — treat it like a local
shell. Anyone who can reach the port can run anything as your user.

## License

MIT — see [LICENSE](LICENSE).
