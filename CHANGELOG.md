# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog 1.1](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.5.0] — first usable cut

A minimal kernel-function-hook framework for macOS. Built as a Lilu replacement
for the mos suite.

### Added

- 24/24 IOFramebuffer methods hooked end-to-end, runtime-proven on Sequoia
  15.7.5 inside QEMU/KVM.
- `MP_ROUTE_PAIR` API — consumers don't hand-mangle Itanium C++ ABI parameter
  signatures.
- Per-instance vtable swap via `IOService::addMatchingNotification` — Sequoia-safe
  (no `__DATA_CONST` writes).
- Universal `__LINKEDIT`-relative symbol resolution — works across Boot KC,
  System KC, and Aux KC with one formula.
- Ioreg diagnostic properties — per-route status, hook coverage, gaps; bypass
  kernel log buffer drops.

### Status

Usable but barebones. Production hardening incomplete — see README for known
limitations (multi-consumer isolation, multiple macOS versions, ARM64).

### License

AGPL-3.0. Network use counts as distribution.

[Unreleased]: https://github.com/MattJackson/mos-patcher/compare/v0.5.0...HEAD
[0.5.0]: https://github.com/MattJackson/mos-patcher/releases/tag/v0.5.0
