# Security Policy

## Reporting a Vulnerability

Please report suspected vulnerabilities privately via GitHub Security Advisories:

<https://github.com/MattJackson/mos-patcher/security/advisories/new>

Do **not** open a public issue for security-relevant reports. We'll acknowledge
receipt, investigate, and coordinate a fix + disclosure timeline with you.

## Supported Versions

| Version | Supported | Platform                                 |
| ------- | --------- | ---------------------------------------- |
| 0.5.x   | Yes       | macOS 15.7.5 (Sequoia), x86_64           |
| < 0.5   | No        | —                                        |

Only the most recent 0.5.x release receives security fixes. ARM64 / Apple
Silicon, other macOS versions, and older releases are out of scope.

## Threat Model Note

mos-patcher is a **kernel extension**. It executes in ring 0, writes to kernel
`__TEXT` pages (prologue patching via CR0 WP-bit toggle), allocates and swaps
per-instance vtables, and resolves Mach-O symbols by parsing live kext
headers. That's a large elevated-privilege attack surface.

Reports we take especially seriously:

- **Sandbox escape / privilege escalation** — anything that lets an unprivileged
  userspace caller reach an exploitable path through our ioreg properties, our
  hooked methods, or our symbol-resolution code.
- **Memory corruption** in the prologue patcher, vtable-copy allocator, or
  Mach-O parser (malformed kext images, edge cases in `__LINKEDIT` offset
  arithmetic, trampoline RIP-rel rewriting, etc.).
- **Hook-chain hijack** — any way for a malicious kext to force our
  `mp_route_*` bookkeeping into calling attacker-controlled replacements on
  behalf of another consumer.

If you're not sure whether something qualifies, report it — we'd rather
triage false positives than miss real issues.
