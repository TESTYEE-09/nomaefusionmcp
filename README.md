# Fusion 360 MCP with viewport vision

Control Autodesk Fusion 360 from an MCP agent, capture the real viewport through
Fusion's API, and return PNG images to a local vision model.

```text
Vision model ↔ MCP stdio server ↔ TCP :9876 ↔ Fusion add-in ↔ Fusion API
                                              └─ PNG bytes → MCP ImageContent
```

The default compact profile is designed for Qwen3.8-27B Q3 and other
small-context models. All 93 Fusion tools remain available without placing all
93 schemas in every prompt.

## Context reduction

| Profile | Resident tools | Qwen prompt | 5,632-token context |
|---|---:|---:|---|
| `compact` (default) | 12 including gateways | Small enough for 5–12k contexts | Works |
| `full` | 93 | Large | Use with large-context models |

Compact mode reduced the tested prompt by about 90%. It keeps common inspection
and viewport tools resident and loads uncommon schemas only when requested.

## Install

Requirements: Autodesk Fusion 360, Python 3.10+, and an MCP client. `uv` is
recommended.

### 1. Install the Fusion add-in

Windows PowerShell:

```powershell
Copy-Item -Recurse addon "$env:APPDATA\Autodesk\Autodesk Fusion 360\API\AddIns\Fusion360MCP"
```

macOS:

```bash
cp -r addon ~/Library/Application\ Support/Autodesk/Autodesk\ Fusion\ 360/API/AddIns/Fusion360MCP
```

In Fusion, open **Shift+S → Add-Ins → Fusion360MCP → Run**. The add-in listens
on `localhost:9876` by default.

### 2. Register the MCP server

Claude Code:

```bash
claude mcp add fusion360 -- uvx fusion360-mcp-server --mode socket
```

Generic MCP configuration:

```json
{
  "mcpServers": {
    "fusion360": {
      "command": "uvx",
      "args": ["fusion360-mcp-server", "--mode", "socket"]
    }
  }
}
```

For a local checkout:

```bash
python -m pip install -e .
python -m fusion360_mcp
```

### Unsloth Desktop

Register the server using Unsloth's Python environment, for example:

```text
"C:\Users\eamon\.unsloth\studio\unsloth_studio\Scripts\python.exe" -m fusion360_mcp
```

Unsloth Desktop 0.1.800-beta required a backend patch so MCP images reach the
next multimodal inference request instead of only the UI. See
[UNSLOTH_INTEGRATION.md](UNSLOTH_INTEGRATION.md) and
[`patches/unsloth-mcp-image-forwarding.patch`](patches/unsloth-mcp-image-forwarding.patch).

## Compact tool workflow

Ten Fusion tools plus two compact gateways stay in context:

- `ping`
- `get_scene_info`
- `get_object_info`
- `get_bounding_box`
- `measure_distance`
- `check_interference`
- `capture_viewport`
- `capture_model_views`
- `create_annular_ring`
- `create_print_in_place_gyro`
- `find_fusion_tools`
- `run_fusion_tool`

Use the gateway for uncommon operations:

```text
find_fusion_tools({"query":"parametric box"})
run_fusion_tool({"tool":"create_box_parametric","arguments":{...}})
capture_viewport({"view":"isometric"})
```

### Reliable long CAD tasks

Small local models often end a tool loop voluntarily even when the client limit
is set to Max. The solution is to reduce the number of decisions, not merely
raise the limit. Two high-level tools are therefore resident:

```text
create_annular_ring({
  "outer_diameter_mm":60,
  "inner_diameter_mm":52,
  "thickness_mm":10,
  "body_name":"Outer_Frame"
})

create_print_in_place_gyro({
  "overall_diameter_mm":60,
  "thickness_mm":10,
  "ring_wall_mm":4,
  "clearance_mm":0.6
})
```

The gyro command atomically creates `Outer_Frame`, `Middle_Gimbal`, and
`Central_Rotor` with the requested radial clearance. It uses direct annular
profiles, so it cannot fall into the unsupported guessed-cut workflow that
previously caused Qwen to abandon the task. Follow it with `capture_viewport`
and `check_interference`.

For a large-context model, expose every schema:

```bash
python -m fusion360_mcp --tool-profile full
```

Or set `FUSION_MCP_TOOL_PROFILE=full`.

## Viewport tools

`capture_viewport` returns text metadata and a protocol-native PNG image:

```json
{
  "view": "isometric",
  "width": 1280,
  "height": 1280,
  "fit_to_model": true
}
```

Views: `current`, `isometric`, `top`, `front`, `right`, `left`, `back`, and
`bottom`. Dimensions must be 64–4096 pixels.

`capture_model_views` returns isometric, top, front, and right images. The older
`render_view` tool remains available in the full profile for compatibility.

PNG bytes originate inside Fusion, cross the existing TCP connection as base64,
and become MCP `ImageContent`. No shared filesystem is required, so trusted-LAN
setups work too. Temporary PNG files are removed after encoding.

## CAD decision loop

The MCP instructions tell the agent to prefer atomic high-level operations and:

1. Act instead of returning a plan or progress report.
2. Preserve completed geometry after a failure.
3. Capture the viewport after a meaningful stage.
4. Decide from the image once, then continue or repair.
5. Verify with bounding boxes, distances, and interference checks.

Images are for placement, orientation, disconnected shapes, and visual sanity.
Fusion measurements remain authoritative for dimensions, clearance, and
collisions. If the model repeatedly reconsiders the same geometry, it should
stop reasoning and inspect the viewport.

## Tool coverage

The full catalog covers:

- Sketches, constraints, dimensions, offsets, trim, extend, and projection
- Extrude, revolve, sweep, loft, fillet, chamfer, shell, holes, and patterns
- Direct primitives, booleans, transforms, parameters, and appearances
- Components, joints, rigid groups, construction geometry, and interference
- Surface, sheet-metal, CAM, import, and STL/STEP/F3D export
- Scene queries, physical properties, exact measurements, and code execution

Use `find_fusion_tools` for an exact name and schema instead of guessing.

## LAN configuration

On the Fusion host:

```powershell
$env:FUSION_MCP_HOST = "0.0.0.0"
```

On the MCP host:

```bash
python -m fusion360_mcp --host 192.168.1.42
```

The TCP socket has no authentication. Use only a trusted LAN and never expose
port 9876 to the public internet.

## Verify and develop

Call `ping`, then `capture_viewport`. A successful viewport result contains
text metadata followed by an `image/png` MCP block.

```bash
uv sync --dev
uv run pytest -q       # 308 tests
uv run ruff check src tests
```

Tested end to end with Qwen3.8-27B Q3 plus an mmproj. The model identified
created geometry and detected a deliberately floating cylinder in a front view;
Fusion independently measured the 2 cm gap.

## Troubleshooting

- **Not connected:** start Fusion and run the Fusion360MCP add-in.
- **Context overflow:** restart the MCP client and use the default compact profile.
- **Image only appears in Unsloth UI:** apply the supplied Unsloth integration patch.
- **Timeout:** close Fusion modal dialogs and retry after `get_scene_info`.
- **Wrong spatial conclusion:** capture front/top/right views and verify exactly.

Low-level Fusion API tools use centimeters. The high-level ring and gyro tools
accept millimeters explicitly. Prefer one low-level geometry operation per call;
the atomic high-level builders safely perform their documented compound task.
Add-in logs are written to `~/fusion360mcp.log`.

## License

MIT
