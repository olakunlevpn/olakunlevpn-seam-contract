---
name: olakunlevpn-seam-contract
description: Use when client and server code are about to meet, or already met by hand -- adding an endpoint a client will consume, writing a component that reads API data, "connect the frontend to X", "wire this up to the API", or whenever a type or interface is hand-written to mirror a response shape. Establishes ONE source of truth for the wire shape and derives both sides from it, so the client and server cannot drift. Runs a mandatory duplication scan with pasted grep output to find every hand-written mirror of a server shape, and a typecheck gate. Turns field-name and type mismatches from a bug you audit into a bug that cannot compile. Standalone -- depends on no other skill. Gate is countable and passes only at 1.0. Do NOT use for server-internal refactors with no client, or pure styling work.
---

# Seam Contract — Make the Mismatch Impossible

There is a class of bug that costs more time than it has any right to: the server sends `first_name`, the component reads `firstName`, and both files review clean. Nobody wrote a bug. Two people wrote the same shape twice, and the copies diverged.

Every mismatch of this kind — a renamed field, a changed type, a nested object flattened, a bare array that became paginated, a value that can now be null — has the same root cause. **The wire shape is written down in two places.** Auditing for the divergence afterwards works, and it is strictly weaker than not having two places.

This skill does the stronger thing. It finds or establishes one source of truth for the shape, derives both ends from it, and then proves no hand-written copy survives anywhere. After it runs, a renamed field is a compile error instead of an afternoon.

**SECURITY: Never write credentials, keys, tokens or connection strings into a schema, type file, or generated client. Never paste a `.env` value into pasted command output — redact it. Report only whether a secret is handled correctly (YES/NO), never the value.**

## Standalone by Design

This skill carries everything it needs. It does not call, summon, require, or degrade without any other skill. The mechanism survey, the duplication scan, the derivation rules and the gate are all defined in this file. It runs identically in any agent, in any repo, in any language.

## When to Use (Auto-Trigger)

Load this skill automatically. Do not wait to be asked. Run it when ANY of these occur:

- An endpoint is being written that a client will consume.
- A component, page, or client module is being written that reads server data.
- The user says "connect", "wire up", "hook up", "integrate" about a frontend and a backend.
- You are about to write a type, interface, struct, or DTO that describes a response you do not own.
- You notice an existing hand-written type that mirrors a server response shape.
- A response shape is changing — a field renamed, added, removed, or retyped.
- A field renders as `undefined`, a list renders empty, or a `.map` throws on a shape that "should" be an array — the symptoms of a shape that already drifted.

**How to auto-trigger:**
1. Announce: "Establishing the seam contract for `<feature or endpoint>` — one source of truth, then the duplication scan."
2. Walk Phase 0 → 4 in order.
3. Run the duplication scan and paste its output. No pasted output = no clean report.
4. Apply the Gate. Below 1.0 the seam is not safe, and say so plainly.

## When This Is The Wrong Skill

**If nothing crosses a process boundary, stop.** A refactor entirely inside the server, or entirely inside the client, has no seam. Say so and move on rather than manufacturing a contract for a boundary that does not exist.

## Golden Rules

1. **One source of truth, always.** Exactly one artifact defines the wire shape. Everything else is derived from it, generated from it, or imports it. Two authoritative copies is the bug.
2. **Derive, never retype.** Hand-writing a type that mirrors a response is the act this skill exists to prevent — even when you are copying it accurately today.
3. **Prevention over detection.** A mismatch that cannot compile beats a mismatch you find by reading. Where the stack can enforce the shape, enforcing it is the job; auditing it is the fallback.
4. **Prove the absence of copies.** "I checked, there are no duplicates" is a claim. A pasted grep with its output is evidence. Only one of those counts.
5. **The envelope is part of the shape.** Pagination wrappers, `data` keys, status envelopes, error bodies. A contract covering only the happy payload leaves the most common break uncovered.
6. **Errors have shapes too.** Every non-2xx body the server can emit is part of the contract. An error shape nobody defined is an error path nobody handled.
7. **1.0 or the seam is not safe.** See the Gate. Counts are countable; rounding up is banned.

---

## Phase 0 — Survey the Mechanism

Before establishing anything, find out what this stack can already enforce. The strongest available mechanism wins; do not introduce a weaker one alongside it.

Read the project and determine which of these it has, or could have cheaply:

```
TIER 1 — the shape cannot drift (a mismatch does not compile)
  Shared types imported by both sides (a monorepo, one TypeScript package)
  RPC with inferred types end to end (types ARE the contract, no codegen)
  A schema both sides derive from, with generated clients
    (GraphQL schema, OpenAPI + generator, protobuf/gRPC)
  A runtime schema shared by both sides (one Zod/Pydantic/Valibot schema
    imported by the server for validation and by the client for parsing)

TIER 2 — the shape is checked, but at runtime or at build
  Generated types from a live endpoint or spec, regenerated in CI
  A serializer/resource layer on the server that is the single definition,
    with client types generated from it
  Contract tests: the consumer records what it needs, the provider verifies it

TIER 3 — the shape is written down only
  A spec document, a design artifact, a comment
  Hand-written types on both sides that a human keeps in sync
```

