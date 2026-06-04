# NIXL + nixlbench on AMD ROCm/HIP

Validated on AMD Instinct MI300X (`gfx942`) and MI355X (`gfx950`) with
AMD Pensando AINIC (ionic) RoCE NICs.

## Use the pre-built image (fast path)

A ready-to-go image is published at:

```
ghcr.io/ai-dynamo/nixl-rocm:base-latest
```

This image has UCX 1.21.x built with `--with-rocm --with-verbs`, all
apt-side build deps, and the Python build toolchain. **It does not
contain the NIXL or nixlbench binaries** — those are built on top by
the consumer so the image is reusable across nixl branches/PRs.

To use it for a one-off build:

```bash
podman run --rm -it \
  --device /dev/kfd --device /dev/dri --device /dev/infiniband \
  --group-add keep-groups --security-opt seccomp=unconfined \
  --network=host --ipc=host \
  -v /boot:/boot:ro \
  -v $(pwd):/workspace/nixl \
  -e UCX_ROCM_COPY_DMABUF=yes \
  -e UCX_ROCM_IPC_MIN_ZCOPY=0 \
  ghcr.io/ai-dynamo/nixl-rocm:base-latest \
  bash -c '
    cd /workspace/nixl && \
    meson setup build -Ducx_path=/usr/local/ucx -Dwheel_variant=rocm && \
    ninja -C build install && \
    cd benchmark/nixlbench && \
    meson setup build -Duse_rocm=true -Drocm_path=/opt/rocm -Dnixl_path=/usr/local && \
    ninja -C build install && \
    /usr/local/nixlbench/bin/nixlbench --help
  '
```

## Build the image yourself

`Dockerfile.rocm` in this directory contains the full multi-stage
recipe. Build from the repo root:

```bash
docker build -f benchmark/nixlbench/contrib/Dockerfile.rocm \
             -t nixl-rocm:latest .
```

The Dockerfile builds in four stages:

1. **base** — Ubuntu 24.04 + ROCm 7.1.1-complete + apt deps + python
   toolchain
2. **ucx-build** — UCX from source with `--with-rocm --with-verbs`
3. **nixl-build** — NIXL + nixlbench compiled and installed on top
4. **runtime** — slimmer final layer with only install artifacts

## Why the funny `/boot` mount + UCX env vars?

AMD Pensando AINIC NICs require UCX's dmabuf-based VRAM registration
path. Without it, UCX falls back to `ibv_reg_mr` on raw VRAM pointers,
which the ionic kernel driver rejects with `EINVAL`. To unlock the
dmabuf path:

1. **`-v /boot:/boot:ro`** — UCX's `uct_rocm_base_is_dmabuf_supported()`
   reads `/boot/config-$(uname -r)` to verify `CONFIG_PCI_P2PDMA=y` +
   `CONFIG_DMABUF_MOVE_NOTIFY=y` are compiled into the host kernel.
   Without `/boot` mounted, the check silently returns false and the
   dmabuf code path is disabled.

2. **`UCX_ROCM_COPY_DMABUF=yes`** — opt into the dmabuf path in the
   `rocm_copy` memory domain (default is `no` in upstream UCX). This is
   the only `*_DMABUF` config knob exposed in upstream UCX v1.21.x; the
   `rocm_ipc` MD inherits dmabuf behavior via the shared rocm base
   detection rather than its own switch.

3. **`UCX_ROCM_IPC_MIN_ZCOPY=0`** — required to engage the rocm_ipc
   zero-copy path; see @tvegas1 [comment on #1642][1].

These two knobs are pre-set as `ENV` in the `runtime` stage so
users only need to remember the `-v /boot:/boot:ro` bind-mount.

[1]: https://github.com/ai-dynamo/nixl/pull/1642#issuecomment-4603794788

## CI integration

- **`rocm-build-check.yml`** — compile-only CI on every PR touching
  NIXL or nixlbench source. Uses the pre-built `ghcr.io/...` image so
  only NIXL itself rebuilds. Per @edgargabriel's short-term proposal
  on [#1647][2].
- **`wheel-rocm.yml`** — `nixl_rocm` wheel generation on tagged
  releases, uses `wheel_variant=rocm` (added by #1642). Per
  @tvegas1's release-side ask on [#1647][3].

[2]: https://github.com/ai-dynamo/nixl/pull/1647#discussion_r_2604240944
[3]: https://github.com/ai-dynamo/nixl/pull/1647#discussion_r_2604233231

## Cross-node validation

Pairs naturally with the example launch scripts at
[`ROCm/dynamo-examples`](https://github.com/ROCm/dynamo-examples) which
exercise NIXL/nixlbench through Dynamo's SGLang and vLLM backends.

## Stacks on

- ai-dynamo/nixl#1642 — adds `wheel_variant` Meson option (merged)
- ai-dynamo/nixl#1647 — adds `use_rocm` Meson option + ROCm code paths
  in nixlbench
