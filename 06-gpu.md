# 6. GPU acceleration

NVIDIA never published the Tegra 2 GPU's instruction set. The **Grate** project
(https://gitlab.com/grate-driver) reverse engineered it and wrote a Mesa (Gallium) driver. Mainline
Mesa has no Tegra 2 3D driver.

On mainline Linux 7.2 the Grate driver needed a lot of work before a desktop ran. That work was
done for this guide's machine: a new fragment shader compiler, blending, depth/stencil/scissor,
textures of all formats, batching of draw calls, buffer sharing with the compositor. **Those Mesa
changes are not published yet.** Grate's own branches give you a driver that initialises, but
draws little beyond the simplest shaders.

## What it takes on the kernel side

- `CONFIG_DRM_TEGRA_STAGING=y`: the driver uses the old job interface.
- `CONFIG_TEGRA_HOST1X_FIREWALL` **off**: with it on, the kernel rejects the 3D jobs.
- The `drm/tegra: skip CPU cache sync for write-combined buffers` commit
  ([03-kernel.md](03-kernel.md#patches)): without it most of the time goes into cache cleaning.
- `cma=192M` on the command line.

## Building Mesa for it

- Build for armv7 without NEON, and with **`-Dc_args=-mtls-dialect=gnu -Dcpp_args=-mtls-dialect=gnu`**.
  Mesa's configure test for TLS descriptors passes under qemu, but the resulting library hangs in
  its first GL call on the device. Alpine's own Mesa package forces the same flag.
- Install it to a prefix of its own (for example `/opt/grate`) and start programs with
  `LD_LIBRARY_PATH=/opt/grate/lib`, so the distribution's Mesa stays untouched.

## What you get

| | |
|---|---|
| API | OpenGL ES 2.0, OpenGL 2.0 |
| Weston 14, GL renderer | dragging a window at 59–60 frames/s |
| sway | runs |
| Textures | all GLES 2 formats; sizes that are not powers of two are clamped and have no mipmaps (hardware) |
| Depth | 16 bits |
| Not available | loops that Mesa cannot unroll, `dFdx`/`dFdy`, cube maps (no hardware mode found), point size per vertex |

Hardware limits worth knowing: 16 registers per pixel, 32 uniform scalars per fragment program,
one uniform and one attribute per vertex instruction. Shaders written for modern GPUs often do not
fit: GTK 4 wants OpenGL ES 3 and falls back to the CPU; WebKit's rounded-corner clip shader needs
280 uniform scalars.

## Speed

| Measurement | Result |
|---|---|
| Textured pixels | about 190 million pixel-steps a second; a textured pixel is 1 step, per-pixel lighting 12–18 |
| Draw calls | 256 per frame at 67 frames/s after batching (was 22) |
| Triangles | 4–8 million a second; Luanti's 123 000 triangles a frame give ~10 frames/s |

## Hardware facts found on the way

Measured on the machine and written down because nobody had:

- `PSEQ_CTL` (register 0x501) must be written with the right pixel limits, or programs that span
  several passes draw nothing.
- Temporaries must live in per-pixel register rows; the "global" registers are shared by all pixels.
- The ALU has one packet of read latency; dependent packets need a no-op between them.
- Blending is done by the fragment program, reading the destination pixel through the sequencer.
- Row pitch of textures and render targets must be a multiple of 64 bytes.
- Don't create scanout buffers bottom-up: Mesa already flips window framebuffers, and the display
  controller then flips again.

The full measured fragment instruction set, with the experiments behind each fact, exists as
working notes from this bring-up and can be published with the driver changes.