**State which tier this project is at, and which one this feature will use.** Tier 3 is not a contract; it is a hope. If Tier 3 is genuinely the only option available here, say so explicitly and record it as a known risk rather than proceeding as though the seam is safe.

**Do not introduce a new mechanism when one exists.** If the project already generates types from an OpenAPI spec, the answer for this feature is that spec — not a hand-written interface next to it, and not a second mechanism. Two mechanisms is two sources of truth with extra steps.

**Exit condition:** The project's current tier named, the mechanism this feature will use named, and — if Tier 3 — the risk stated.

---

## Phase 1 — Establish the Single Source of Truth

Point at the one artifact that defines this shape. It must be a specific file.

```
SOURCE OF TRUTH:  <file path>
MECHANISM:        <shared type / inferred RPC / schema + codegen / runtime schema>
DEFINES:          the request shape, the success shape, every error shape
CONSUMED BY:      how the server gets its types from it (file:line)
                  how the client gets its types from it (file:line)
REGENERATION:     if codegen is involved — the exact command, and whether CI runs it
```

If the source of truth does not exist yet, create it — that is the substance of this phase. Define the full shape once:

- **Request:** every field, its type, whether required.
- **Success:** the status code and every field with its type, including the envelope.
- **Errors:** every non-2xx status this endpoint can emit, and the body shape of each.
- **Nullability:** part of every type, not an afterthought. `avatar_url: string | null` is a different contract from `avatar_url: string`, and the difference is whether the client crashes.
- **Formats for dates and money:** ISO 8601, epoch, minor units, a precision. "A date" is not a type.
- **One casing convention**, stated once and applied everywhere. This one rule prevents the most common seam bug in existence.

**Exit condition:** The source of truth exists at a named path, defines request, success and every error shape, and both sides are shown deriving from it with file:line.

---

## Phase 2 — The Duplication Scan (mandatory — run it, paste the output)

A single source of truth is only real if no copy survives. Noticing copies while reading is not a scan — you will skim past the one that matters, because a stale duplicate looks exactly like a correct one.

**The rule that makes this work: no pasted command output means the report line is `UNPROVEN`, never "none".**

Run these across the client and server code for this feature, and paste each command with its raw output — including the empty results:

```
1. HAND-WRITTEN MIRRORS
   Search for type/interface/struct/class declarations whose fields mirror a
   response shape. Start from the field names in your source of truth:
     grep -rn -E 'interface |type .*=|class .*Response|struct ' <client paths>
   Then, for each response field name, find every declaration that names it:
     grep -rn '<field_name>' <client paths>

2. INLINE SHAPE ASSERTIONS
   grep -rn -E 'as [A-Z][A-Za-z]*(\[\])?|<[A-Z][A-Za-z]*(\[\])?>|: *any' <client paths>
   Casts and `any` are how a mismatch gets silenced instead of caught.

3. PARALLEL SERIALIZERS
   On the server, more than one place composing the same response shape —
   a controller building it inline while a resource/serializer also defines it:
     grep -rn -E 'toArray|serialize|to_json|Marshal|jsonify|JsonResponse' <server paths>

4. STALE GENERATED FILES
   If codegen is involved, confirm the generated output matches the schema NOW:
     <the project's generate command>, then `git diff --stat` on the output.
   A generated file with uncommitted changes after regeneration was stale.

5. DISABLED SAFETY AT THE SEAM
   grep -rn -E '@ts-ignore|@ts-expect-error|eslint-disable|# type: ignore|
                phpstan-ignore|psalm-suppress' <client paths> <server paths>
   Suppression at a seam is usually a mismatch someone silenced.
```

Adjust the patterns to the languages in the repo, but run all five categories.

**Triage every hit individually.** A hit is either:
- **A duplicate** — it defines a shape the source of truth already defines. Replace it with a derivation from the source, or delete it. This is the work.
- **Legitimate** — a client-only view model that genuinely is not the wire shape, a test fixture, a type for a third-party response you do not own. State why, per hit.

**No hit is cleared by category.** Read each one. "Those are all view models" is exactly how the stale duplicate survives.

**Exit condition:** All five categories run with output pasted, every hit triaged individually, every duplicate removed or derived.

---

## Phase 3 — The Typecheck Gate (mandatory — run it, paste the output)

A contract that has not been checked by a machine is a contract you believe in. Run the project's real commands and paste the real output:

