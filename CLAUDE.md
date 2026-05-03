# mos15-patcher

Lilu replacement: ~600-line kernel hook framework for the **mos**
project. Per-instance vtable swap via IOService publish notifications
(in-place `__DATA_CONST` patches crash on Sequoia).

This repo is one of several mos satellites. **Project context, current
state, mental model, build/deploy, and don't-do-this list all live in
the parent project's CLAUDE.md and library:**

- Airport map + working mode + current state: `../mos/CLAUDE.md`
- Standing rules + closed-milestone facts: `../mos/memory/MEMORY.md`
- Patcher-specific design notes: `../mos/memory/project_mos15_patcher.md`
  and `../mos/memory/project_mos_patcher_metal_fallback_gap.md`

## Scope of this repo

Kernel-side hooks for non-rendering kexts (rendering goes through
the Vulkan/lavapipe path in `libapplegfx-vulkan`, untouched).

Currently dormant during the M5 push — most M5 work is host-side.

## Conventions

- **Per-instance vtable swap only.** Never patch `__DATA_CONST` in
  place. CR0.WP toggling crashes Sequoia. See
  `../mos/memory/feedback_per_instance_vtable_swap.md`.
- **Lilu hook patterns:** trampolines safe, `routeMultiple` aborts the
  whole batch on one failure. See
  `../mos/memory/feedback_lilu_route_safety.md`.
- **No `Co-Authored-By: Claude`** trailers; see
  `../mos/memory/feedback_no_ai_attribution_in_commits.md`.
