# Unsloth Desktop MCP image integration

Fusion viewport capture returns real MCP `ImageContent`, but Unsloth Desktop
0.1.800-beta originally stripped its internal `__MCP_IMAGES__` envelope before
the next model generation. That made screenshots visible in the UI but invisible
to the local vision model.

The tested local integration changes these installed backend files:

- `studio/backend/core/inference/mcp_client.py`
- `studio/backend/core/inference/tool_loop_controller.py`

`mcp_client._flatten_result` retains MCP image blocks as `{data, mimeType}` and
preserves all text metadata. Before an executed MCP result is appended to the
conversation, `tool_loop_controller` validates and splits that envelope and
converts each image to the same OpenAI-compatible content part used by normal
image uploads:

```python
{
    "type": "image_url",
    "image_url": {
        "url": f"data:{image['mimeType']};base64,{image['data']}",
    },
}
```

The resulting `role="tool"` message contains an ordered text part followed by
one or more image parts. Text-only MCP results remain strings. Existing
text-only model handling therefore remains unchanged, while vision models
receive the screenshot in the next inference request.

The Fusion MCP server should be registered as a local stdio command:

```text
"C:\Users\eamon\.unsloth\studio\unsloth_studio\Scripts\python.exe" -m fusion360_mcp
```

On a standard pip installation, replace the executable with the Python runtime
where this package is installed.

## Verified path

```text
Fusion activeViewport.saveAsImageFile
  -> temporary PNG bytes
  -> base64 in the existing newline-delimited TCP response
  -> MCP ImageContent(image/png)
  -> Unsloth multimodal image_url data URL
  -> local llama.cpp vision request
```

The integration was verified with a local Qwen3.8 vision model and an mmproj.
A box was captured, then a cylinder was added with a deliberate 2 cm gap. The
model identified the box and detected the floating cylinder in the canonical
front view; Fusion `measure_distance` independently confirmed the 2 cm gap.

