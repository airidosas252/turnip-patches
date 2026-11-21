# Turnip patches for A8XX (Adreno 840 / X2-90)

A series of patches for Mesa's Turnip Vulkan driver. Mesh shaders, fragment
shader barycentrics, 64-bit shared atomics and a few fixes for A8XX, plus an
experimental ray tracing pipeline.

Base: upstream Mesa main at `6c2135ab766` ("ac: fix off-by-one guardband
computation", 2026-11-20).

## Applying

```
git checkout -b turnip-a8xx 6c2135ab766
git am *.patch
```

A840 and X2-90 also need upstream MR !46551 (the A8XX VPC position cache size
fix by Artem Peshkov) until it lands. It is not part of this series.

## Patches

| # | What |
|---|------|
| 0001 | `VK_EXT_mesh_shader` for A8XX |
| 0002 | `VK_KHR_fragment_shader_barycentric` for A8XX |
| 0003 | Emulate 32-wide subgroups on A8XX |
| 0004 | Fix memory growth, XFB and mesh keys with barycentric libraries |
| 0005 | Use a weak-ref device pipeline cache |
| 0006 | Extend the native A8XX mesh path |
| 0007 | Reduce fragment preamble cost |
| 0008 | kgsl: split bind ranges on sparse unbind |
| 0009 | Use the native mesh path only when the kernel enabled the AQE |
| 0010 | nir: keep hit attribute writes alive across `report_ray_intersection` |
| 0011 | `VK_KHR_ray_tracing_pipeline`, experimental, see the warning below |
| 0012 | `TU_DEBUG=deck_emu`, local hack |
| 0013 | `TU_DBG_BOLOG` and `TU_DBG_AS_LOG` debug logging, local hack |
| 0014 | Turn on mesh_shader, mesh_native and barycentric by default, local hack |
| 0015 | freedreno: full sysmem depth CCU cache on X2-90 (fixes a GPU hang) |
| 0016 | freedreno: add the X2-90 chip id reported by the laptop kernel |
| 0017 | ir3: 64-bit shared memory atomics |
| 0018 | Expose `shaderSharedInt64Atomics` on A8XX |
| 0019 | ir3: native 64-bit umin/umax atomics on A8XX |
| 0020 | Expose `fullyCoveredFragmentShaderInputVariable` on A8XX |
| 0021 | ir3: keep spill next-use distances only for live-in and live-out values (fixes running out of memory on huge shaders) |
| 0022 | ir3: don't rescan the whole shader after every jump change (big shaders compile about 5x faster) |
| 0023 | Rebase the header pointers when copying an acceleration structure (fixes GPU faults after compaction) |

Patches 0012, 0013 and 0014 are local hacks and not meant for upstream. Skip
them if you only want the features.

## Ray tracing pipeline (0011): experimental

It passes the ray tracing pipeline CTS, small test apps and Quake II RTX. With
0021 to 0023, Cyberpunk 2077 runs with every ray tracing option and no
crashes, but it is very slow. Cronos (46 stage pipeline) used to run out of
memory while compiling and has not been tested again since 0021. It stays off
unless `TU_EXPERIMENTAL` contains `rt_pipeline`.

0021 to 0023 also fix bugs in upstream code (0023 affects ray queries too), so
they are useful without 0011.

## Environment variables

`TU_EXPERIMENTAL` is a comma separated list of:

- `mesh_shader`: `VK_EXT_mesh_shader`
- `mesh_native`: the native A8XX mesh path (needs a kernel that enabled the AQE)
- `barycentric`: `VK_KHR_fragment_shader_barycentric`
- `rt_pipeline`: `VK_KHR_ray_tracing_pipeline`, experimental, see above

Without 0014 all of these are off unless you set the variable. With 0014 the
default is `mesh_shader,mesh_native,barycentric` when `TU_EXPERIMENTAL` is
unset, and setting it replaces that list. To try ray tracing pipelines:
`TU_EXPERIMENTAL=mesh_shader,mesh_native,barycentric,rt_pipeline`.

Ray tracing pipeline compile mode (0011), only with `rt_pipeline`:

- `TU_RT_MODE=monolithic`, `separate` or `aqe` forces a mode, `log` prints how
  each pipeline compiled. Combine with commas. By default small pipelines are
  monolithic and big ones are separate. `aqe` needs a kernel with the AQE.
- `TU_RT_AQE_MAX_ACTIVE=<n>` limits the rays in flight in `aqe` mode, for testing.
- `TU_DBG_RT_SKIP=1` skips the ray tracing launches, a debugging aid.

Debugging (0012, 0013):

- `TU_DEBUG=deck_emu` reports a Steam Deck GPU (RADV, 0x1002:0x163F).
- `TU_DBG_BOLOG=<dir>` logs BO map and unmap ranges to `<dir>/bolog-<pid>.log`.
- `TU_DBG_AS_LOG=1` logs acceleration structure builds.
