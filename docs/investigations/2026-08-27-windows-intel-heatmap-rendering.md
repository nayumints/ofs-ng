# Fix Windows Intel Heatmap Rendering by Configuring Custom Shader Vertex Attributes

Investigation date: 2026-08-27

## Summary

The seek-bar/editor heatmap was invisible on Windows systems using Intel Graphics, even though exporting the same heatmap to PNG produced the expected output.

The on-screen draw callback reused Dear ImGui's vertex array object (VAO) without configuring it for the heatmap shader's explicit vertex attribute locations. Explicitly configuring those attributes before drawing fixes the issue.

## Environment

- Windows 11
- Intel(R) Graphics
- Driver 32.0.101.7088
- OpenGL 3.3
- OFS-NG revision `816c57d8bfa5759b6225b806f25e702ea2273f1c`
- Reported version `v0.2.2-dirty`
- Dear ImGui pinned to a 1.92.9b-era source/backend
- Display scaling at 100%
- Hardware video decoding disabled during diagnosis

The issue reproduced with unrelated videos and funscripts. It persisted after:

- Resetting `%APPDATA%\ofs\ofs-ng`
- Testing OFS-NG v0.2.1 and v0.2.2
- Restarting Windows
- Updating the Intel graphics driver
- Disabling hardware video decoding
- Removing the MuMu virtual display adapter
- Setting display scaling to 100%

## Symptoms

- The on-screen seek-bar heatmap was invisible.
- PNG heatmap export had the correct data and colors.
- No shader compilation or program-linking errors occurred.
- The log only reported `OFS: Loaded OpenGL 3.3`.

The working PNG export established that heatmap speed calculation, texture and color-LUT uploads, shader execution, and off-screen framebuffer rendering were all correct.

## Root Cause

`Heatmap::draw()` uses an ImGui draw callback to activate the custom heatmap shader, then relies on the existing ImGui VAO and vertex buffer object (VBO) to draw the image quad.

The heatmap vertex shader declares explicit attribute locations:

```glsl
layout (location = 0) in vec2 Position;
layout (location = 1) in vec2 UV;
layout (location = 2) in vec4 Color;
```

The ImGui OpenGL backend configures its VAO using locations obtained from ImGui's own linked shader. Attribute locations are program-specific and driver-dependent, so they are not guaranteed to match the heatmap shader's locations `0`, `1`, and `2`.

On the affected Intel Windows driver, the reused ImGui VAO did not provide valid vertex data at the locations expected by the heatmap shader. The draw completed without an OpenGL error but produced no visible fragments.

`Heatmap::renderToBitmap()` was unaffected because it creates its own VAO and explicitly configures locations `0`, `1`, and `2`.

## Diagnostic Evidence

A focused diagnostic build forced the on-screen fragment shader to output opaque magenta. Before configuring the attribute pointers, no magenta appeared, while the logged GL state showed that the rest of the draw path was valid:

```text
[DEBUG-heatmap-gl] before draw program=9/9 fbo=0 vao=4 viewport=0,0,1920,1032 scissor=19,29,1882,82 rect=19,921,1901,1003
[DEBUG-heatmap-gl] after draw error=0x0 program=9/9 active=0 tex0=2/2 sampler0=1 tex2=1/1 sampler2=0 blend=true
```

This confirmed that the custom shader, framebuffer, viewport, scissor rectangle, textures, samplers, and blending state were correct, and that the draw produced no GL error.

After explicitly configuring vertex attributes `0`, `1`, and `2` in the callback, the magenta quad became visible. Restoring the normal fragment shader then produced the expected colored heatmap.

## Fix

Before drawing the heatmap quad, configure the active ImGui VAO and VBO for the heatmap shader:

```cpp
glEnableVertexAttribArray(0);
glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, sizeof(ImDrawVert),
                      reinterpret_cast<void*>(offsetof(ImDrawVert, pos)));

glEnableVertexAttribArray(1);
glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, sizeof(ImDrawVert),
                      reinterpret_cast<void*>(offsetof(ImDrawVert, uv)));

glEnableVertexAttribArray(2);
glVertexAttribPointer(2, 4, GL_UNSIGNED_BYTE, GL_TRUE, sizeof(ImDrawVert),
                      reinterpret_cast<void*>(offsetof(ImDrawVert, col)));
```

The existing `DrawCallback_ResetRenderState` command remains after the heatmap quad, allowing the ImGui OpenGL backend to restore its shader and vertex layout before subsequent UI rendering.

The patch also replaces the invalid image texture reference:

```cpp
drawList->AddImage(0, min, max);
```

with the actual speed texture:

```cpp
drawList->AddImage(
    ImTextureRef(static_cast<ImTextureID>(m_speedTexture)),
    min,
    max);
```

The shader now samples the backend-bound speed texture from texture unit 0. This matches the current ImGui `ImTextureRef` contract, in which texture ID `0` is `ImTextureID_Invalid`.

The `ImTextureRef` correction alone did not resolve the rendering failure. Configuring the vertex attributes was the causal fix.

## Scope

Changed:

- `src/UI/Heatmap.cpp`

Unchanged:

- Funscript processing
- Heatmap speed calculation
- PNG export logic
- Preferences
- Plugins
- Video decoding
- Vendored ImGui backend

## Validation

Confirmed on the affected Windows 11 Intel Graphics system:

- The normal colored on-screen heatmap is visible.
- PNG heatmap export still works.
- Other ImGui UI remains intact.
- No OpenGL errors are produced.
- Temporary magenta output and diagnostic logging were removed.
- Focused C++ syntax compilation passed.
- `git diff --check` passed.

Build command:

```powershell
cmake --build build-win --config Release --target ofs-ng -j 8
```

Tested executable:

```text
.\bin\Release\ofs-ng.exe
```

## Why This Fix Is Minimal

The patch retains both existing rendering paths and only corrects the GL vertex-input state required by the on-screen custom shader. PNG export remains unchanged because it already configures its own VAO correctly.