```
1. TYPECHECK      tsc --noEmit | phpstan | mypy | go vet | cargo check ...
2. BUILD          the project's real build command
3. CODEGEN FRESH  if generated: regenerate, then confirm zero diff
4. THE PROOF      rename one field in the source of truth and confirm the
                  typecheck FAILS on both sides, then revert it
```

**Step 4 is the one that matters and the one that gets skipped.** A contract that compiles proves nothing on its own — a hand-written type that happens to be correct today also compiles. The only proof that the seam is actually wired to the source of truth is that breaking the source breaks the check. If the rename does not fail the typecheck, some side is not deriving from the source, and you have not found the duplicate yet.

Revert the rename before finishing. Confirm the revert in the report.

**Rules:**
- Paste actual commands and actual output. Summarizing the output is not the output.
- If a command does not exist in this project, say so explicitly — "no typecheck configured" — and record it. A missing tool is a stated fact, not a silent skip.
- A failing typecheck at the seam is the bug this skill exists for. Fix it; do not suppress it. Adding `any`, a cast, or an ignore comment to make this gate pass is banned.

**Exit condition:** Typecheck and build run with output pasted; the rename proof observed failing and then reverted.

---

## Phase 4 — The Gate

Countable. Do the arithmetic, then apply it.

```
COUNTS
  Fields in the source of truth:       N
  Fields consumed by the client:       N
  Client fields derived from source:   N   <- must equal fields consumed
  Error statuses defined:              N
  Error statuses with a client branch: N   <- must equal statuses defined
  Duplicate shapes found:              N
  Duplicates removed or derived:       N   <- must equal duplicates found
  Scan categories run and pasted:      N   <- must be 5
  Casts / any / ignores at the seam:   N   <- must be 0
  Rename proof failed the typecheck:   YES | NO   <- must be YES
```

- **`1.0`** — one named source of truth, every consumed field derived from it, every error status branched, zero surviving duplicates, all five scan categories pasted, zero suppressions at the seam, and the rename proof observed failing. **SEAM SAFE.** The only clean pass.

- **`0.95 – 0.99`** — **CONDITIONAL.** Permitted only when every gap is named: which fields are still hand-written, which duplicates remain and why, which command could not run. A conditional result may not be described as safe or drift-proof, and may not be summarized in a way that hides the gaps.

- **`< 0.95`** — **NOT SAFE.** Report what remains duplicated and what blocks certainty. You may not describe the seam as contract-backed, type-safe, or done under any wording.

**The score is computed, not chosen.** It comes from the counts. Never round up: 0.94 is not 0.95, and 0.99 is not 1.0.

**Any of these caps the score below 1.0 regardless of the counts:** a surviving hand-written mirror; a scan category not run; a cast, `any`, or ignore comment at the seam; a rename that did not fail the typecheck; an error status with no client branch; Tier 3 as the mechanism when a higher tier was available.

### The Leniency Check

Before writing the number:

```
1. THE ARITHMETIC   Does every count equal its target?
2. COMMANDS RUN     How many commands did I execute and paste? Zero pasted
                    output means the score cannot exceed 0.95.
3. THE PROOF        Did I actually break the source and watch both sides fail?
                    If I skipped it, I have not proven derivation — only that
                    the current code compiles.
4. WHAT I DID NOT   Every path I did not scan, every command I could not run.
   CHECK            MANDATORY even at 1.0.
5. THE HONEST ONE   Where is a stale copy most likely still hiding? Name a
                    specific file or directory. "Nowhere" is not acceptable.
```

---

## Banned (any one = the run does not pass)

- **Reporting a scan category as "none" without the pasted command output.**
- **Dismissing duplication hits by category** instead of triaging each one.
- **Hand-writing a type that mirrors a shape the source of truth already defines** — even accurately.
- **Adding a cast, `any`, `@ts-ignore`, or any suppression** to make the typecheck pass at a seam.
- **Skipping the rename proof**, or claiming derivation without having watched the break.
- **Leaving a second mechanism in place** alongside the source of truth.
- **Treating a written document as a contract** when the stack could have enforced it.
- **Defining only the success shape** and leaving error bodies undefined.
- **Reporting CONDITIONAL as safe**, or describing anything under 1.0 as drift-proof.
- **Rounding the count up.**

---

## Handoff

This skill is self-contained. When the seam is established, the natural next steps — none of which this skill invokes, waits on, or requires:

| Next | When |
|---|---|
| Build both sides against the source of truth | Gate is 1.0 |
| Remove the surviving duplicates, re-run the gate | Gate is conditional |
| Record the risk and proceed knowingly | Tier 3 was genuinely the only option |
| Verify the whole flow end to end | After both sides are built |

One source of truth. Derive, never retype. Prove no copy survived. Safe only at 1.0.
