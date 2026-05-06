# CLAUDE.md — mos-patcher

Lilu replacement for the **mos** suite. Per-instance vtable swap via
IOService publish notifications because Sequoia broke in-place
`__DATA_CONST` patching. ~1.2 KLOC of C++ across five source files
we own (see [Source layout](#source-layout)).

This file is for AI coding agents and humans dropping into this repo.
Project-wide context lives in the parent project's working notes —
prefer those when in doubt.

## Status

**Stable.** v0.5, runtime-proven on Sequoia 15.7.5 inside QEMU/KVM.
24/24 `IONDRVFramebuffer` methods hooked end-to-end via the publish
path with zero gaps. The framework does what it's supposed to.
Open work is hardening (multi-consumer isolation, multi-version
testing, ARM64 port) — not redesign.

## Naming

On disk: `mos15-patcher`. Public canonical: `mos-patcher`. Bundle ID
is `com.docker-macos.kext.mos15Patcher`. The "15" was a Sequoia-era
shorthand we kept to avoid breaking cross-repo links. Don't rename
in code; do refer to it as `mos-patcher` in user-facing docs.

## Standing rules

Universal mos rules: **see `../mos/CLAUDE.md`**. The ones
specifically load-bearing here:

- **Per-instance vtable swap only.** Never patch `__DATA_CONST` in
  place. CR0.WP toggling crashes Sequoia kernels. See
  `../mos/memory/feedback_per_instance_vtable_swap.md`.
- **Trampolines safe; `routeMultiple`-style batches risky.** A
  single failed mangled-name resolution must not abort the rest of
  the batch. We log per-route status in `MPStatus` ioreg properties
  and continue. See `../mos/memory/feedback_lilu_route_safety.md`.
- **Never scan the full kmod chain.** `mp_route_on_publish` takes a
  NULL-terminated array of bundle IDs (primary first, fallbacks
  after). Callers pass it explicitly.
- **No `Co-Authored-By: Claude`** trailers. See
  `../mos/memory/feedback_no_ai_attribution_in_commits.md`.

## Build / iteration loop

```sh
KERN_SDK=/path/to/MacKernelSDK ./build.sh
# build/mos15-patcher.kext

# Bundle into OpenCore.img (handled by mos-opencore build script
# in the typical workflow), then reboot the VM.
```

There is no userspace test harness — the kext only does anything
inside a running kernel. Iteration is: edit → `./build.sh` → cp
into `OpenCore.img` → reboot the QEMU VM → `ioreg -l | grep MP`
to read per-route status. Allow ~30 s per cycle.

## Source layout

| File | Role |
|---|---|
| `include/mos15_patcher.h` | Public API: `mp_route_on_publish`, `mp_route_kext`, `mp_route_addr`, `MP_ROUTE_*` macros |
| `src/start.cpp` | Kext entry, API impl, pending-route drain timer |
| `src/notify.cpp` | Publish-notification subscription + per-instance vtable swap (the heart) |
| `src/vtable.cpp` | Vtable-slot scan + replacement |
| `src/macho.cpp` | Mach-O symbol lookup (Boot KC + System KC + Aux KC) — universal `__LINKEDIT`-relative formula |
| `src/patch.cpp` | x86_64 14-byte absolute-JMP prologue patching + trampolines |
| `src/kmod_info.c` | `kmod_info_t` for the kext loader |
| `Info.plist` | Matches on `IOResources`/`IOKit`. **No `IOPCIPrimaryMatch`** — this kext doesn't bind to a PCI device. |

The patcher is x86_64-only (the prologue patcher is ISA-specific);
ARM64 is on the v1.0 list, not v0.5. Cross-compile target is
`x86_64-apple-macos10.15` regardless of host arch — `build.sh`
runs cleanly on Apple Silicon.

## When in doubt

Read **`../mos-docs/whitepapers/08-ioservice-publish-vs-vtable-patch.md`**.
That whitepaper is the canonical design rationale for this kext —
why per-instance, why publish notifications, why we wrote our own
instead of patching Lilu, the three-mode operator-fallback pattern,
the universal symbol-resolver formula. If you're touching anything
non-trivial in `notify.cpp` or `macho.cpp`, read that first.

For the kext bundle list and load order, see
`../mos-docs/reference/kext-bundle-list.md`.

For the project-wide component map (six repos, dependency graph),
see `../mos-docs/architecture/01-component-map.md`.
