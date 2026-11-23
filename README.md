# Turnip patches for A8XX

Patches for Mesa's Turnip Vulkan driver on Adreno A8XX GPUs: mesh shaders,
fragment shader barycentrics, 64-bit shared atomics, experimental ray tracing
pipelines and some fixes. Developed and tested on a Snapdragon X2 Elite Extreme
laptop (Adreno X2-90), partly also on an Adreno 840 phone.

They apply on top of upstream Mesa main at `6c2135ab766` ("ac: fix off-by-one
guardband computation", 2026-11-20).

Use at your own risk. Testing was mostly Vulkan CTS and a handful of games,
and much of this is still work in progress.

## Applying

`series` lists every patch in order. From a Mesa checkout:

```
git checkout -b turnip-a8xx 6c2135ab766
TP=/path/to/turnip-patches
git am $(sed "s|^|$TP/|" $TP/series)
```

Also apply upstream MR !46551 (Artem Peshkov's A8XX VPC position cache size
fix) until it lands.

## What you need for what

| Feature | Patches |
|---------|---------|
| Ray tracing pipelines | The whole series. Only `shader_int64_atomics` and `fully_covered` can be left out |
| Mesh shaders, barycentrics | 0001 to 0009, in order |
| 64-bit shared atomics | `shader_int64_atomics`, applies on its own |
| Fully covered fragments | `fully_covered`, applies on its own |
| X2-90 support | `x2-90`, applies on its own |
| Upstream bug fixes | `fixes`, applies on its own |

Everything not listed as "on its own" only applies in `series` order.

### Ray tracing pipelines

The extension itself is 0010 and 0011, but it needs the rest to be usable
and fast:

| Patches | Why |
|---------|-----|
| 0001 to 0009 | 0011 is built on top of them |
| 0010, 0011 | `VK_KHR_ray_tracing_pipeline` |
| 0032 | Fixes slow global atomics that 0011 caused (Quake II RTX ray queries 45 to 59 FPS) |
| 0021, 0022 | Big pipelines (Cyberpunk 2077) compile without running out of memory, about 5x faster |
| 0023 | Fixes GPU faults after acceleration structure compaction |
| 0024, 0025 | Acceleration structure builds 15 to 25x faster |
| 0026 to 0028 | Hardware BVH traversal step, 10 to 17% faster |
| 0029 to 0031 | Ray stack in GMEM, up to 2x faster with big payloads |
| 0012 | 0031 doesn't apply without it |
| 0015 | Fixes a GPU hang on the X2-90 |

Cyberpunk 2077 benchmark on the X2-90 (2560x1600, FSR 2.1 Balanced, RT local
shadows): 3.5 FPS with 0011 alone, 16.9 with 0025, about 19 with everything.
It passes the ray tracing pipeline CTS and runs Quake II RTX.

It's off unless `TU_EXPERIMENTAL` contains `rt_pipeline`.

## Patches

| # | Folder | What |
|---|--------|------|
| 0001 | mesh_shader | `VK_EXT_mesh_shader` for A8XX |
| 0002 | fragment_shader_barycentric | `VK_KHR_fragment_shader_barycentric` for A8XX |
| 0003 | subgroups | Emulate 32-wide subgroups |
| 0004 | fragment_shader_barycentric | Fix memory growth, XFB and mesh keys with barycentric libraries |
| 0005 | fixes | Weak-ref device pipeline cache |
| 0006 | mesh_shader | Extend the native A8XX mesh path |
| 0007 | performance | Reduce fragment preamble cost |
| 0008 | fixes | kgsl: split bind ranges on sparse unbind |
| 0009 | mesh_shader | Native mesh path only when the kernel enabled the AQE |
| 0010 | ray_tracing_pipeline | nir: keep hit attribute writes alive across `report_ray_intersection` |
| 0011 | ray_tracing_pipeline | `VK_KHR_ray_tracing_pipeline` for A8XX |
| 0012 | local | `TU_DEBUG=deck_emu` |
| 0013 | local | `TU_DBG_BOLOG` and `TU_DBG_AS_LOG` debug logging |
| 0014 | local | Turn mesh_shader, mesh_native and barycentric on by default |
| 0015 | x2-90 | Full sysmem depth CCU cache on X2-90, fixes a GPU hang |
| 0016 | x2-90 | X2-90 chip id the laptop kernel reports |
| 0017 | shader_int64_atomics | ir3: 64-bit shared memory atomics |
| 0018 | shader_int64_atomics | Expose `shaderSharedInt64Atomics` |
| 0019 | shader_int64_atomics | ir3: native 64-bit umin/umax atomics |
| 0020 | fully_covered | Expose `fullyCoveredFragmentShaderInputVariable` |
| 0021 | fixes | ir3: spill keeps next-use distances only for live-in and live-out values |
| 0022 | fixes | ir3: don't rescan the whole shader after every jump change |
| 0023 | fixes | Fix header pointers when copying an acceleration structure |
| 0024 | fixes | nir: merge atomics only when the buffer or image is uniform too |
| 0025 | performance | ir3: uniform-address atomics once per subgroup in compute |
| 0026 | ray_stack | ir3/isa: A8XX `ray_stack` instruction |
| 0027 | ray_stack | ir3: `ray_stack_ir3` and its reserved state block |
| 0028 | ray_stack | Traverse the BVH with `ray_stack` |
| 0029 | gmem_stack | ir3: `getspid` and `getwid` for the SM and warp ID intrinsics |
| 0030 | gmem_stack | Fix alignment of the separate mode stack accesses |
| 0031 | gmem_stack | Keep the bottom of the separate mode ray stack in GMEM |
| 0032 | ray_tracing_pipeline | ir3: encode the coherent bit of global atomics |

0021 to 0024 fix bugs that are also in upstream Mesa. `local` holds hacks for
my own setup that won't go upstream.

## Environment variables

`TU_EXPERIMENTAL` takes a comma-separated list of `mesh_shader`,
`mesh_native` (needs a kernel with the AQE), `barycentric` and `rt_pipeline`.
Everything is off by default. With 0014 the default is
`mesh_shader,mesh_native,barycentric`, and setting the variable replaces it,
so for ray tracing use
`TU_EXPERIMENTAL=mesh_shader,mesh_native,barycentric,rt_pipeline`.

Ray tracing:

- `TU_RT_MODE=monolithic|separate|aqe` forces how pipelines compile (`aqe`
  needs the AQE kernel). Add `,log` to print what each pipeline got. By
  default small pipelines are monolithic and big ones separate.
- `TU_DEBUG=nogmemstack` keeps the ray stack out of GMEM.
- `TU_RT_AQE_MAX_ACTIVE=<n>` caps rays in flight in `aqe` mode (testing).
- `TU_DBG_RT_SKIP=1` skips ray tracing dispatches (debugging).

Debugging:

- `TU_DEBUG=deck_emu` reports a Steam Deck GPU (RADV, 0x1002:0x163F).
- `TU_DBG_BOLOG=<dir>` logs BO maps and unmaps to `<dir>/bolog-<pid>.log`.
- `TU_DBG_AS_LOG=1` logs acceleration structure builds.

## License

MIT, see [LICENSE](LICENSE), the same as the Mesa code these patches change.
The copyright notices in those Mesa files stay as they are.
