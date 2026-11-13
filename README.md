# Seam Contract

[![GitHub stars](https://img.shields.io/github/stars/olakunlevpn/olakunlevpn-seam-contract?style=flat-square)](https://github.com/olakunlevpn/olakunlevpn-seam-contract/stargazers)
[![License: MIT](https://img.shields.io/badge/License-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)

Establishes **one** source of truth for the wire shape between client and server, derives both sides from it, and proves no hand-written copy survives.

Turns a field-name mismatch from a bug you audit into a bug that cannot compile.

This is an **agent skill** that works with Claude Code, Cursor, Cline, Gemini CLI and 40+ other AI coding agents.

Standalone — depends on no other skill.

## Installation

```bash
npx skills add olakunlevpn/olakunlevpn-seam-contract
```

## The Problem

The server sends `first_name`. The component reads `firstName`. Both files review clean, because nobody wrote a bug — two people wrote the same shape twice and the copies diverged.

Every variant has the same root cause: **the wire shape is written down in two places.** A renamed field, a changed type, a flattened object, a bare array that became paginated, a value that can now be null. Auditing for the divergence works, and is strictly weaker than not having two places.

## What It Does

**Phase 0 — Survey the mechanism.** Determines what this stack can already enforce, across three tiers: the shape cannot drift (shared types, inferred RPC, a schema both sides derive from, one runtime schema imported by both); the shape is checked at build or runtime (generated types, contract tests); or the shape is only written down. Tier 3 is not a contract, it is a hope — and if it is genuinely the only option, that gets recorded as a risk rather than glossed. It also refuses to add a second mechanism where one exists.

**Phase 1 — Establish the single source of truth.** One named file defining the request shape, the success shape, and every error shape — with nullability in the types, formats for dates and money, and one casing convention. Both sides shown deriving from it with `file:line`.

**Phase 2 — The duplication scan.** Five mandatory grep categories with raw output pasted: hand-written mirrors, inline casts and `any`, parallel serializers on the server, stale generated files, and suppressions at the seam. Every hit triaged individually — never cleared by category, because "those are all view models" is exactly how the stale copy survives. **No pasted output means the line reports `UNPROVEN`, never "none".**

**Phase 3 — The typecheck gate.** Typecheck, build, fresh codegen — and then the proof that matters: **rename a field in the source of truth and confirm the typecheck fails on both sides**, then revert. Code that merely compiles proves nothing; a hand-written type that happens to be correct today also compiles. If breaking the source does not break the check, some side is not deriving from it and the duplicate has not been found.

**Phase 4 — The gate.** Countable. Client fields derived must equal fields consumed. Duplicates removed must equal duplicates found. Scan categories pasted must be five. Suppressions at the seam must be zero. The rename proof must have failed. `1.0` is the only clean pass.

## Use

```
/seam-contract orders API
connect the dashboard to the metrics endpoint
wire this component up to the API
```

It also auto-triggers when you are about to hand-write a type mirroring a response you do not own, and on the symptoms of a shape that already drifted — a field rendering `undefined`, a `.map` throwing on a paginated response.

## What It Will Not Do

- Pass a contract without verifying both ends derive from the same source
- Tolerate hand-written interface mirrors next to generated types
- Clear duplication hits by category without inspecting each one
- Reach 1.0 without running the rename mutation proof

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
