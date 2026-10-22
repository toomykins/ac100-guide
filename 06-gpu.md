# 6. GPU acceleration

NVIDIA never published the Tegra 2 GPU's instruction set. The **Grate** project
(https://gitlab.com/grate-driver) reverse engineered it and wrote a Mesa (Gallium) driver. Mainline
Mesa has no Tegra 2 3D driver.

On mainline Linux 7.2 the Grate driver needed a lot of work before a desktop ran: a new fragment
shader compiler, blending, depth/stencil/scissor, textures of all formats, batching of draw calls,
buffer sharing with the compositor. That work is branch
[`ac100-grate` on github.com/toomykins/mesa](https://github.com/toomykins/mesa/tree/ac100-grate):
16 commits on top of Grate's `grate-25.1.9` branch (commit `647c7941`, included as a single base
commit without its history). Grate's own branches give you a driver that initialises but draws
little beyond the simplest shaders.

## What it takes on the kernel side

- `CONFIG_DRM_TEGRA_STAGING=y`: the driver uses the old job interface.
- `CONFIG_TEGRA_HOST1X_FIREWALL` **off**: with it on, the kernel rejects the 3D jobs.
- The `drm/tegra: skip CPU cache sync for write-combined buffers` commit
  ([03-kernel.md](03-kernel.md#patches)): without it most of the time goes into cache cleaning.
- `cma=192M` on the command line.

## Building Mesa for it

Build in an armv7 Alpine (on the AC100 itself, slowly, or in an Alpine armv7 chroot under
qemu-user on a PC). The packages the build root had (Alpine 3.24):

```sh
apk add build-base linux-headers meson samurai pkgconf python3 py3-mako py3-packaging py3-yaml \
        bison flex expat-dev zlib-dev zstd-dev libdrm-dev wayland-dev wayland-protocols \
        libx11-dev libxcb-dev libxdamage-dev libxext-dev libxfixes-dev libxrandr-dev \
        libxshmfence-dev libxxf86vm-dev
```

```sh
git clone --depth=1 -b ac100-grate https://github.com/toomykins/mesa.git && cd mesa
meson setup build --prefix=/opt/grate --buildtype=release -Db_ndebug=true \
  -Dgallium-drivers=grate,softpipe -Dvulkan-drivers= -Dllvm=disabled \
  -Dplatforms=x11,wayland -Degl=enabled -Dgbm=enabled -Dglx=dri -Dopengl=true \
  -Dgles1=disabled -Dgles2=enabled -Dshared-glapi=enabled -Dglvnd=disabled \
  -Dgallium-va=disabled -Dgallium-rusticl=false -Dvideo-codecs= -Dtools= \
  -Dlibunwind=disabled -Dvalgrind=disabled -Dlmsensors=disabled -Dzstd=enabled \
  -Dc_args=-mtls-dialect=gnu -Dcpp_args=-mtls-dialect=gnu
ninja -C build && ninja -C build install
```

- **`-mtls-dialect=gnu` is required.** Mesa's configure test for TLS descriptors passes under qemu,
  but the resulting library hangs in its first GL call on the device. Alpine's own Mesa package
  forces the same flag.
- The prefix `/opt/grate` keeps the distribution's Mesa untouched. Start programs with
  `LD_LIBRARY_PATH=/opt/grate/lib`, for example `LD_LIBRARY_PATH=/opt/grate/lib weston --backend=drm --renderer=gl`.
- `GRATE_DEBUG=fp` prints the fragment programs, `MESA_GLSL=errors` shader link errors.

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
working notes from this bring-up; the commit messages on the branch describe each step.
