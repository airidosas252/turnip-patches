# Turnip patches for A8XX

Patches for Mesa's Turnip Vulkan driver on Adreno A8XX GPUs. They add mesh
shaders, fragment shader barycentrics and 64-bit shared atomics, fix a few
things, and add an experimental ray tracing pipeline.

I developed and tested them on a Snapdragon X2 Elite Extreme laptop (Adreno
X2-90), and some of the earlier work also on an Adreno 840 phone. Other A8XX
devices should work in principle, but your mileage will vary.

They apply on top of upstream Mesa main at `6c2135ab766` ("ac: fix off-by-one
guardband computation", 2026-11-20).

## Warning

Use these at your own risk. I can't promise they'll work for you. Most of the
testing was Vulkan CTS plus a handful of games that use these features, and a
lot of it is still work in progress.

## Applying

The patches are in folders by feature, and `series` lists all of them in the
order they apply. From a Mesa checkout:

```
git checkout -b turnip-a8xx 6c2135ab766
TP=/path/to/turnip-patches
git am $(sed "s|^|$TP/|" $TP/series)
```

The A840 and X2-90 also need upstream MR !46551, Artem Peshkov's fix for the
A8XX VPC position cache size. It isn't in this series, so apply it as well
until it lands upstream.

### Picking only some folders

`fixes`, `x2-90`, `shader_int64_atomics` and `fully_covered` apply on their
own, in any combination:

```
git am $TP/patches/fixes/*.patch $TP/patches/x2-90/*.patch
```

The rest build on each other. `mesh_shader`, `fragment_shader_barycentric`,
`subgroups`, `performance` and `ray_tracing_pipeline` (0001 to 0011, plus
0025) only apply together and in `series` order. In `local`, 0014 needs the
mesh and barycentric patches, while 0012 and 0013 apply on their own.

## Patches

The numbers keep the order of the whole series.

**mesh_shader**

| # | What | Comments |
|---|------|----------|
| 0001 | `VK_EXT_mesh_shader` for A8XX | WIP, needs more testing |
| 0006 | Extend the native A8XX mesh path | |
| 0009 | Use the native mesh path only when the kernel enabled the AQE | |

**fragment_shader_barycentric**

| # | What | Comments |
|---|------|----------|
| 0002 | `VK_KHR_fragment_shader_barycentric` for A8XX | WIP, needs more testing |
| 0004 | Fix memory growth, XFB and mesh keys with barycentric libraries | |

**subgroups**

| # | What | Comments |
|---|------|----------|
| 0003 | Emulate 32-wide subgroups on A8XX | |

**performance**

| # | What | Comments |
|---|------|----------|
| 0007 | Reduce fragment preamble cost | |
| 0025 | ir3: do uniform-address atomics once per subgroup in compute shaders | Big acceleration structure builds 15-25x faster, needs 0024 |

**ray_tracing_pipeline**

| # | What | Comments |
|---|------|----------|
| 0010 | nir: keep hit attribute writes alive across `report_ray_intersection` | |
| 0011 | `VK_KHR_ray_tracing_pipeline` for A8XX | Experimental, see below |

**shader_int64_atomics**

| # | What | Comments |
|---|------|----------|
| 0017 | ir3: 64-bit shared memory atomics | |
| 0018 | Expose `shaderSharedInt64Atomics` on A8XX | |
| 0019 | ir3: native 64-bit umin/umax atomics on A8XX | |

**fully_covered**

| # | What | Comments |
|---|------|----------|
| 0020 | Expose `fullyCoveredFragmentShaderInputVariable` on A8XX | |

**x2-90**

| # | What | Comments |
|---|------|----------|
| 0015 | freedreno: full sysmem depth CCU cache on X2-90 | Fixes a GPU hang |
| 0016 | freedreno: add the X2-90 chip id the laptop kernel reports | |

**fixes**

| # | What | Comments |
|---|------|----------|
| 0005 | Use a weak-ref device pipeline cache | |
| 0008 | kgsl: split bind ranges on sparse unbind | |
| 0021 | ir3: keep spill next-use distances only for live-in and live-out values | Huge shaders no longer run out of memory |
| 0022 | ir3: don't rescan the whole shader after every jump change | Huge shaders compile about 5x faster |
| 0023 | Fix the header pointers when copying an acceleration structure | Fixes GPU faults after compaction |
| 0024 | nir: only merge atomics when the buffer or image is uniform too | Needed by 0025 |

0021 to 0024 fix bugs that are also in upstream Mesa, so they're worth having
even without the ray tracing patch. 0023 affects ray queries too.

**local**

| # | What | Comments |
|---|------|----------|
| 0012 | `TU_DEBUG=deck_emu` | Local hack |
| 0013 | `TU_DBG_BOLOG` and `TU_DBG_AS_LOG` debug logging | Local hack |
| 0014 | Turn mesh_shader, mesh_native and barycentric on by default | Local hack |

These are hacks for my own setup and won't go upstream. Skip them if you only
want the features.

## Ray tracing pipelines (0011)

This one is experimental. It passes the ray tracing pipeline CTS and runs a few
small test apps and Quake II RTX. Cyberpunk 2077 also works with every ray
tracing option (with 0021 to 0023), but it's very slow. Cronos (a 46-stage
pipeline) used to run out of memory while compiling, and I haven't tried it
again since 0021.

It's off unless `TU_EXPERIMENTAL` contains `rt_pipeline`.

## Environment variables

`TU_EXPERIMENTAL` takes a comma-separated list of:

- `mesh_shader`: `VK_EXT_mesh_shader`
- `mesh_native`: the native A8XX mesh path, needs a kernel with the AQE enabled
- `barycentric`: `VK_KHR_fragment_shader_barycentric`
- `rt_pipeline`: `VK_KHR_ray_tracing_pipeline` (experimental)

Without 0014, everything is off until you set it. With 0014, the default is
`mesh_shader,mesh_native,barycentric`, and setting the variable replaces that
list rather than adding to it. So to try ray tracing pipelines, use
`TU_EXPERIMENTAL=mesh_shader,mesh_native,barycentric,rt_pipeline`.

These only matter with `rt_pipeline` (0011):

- `TU_RT_MODE` forces how pipelines get compiled: `monolithic`, `separate` or
  `aqe` (`aqe` needs a kernel with the AQE). Add `log` to print how each
  pipeline compiled, for example `TU_RT_MODE=separate,log`. By default, small
  pipelines are monolithic and big ones are separate.
- `TU_RT_AQE_MAX_ACTIVE=<n>` caps the rays in flight in `aqe` mode. For testing.
- `TU_DBG_RT_SKIP=1` skips ray tracing dispatches. For debugging.

Debugging (0012, 0013):

- `TU_DEBUG=deck_emu` makes the driver report a Steam Deck GPU (RADV,
  0x1002:0x163F).
- `TU_DBG_BOLOG=<dir>` logs every BO map and unmap to `<dir>/bolog-<pid>.log`.
- `TU_DBG_AS_LOG=1` logs acceleration structure builds.

## License

MIT, see [LICENSE](LICENSE). That's the same license as the Mesa code these
patches change, and the copyright notices already in those Mesa files stay as
they are.
