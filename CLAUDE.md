# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
# Configure (first time, fetches all dependencies)
cmake -S . -B build

# Build
cmake --build build

# Build Release
cmake --build build --config Release

# Run
./build/vk_minimal_latest

# Enable semaphore debug logging
cmake -S . -B build -DVK_SEMAPHORE_DEBUG=ON

# Force GLSL shaders instead of Slang
cmake -S . -B build -DUSE_SLANG=OFF
```

All dependencies (GLFW, GLM, Volk, VMA, ImGui, Slang) are fetched automatically via `FetchContent` — only the Vulkan SDK must be installed separately with `VULKAN_SDK` set.

Shaders are compiled at build time by CMake custom commands (via `cmake/CompilerSlangShader.cmake` and `cmake/CompilerGlslShader.cmake`) and output as embedded C++ headers to `_autogen/`. Both Slang and GLSL shader sets are always compiled; the `USE_SLANG` flag only selects which set the C++ code includes.

## Architecture

### Two-file design
- **`src/vk_framework.h`** — All reusable Vulkan helpers (read this first). Defines `utils::Buffer`, `utils::Image`, `utils::ImageResource`, `Context`, `Swapchain`, `RenderTarget`, `DescriptorHeap`, `FramePacer`, and associated utilities. Contains the `VK_CHECK` and `ASSERT` macros.
- **`src/minimal_latest.cpp`** — The application (`VkMinimalApp`). Owns all per-app Vulkan objects and the render loop. Also defines `VMA_IMPLEMENTATION` (exactly once; `vk_framework.h` brings in VMA types only).

### Shader language
**Slang is canonical.** `shaders/shader.rast.slang` and `shaders/shader.comp.slang` are the reference implementations. The GLSL files (`shaders/shader.vert.glsl`, `shader.frag.glsl`, `shader.comp.glsl`) are side-by-side equivalents for comparison. `shaders/shader_io.h` is the shared host/device struct header (included by both C++ and shader code); it guards Slang-specific type aliases behind `#ifdef __SLANG__`.

### Key Vulkan patterns used

**No traditional pipeline objects for graphics.** The app uses `VK_EXT_shader_object` (`VkShaderEXT`) with per-frame `vkCmdSet*EXT` dynamic state calls instead of `VkPipeline`. All shader objects are created with `VK_SHADER_CREATE_DESCRIPTOR_HEAP_BIT_EXT`, so `pipelineLayout` is `VK_NULL_HANDLE`. Compute intentionally stays on the traditional `VkPipeline` path to show both styles side by side.

**Bindless resources via `VK_EXT_descriptor_heap`.** Textures and samplers live in GPU heap buffers. Shaders access them by index (`Texture2D.Handle(uint2(index, 0))` in Slang). ImGui still uses a small legacy descriptor pool because its backend predates the heap.

**Buffer device address (BDA) throughout.** `SceneInfo` and vertex/point buffers are passed to shaders as raw 64-bit addresses (stored in `utils::Buffer::address`) rather than through descriptor sets. `vkCmdUpdateBuffer` writes the per-frame `SceneInfo` struct directly.

**`VK_KHR_unified_image_layouts`.** All attachments (color, depth, swapchain) use `VK_IMAGE_LAYOUT_GENERAL`, eliminating layout-transition barriers except for the final swapchain present transition.

### Frame synchronization model

Two independent counts govern frame resources:
- **Swapchain images** (default 3) — sized by `vkGetSwapchainImagesKHR`; own `presentSemaphore` (binary, per-image).
- **Frames in flight** (default 2) — application choice; each slot owns a command pool, command buffer, `acquireSemaphore` (binary), and `lastSignalValue` (timeline).

A single **monotonic timeline semaphore** (`m_frameTimelineSemaphore`, counter `m_frameCounter`) paces CPU–GPU. Each slot waits on its `lastSignalValue` before reuse, then signals `m_frameCounter++`. The slot index cycles `[0, framesInFlight)` independently of the swapchain image index returned by `vkAcquireNextImageKHR`.

See `doc/swapchain_restaurant.md` for a narrative walkthrough of this model.

### Shader specialization
The rasterization fragment shader has a `useTexture` specialization constant (constant_id 0). CMake compiles two `VkShaderEXT` variants at startup; the app swaps between them per draw call (`vkCmdBindShadersEXT`).
