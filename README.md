# mos-patcher

[![Release](https://img.shields.io/github/v/release/MattJackson/mos-patcher?display_name=tag&sort=semver)](https://github.com/MattJackson/mos-patcher/releases)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

A Lilu-style kernel-extension hook framework for macOS 15 (Sequoia),
built around the only patching mechanism that survives Sequoia's
hardened `__DATA_CONST`: **per-instance vtable swap via IOService
publish notifications**. ~1.2 KLOC of C++ across five source files;
replaces Lilu in the [mos](https://github.com/MattJackson/mos) suite.

> **Status: v0.5 — usable but barebones.** Runtime-proven on macOS
> 15.7.5 (Sequoia, x86_64) inside QEMU/KVM. Production hardening
> incomplete; APIs may still change. See [Status](#status).

## Repo naming

This repo is **`mos15-patcher`** on disk and as the GitHub repository
slug. The public canonical name across the project is **`mos-patcher`**
— used consistently in
[mos-docs](https://github.com/MattJackson/mos-docs), in `mos/README.md`,
and in the kext bundle list. The `15` was a Sequoia-specific shorthand
that outlived its usefulness; we kept the on-disk name to avoid
breaking links into the repo. The other mos satellites have the same
divergence (`qemu-mos15` ↔ `mos-qemu`, `opencore-mos15` ↔
`mos-opencore`). Treat the names as interchangeable.

The kext bundle ID is `com.docker-macos.kext.mos15Patcher` and the
built artifact is `mos15-patcher.kext`.

## Why this exists

For roughly a decade the standard pattern for hooking macOS kext
methods was: locate the method in `__DATA_CONST`, toggle CR0.WP off,
write a 14-byte absolute JMP at the prologue (or rewrite the vtable
slot in place), restore CR0.WP. That's how
[Lilu](https://github.com/acidanthera/Lilu)'s
`KernelPatcher::routeMultiple` works.

**Sequoia broke that pattern.** macOS 15 hardens `__DATA_CONST` page
protection in a way the CR0.WP-toggle trick can't bypass — writes
panic the kernel, every boot, no exceptions. The fallback is to
patch each IOKit instance's *vtable pointer* (the first 8 bytes of
the C++ object, on the heap) so it points at a writable copy
containing your slots. That's what this kext does.

We also tried Lilu first. On Sequoia, in our VM, `onKextLoad` only
fired on roughly 60% of boots for System KC kexts that loaded before
Lilu's `activate()` ran — a race in early-boot kext-load detection.
Rather than fix it upstream in 10–15 KLOC of C++ class hierarchy,
we wrote a smaller framework that solves three things and nothing
else:

1. **Detect IOService class instantiation.** Register
   `IOService::addMatchingNotification` on a class name. Every
   current and future instance is delivered to our callback.
2. **Look up Mach-O symbols** by mangled name in any loaded kext
   (Boot KC, System KC, Aux KC) with one universal
   `__LINKEDIT`-relative resolution formula — no
   standalone-vs-KC-embedded heuristic.
3. **Swap per-instance vtables** for IOService classes
   (Sequoia-safe — no `__DATA_CONST` writes), or patch function
   prologues with 14-byte absolute JMPs for static functions.

Plus prefix-based symbol matching so consumers don't hand-mangle
Itanium C++ ABI parameter signatures.

For the full design rationale, including how this fits beside Lilu's
`routeMultiple` and the three-mode operator-fallback pattern that
generalizes to anything built on top, read
**[whitepaper 08 — IOService publish vs vtable patch](https://github.com/MattJackson/mos-docs/blob/main/whitepapers/08-ioservice-publish-vs-vtable-patch.md)**.

## Where this fits in mos

| Repo | Role |
|---|---|
| [mos-docker](https://github.com/MattJackson/mos-docker) | The runtime — Docker image that boots macOS Sequoia in QEMU/KVM |
| [mos-qemu](https://github.com/MattJackson/mos-qemu) | QEMU 10.2.2 fork (`applesmc`, `vmware_vga`, `dev-hid`, `apple-gfx-pci-linux`) |
| [libapplegfx-vulkan](https://github.com/MattJackson/libapplegfx-vulkan) | Host-side `ParavirtualizedGraphics.framework` against Vulkan/lavapipe |
| **mos-patcher** (this repo) | Per-instance vtable swap on Sequoia |
| [mos-opencore](https://github.com/MattJackson/mos-opencore) | OpenCore EFI image build script |
| [mos-docs](https://github.com/MattJackson/mos-docs) | Documentation library — overview, architecture, whitepapers, reference |
| [mos](https://github.com/MattJackson/mos) | Project meta-repo — RE notes, milestones, memory |

The `mos15-patcher.kext` artifact is bundled into `OpenCore.img` by
[mos-opencore](https://github.com/MattJackson/mos-opencore)'s build
script and loaded at boot via `Kernel.Add` in `config.plist`. Its
sole consumer today is `QEMUDisplayPatcher` in
[mos-docker/kexts/](https://github.com/MattJackson/mos-docker/tree/main/kexts/QEMUDisplayPatcher),
which uses `mp_route_on_publish` to redirect 24/24 `IONDRVFramebuffer`
methods. See
[reference/kext-bundle-list.md](https://github.com/MattJackson/mos-docs/blob/main/reference/kext-bundle-list.md)
for the full bundle list and load order.

For project status, milestones, and where the M5 push currently is,
see
[overview/project-status.md](https://github.com/MattJackson/mos-docs/blob/main/overview/project-status.md).

## Quick start

```c
#include <mos15_patcher.h>

static IOReturn (*orgEnableController)(void *) = nullptr;

static IOReturn patchedEnableController(void *that) {
    IOReturn r = orgEnableController(that);
    // ... your code ...
    return r;
}

extern "C" kern_return_t my_start(kmod_info_t *ki, void *d) {
    // NULL-terminated chain: primary kext first, fallbacks after.
    static const char *const kexts[] = {
        "com.apple.iokit.IONDRVSupport",
        "com.apple.iokit.IOGraphicsFamily",
        nullptr,
    };
    mp_route_request_t reqs[] = {
        // Apple's blessed path — vtable swap on every IONDRVFramebuffer
        // instance, current and future. Prefix-match: pass class+method,
        // patcher resolves the mangled symbol with the ABI param sig.
        MP_ROUTE_PAIR("IONDRVFramebuffer", "IOFramebuffer",
                      "enableController",
                      patchedEnableController, orgEnableController),
    };
    return mp_route_on_publish("IONDRVFramebuffer", kexts,
                               reqs, sizeof(reqs)/sizeof(*reqs));
}
```

That's the whole framework surface area for the IOService path. See
[`include/mos15_patcher.h`](include/mos15_patcher.h) for the full API.

## Public API

```c
// IOService publish — preferred for virtual methods.
// kext_bundle_ids is a NULL-terminated chain (primary first, fallbacks after).
int mp_route_on_publish(const char *class_name,
                        const char *const *kext_bundle_ids,
                        mp_route_request_t *reqs, size_t count);

// Auto-dispatch for static / non-virtual functions in a specific kext.
// Synchronous if loaded; queued if not (drained by an internal timer).
int mp_route_kext(const char *kext_bundle_id,
                  mp_route_request_t *reqs, size_t count);

// Lower-level: prologue patch at a known address.
int mp_route_addr(uint64_t target_addr, void *replacement, void **org);
```

### Route-construction macros (C++ only)

These remove the need to spell out Itanium C++ ABI parameter
signatures:

```cpp
// Most common — class+method only, patcher prefix-matches.
MP_ROUTE("IONDRVFramebuffer", "enableController", patched, org)

// Two routes (derived class + base class) sharing the same replacement.
MP_ROUTE_PAIR("IONDRVFramebuffer", "IOFramebuffer", "enableController",
              patched, org)

// When prefix is ambiguous (overloaded methods) — supply explicit ABI sig.
MP_ROUTE_PAIR_SIG("IONDRVFramebuffer", "IOFramebuffer",
                  "setGammaTable", "jjjPv",
                  patched, org)

// Full mangled-name escape hatch.
MP_ROUTE_EXACT("__ZN17IONDRVFramebuffer3fooEii", patched, org)
```

The ambiguity-detection logic in the symbol resolver logs ambiguous
prefix matches at kext load time, so consumers see the problem
immediately instead of mysteriously.

## How it works

### IOService publish notifications (`src/notify.cpp`)

`addMatchingNotification(gIOPublishNotification, ...)` fires the
instant macOS creates an instance of a matched class — for every
current and future instance. Our callback:

1. Reads instance+0 — the vtable pointer.
2. Allocates a RW vtable copy in heap (8 KB / 1024 slots — bigger
   than any IOKit class's virtual surface).
3. For each routed method: walks the
   `kext_bundle_ids` chain in order, calling
   `macho_find_symbol` (exact) or `macho_find_symbol_by_prefix`
   (E-delimited prefix match) until a match resolves.
4. Scans the copy's slots for the resolved address. Saves the
   original to `*org`, writes the replacement.
5. Atomic-writes the copy's address to instance+0. Only that
   instance is affected — other instances keep the OEM vtable.

Why per-instance: in-place writes to `__DATA_CONST` (where vtables
live on Sequoia) crash even with `CR0.WP` toggled. The instance
itself is on the C++ heap (RW). Writing 8 bytes there is trivial
and safe.

### Symbol lookup (`src/macho.cpp`)

Given a `kmod_info_t.address`, parse the `mach_header_64`. Walk
load commands to `LC_SYMTAB` and `LC_SEGMENT_64`. Read
`__LINKEDIT`'s `vmaddr` and `fileoff`. Compute:

```
symbols = linkedit.vmaddr + (symtab.symoff - linkedit.fileoff)
strings = linkedit.vmaddr + (symtab.stroff - linkedit.fileoff)
```

This formula works universally — standalone Mach-O (where
`hdr = linkedit.vmaddr - linkedit.fileoff` by algebraic identity)
AND KC-embedded kexts (where the kext's mach header has no relation
to KC-relative symtab offsets). One formula, no heuristics.

Prefix match (`macho_find_symbol_by_prefix`): scan the symbol table
for names starting with a given prefix. Returns the unique match,
or 0 with a kernel log line on ambiguity (caller should disambiguate
via `MP_ROUTE_*_SIG` or `MP_ROUTE_EXACT`).

### Function patching (`src/patch.cpp`)

On x86_64, write a 14-byte absolute indirect JMP
(`ff 25 00 00 00 00 <8-byte addr>`) at the target's prologue. A
small length disassembler computes how many original bytes to
displace (must be ≥ 14). Displaced bytes go to a slot in an in-kext
`__TEXT` trampoline pool. RIP-relative displacements in the
displaced bytes are rewritten by `delta = original_addr - tramp_addr`.
A 14-byte JMP back to `target+N` is appended. `*org` becomes the
trampoline.

Writes to kernel `__TEXT` use a CR0 WP-bit toggle (interrupts
disabled, `CR0 &= ~WP`, memcpy, restore). The trampoline pool sits
inside our own kext's `__TEXT,__cstring` section — it comes up
executable from the kext loader with no `vm_protect` upgrade needed
(`vm_protect` returns `kr=2` on recent kernels for kernel pages
anyway).

### Diagnostics

Per-class status is published as ioreg properties on the matched
IOService instance:

- `MPMethodsHooked`, `MPMethodsTotal`, `MPMethodsMissing`
- `MPMethodGaps` — array of mangled symbol pairs that didn't resolve
- `MPStatus` — compact per-route status string (`P` = primary
  patched, `F` = fallback patched, `u/f` = resolved-but-slot-taken,
  `X` = unresolved)
- `MPRoutesPatched`, `MPRoutesRedundant`, `MPRoutesUnresolved`

Read with `ioreg -l | grep MP` from userspace. This bypasses the
kernel-log buffer drops we hit during the publish callback, which
truncated diagnostic lines mid-write.

## Source layout

| File | LOC | Role |
|---|---|---|
| `include/mos15_patcher.h` | 182 | Public API + `MP_ROUTE_*` macros |
| `src/start.cpp` | 250 | Kext entry, public API impl, pending-route drain timer |
| `src/notify.cpp` | 300 | IOService publish-notification subscription + per-instance vtable swap |
| `src/vtable.cpp` | 97 | Vtable-slot scan + replacement |
| `src/macho.cpp` | 187 | Mach-O symbol lookup (Boot KC + System KC + Aux KC) |
| `src/patch.cpp` | 338 | x86_64 14-byte absolute-JMP prologue patching + trampolines |
| `src/kmod_info.c` | 8 | `kmod_info_t` for the kext loader |
| `src/*.hpp` | 154 | Internal headers (notify, macho, patch, vtable) |
| `Info.plist` | — | Bundle metadata; matches on `IOResources`/`IOKit` (no `IOPCIPrimaryMatch`) |

## Build

```bash
KERN_SDK=/path/to/MacKernelSDK ./build.sh
# Output: build/mos15-patcher.kext
```

Prerequisites:

- **Xcode** with the macOS SDK (`xcrun -sdk macosx`)
- **MacKernelSDK** ([acidanthera/MacKernelSDK](https://github.com/acidanthera/MacKernelSDK))
- `KERN_SDK` env var pointing at the `MacKernelSDK` root

`build.sh` cross-compiles for `x86_64-apple-macos10.15` regardless
of the host arch (so it builds on Apple Silicon Macs too — no SIP
relax, no `kextutil`, just `clang -mkernel`). The default
`KERN_SDK` is `../docker-macos/kexts/deps/MacKernelSDK`.

## How it's consumed

[mos-opencore](https://github.com/MattJackson/mos-opencore)'s build
script copies `build/mos15-patcher.kext` into `EFI/OC/Kexts/`
inside `OpenCore.img`. OpenCore injects it into Boot KC at boot
based on the `Kernel.Add` entry in `config.plist`. The kext entry
in `config.plist` lists this kext **before**
`QEMUDisplayPatcher.kext` so the framework is available when the
consumer calls `mp_route_on_publish`.

[mos-docker](https://github.com/MattJackson/mos-docker) bind-mounts
`OpenCore.img` into the runtime container as a SATA drive; QEMU
boots it. From the kext's perspective, none of that matters — it
runs inside a stock Sequoia kernel like any other kext.

## Compatibility with Lilu plugins

**No compatibility layer.** Lilu's API is a rich C++ hierarchy
(`KernelPatcher`, `KextInfo`, `PluginConfiguration`, `IOSubclasser`…).
We don't reimplement any of it. Plugins must port to the much
smaller `mp_route_*` surface — usually a few-line change per route,
plus dropping `PluginConfiguration` boilerplate (boot-arg handling
becomes the plugin's own concern).

## Status

**v0.5 — usable but barebones.** The framework is **runtime-proven**
end-to-end on Sequoia in our VM environment:

- 24/24 IOFramebuffer methods hooked across `IONDRVFramebuffer` +
  `IOFramebuffer` base, 0 gaps, in
  [QEMUDisplayPatcher](https://github.com/MattJackson/mos-docker/tree/main/kexts/QEMUDisplayPatcher)
- iMac20,1 EDID injection delivered intact through 2-block
  `getDDCBlock`
- 7/8 advertised display modes surface in CoreGraphics (the 8th is
  blocked by an upstream `mos-qemu` limit, not us)
- Vtable swap fires on every `IONDRVFramebuffer` instance via the
  publish-notification path; persistent across container restarts
- Ioreg diagnostic properties expose per-route status for fast
  debugging

**Not yet:**

- Hardened against multiple consumers — no consumer-isolation if
  two kexts both call `mp_route_on_publish` on the same class
- Tested across macOS versions — only Sequoia 15.7.5 (build 24G624)
- ARM64 / Apple Silicon support — x86_64 only (the prologue patcher
  is x86 ISA-specific)
- API frozen — `MP_ROUTE_*` macros and `mp_route_*` signatures may
  change in v1.0 once we have a second consumer to validate the
  API shape

## Further reading

- [mos-docs whitepaper 08 — IOService publish vs vtable patch](https://github.com/MattJackson/mos-docs/blob/main/whitepapers/08-ioservice-publish-vs-vtable-patch.md)
  — the full design narrative
- [mos-docs reference — kext bundle list](https://github.com/MattJackson/mos-docs/blob/main/reference/kext-bundle-list.md)
  — every kext mos ships, load order, bundle IDs
- [mos-docs architecture — component map](https://github.com/MattJackson/mos-docs/blob/main/architecture/01-component-map.md)
  — where this kext sits in the six-repo dependency graph
- [mos-docs overview — project status](https://github.com/MattJackson/mos-docs/blob/main/overview/project-status.md)
  — current milestone, what works today

## Contributing

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for
guidelines, and [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) for the code
of conduct.

## Security

To report a vulnerability, see [SECURITY.md](SECURITY.md).

## Changelog

Notable changes are recorded in [CHANGELOG.md](CHANGELOG.md).

## License

[MIT](LICENSE) © 2026 Matthew Jackson.
