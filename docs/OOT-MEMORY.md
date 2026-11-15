# Ocarina of Time on a standard PS5: where the 12 GiB pool goes

Status 2026-11-15, branch `zelda-memory` on v1.000.002. Console: standard PS5 (12,224 MiB pool, no
Pro memory), firmware 12.00. Game: OoT (0400486024DEC000) v131072, `.nup`.

## Symptoms

- v1.000.002 as released: the process is killed by the system, or SIGSEGV at `eboot+0x21e1629`
  (RADV `begin_subpass`, null attachment), 2–3 minutes in, at the first area change after the title.
- Slow motion before the crash: frames over 100 ms (the built-in Dynamic FPS keeps real speed only
  down to 10 FPS).

## Accounting at the crash (my build, 0.5x, RADV report every 10 s)

| Holder | MiB |
|---|---|
| Guest memory held (written 2 MiB slots) | 5,100–5,300 |
| RADV buffers (p14 application memory 3,250–3,525 in ~150–170 BOs; shaders p31 104; p30 7) | 3,360–3,640 |
| Eden heap (1,250 in use) | 1,544 |
| JIT code caches (six per-core caches, 50–134 each) | ~450 |
| Kernel, page tables, reserves, libraries, stacks | ~1,500 |
| Free | 6–46 |

Inside the VMA memory at 86 s (aggressive collector run): 3,273 MiB of VMA blocks against texture-cache
images 381 MiB and buffer cache 509 MiB; staging pools held 0.5–0.9 GB of idle buffers.

## Findings

1. **Guest memory barely compresses**: `EDEN_CRAM_COLD` lz4 ratio 0.83 (MKW 0.27). Packing, the
   lever that let MKW reach races, saves ~17% here; it packed 330 → 97 MiB in one run at the cost of
   91 unpack faults (11 on GPUBack).
2. **Resolution below 1x adds memory**: a rescaled image keeps `original_image` and `scaled_image`
   (vk_texture_cache.cpp `Image::ScaleUp`), and copies between them. 0.5x/0.75x is a cost, not a saving.
3. **Low texture-cache limits thrash**: with `EDEN_NX2_TC_*` at 1–1.5 GiB the collector (pool-based
   usage, read as 5,654 MiB) stayed aggressive: 2,924 images created, 2,035 deleted, 736 downloaded
   back to guest memory per 600 frames.
4. **Lost in the move to the in-repo core** (old patch series → `Eglantine/`): 0023 Vulkan OOM recovery,
   0056 index-buffer span cap, 0059 the GPU-reach hook (`eden_guest_gpu_cached` was defined, never called).
5. **Staging**: idle buffers over 32 MiB stayed up to 64 frames (one level swept per frame).
6. **Decode images**: up to 256 MiB of finished transcode images kept 60 frames, with no release when short.
7. **Recovery retried slots that do not pack** on every call (no back-off, unlike the scanner).
8. **The RADV memory report never ran**: it was tied to budget queries the pool-based collector no longer makes.
9. **The log lost its last lines**: the file flushed only on Error lines; a killed app lost the seconds before.
10. `EDEN_HEAP_SITES_LOST` is cumulative (frees are not subtracted), not live memory.

## Changes on the branch

- Ported 0023, 0056, 0059.
- Framebuffers leave out render targets with no image memory (RADV `begin_subpass` null deref).
- Staging buffers over 32 MiB released every frame once idle.
- Decode images not kept while the pool is short.
- Recovery backs off from slots that would not pack.
- Diagnostics: RADV per-priority report every 10 s; VMA per-type blocks/allocations and kept staging
  bytes at Info; log flushed at least every 2 s.

## Open

- What fills the rest of the VMA memory at 1x (next run with `EDEN_NX2_TC_LOG=1`).
- JIT: per-core caches for Switch 2 total up to 688 MiB; capping costs recompiles.
- Slow motion: memory-recovery stalls, unpack faults and render-thread pipeline builds on first visits.
