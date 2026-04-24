# Contributing to mos-patcher

`mos-patcher` is a minimal kernel-extension hook framework for macOS 15 (Sequoia). It replaces Lilu for the mos suite's needs. Contributions are welcome.

## Build

Prerequisites:

- Xcode with macOS SDK 15+ (`xcrun -sdk macosx`)
- A checkout of `MacKernelSDK` (typically under a sibling `docker-macos/kexts/deps/MacKernelSDK`)
- `KERN_SDK` environment variable pointing at the `MacKernelSDK` root

Then:

```sh
KERN_SDK=/path/to/MacKernelSDK ./build.sh
```

Output lands at `build/mos15-patcher.kext`.

## Style

Match the existing `src/`:

- 4-space indent, no tabs
- K&R braces on functions
- C++17 where C++ is used (`.cpp`, `.hpp`)
- Public API identifiers use the `mp_` prefix (`mp_route_kext`, `mp_route_on_publish`, etc.) — see `include/mos15_patcher.h`

## Testing

This project exposes runtime diagnostics via ioreg properties (`MPMethodsHooked`, `MPStatus`, `MPRoutesPatched`, etc.) rather than relying on kernel-log buffers, which Sequoia drops under contention. Any new route or hook should surface at least one such property so consumers can verify installation.

## Architectural constraints that contributions MUST preserve

- **Per-instance vtable swap, not in-place `__DATA_CONST` patching.** Sequoia's `__DATA_CONST` page protection survives the CR0.WP toggle that worked on older macOS. The per-instance swap (allocate RW copy, mirror, patch our slots, write the copy's address into the instance's vtable slot) is the safe path. Do not introduce in-place patches.
- **`IOService::addMatchingNotification` on `gIOPublishNotification`** is how we observe new instances. Hooks install in the publish callback.
- **Targeted kext-bundle lookups.** `mp_route_on_publish` takes a NULL-terminated array of kext bundle IDs — primary first, fallbacks after. Callers must pass the array explicitly; never scan the full kmod chain.
- **Prefix-match symbol resolution.** `macho_find_symbol_by_prefix` exists so consumers don't have to hand-mangle Itanium C++ ABI parameter signatures. Prefer `MP_ROUTE_PAIR`/`MP_ROUTE_PAIR_SIG` macros over raw mangled names.

## Commit discipline

- One logical change per commit.
- Commit subject: imperative, under 70 characters.
- Body: explain the *why*. Reference ioreg properties or tests that verify the change.
- No personal paths, no credentials, no internal domain references.
- `Co-Authored-By:` trailers are welcome for pair-programming / agent-assisted work.

## License

AGPL-3.0. See `LICENSE`. Network use counts as distribution.
