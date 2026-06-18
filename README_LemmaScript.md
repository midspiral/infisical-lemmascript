# Infisical — Verified with LemmaScript

[![LemmaScript verified](https://img.shields.io/github/actions/workflow/status/midspiral/infisical-lemmascript/lemmascript.yml?branch=lemmascript&label=LemmaScript%20verified)](https://github.com/midspiral/infisical-lemmascript/actions/workflows/lemmascript.yml)

This is a fork of [Infisical/infisical](https://github.com/Infisical/infisical) with
formal verification of the **permission-boundary glob containment** check using
[LemmaScript](https://github.com/midspiral/LemmaScript).
[View as diff](https://github.com/midspiral/infisical-lemmascript/compare/main..lemmascript).

LemmaScript annotates TypeScript directly with `//@` specifications and generates
Dafny for verification. The original `glob-subset.ts` source is **unchanged** —
only annotation comments are added; the matching semantics and the soundness
proof live in the generated/hand-written `glob-subset.dfy`.

## Why this function

Infisical is a secrets-management platform, so its sharpest correctness question
is access control. When a user or machine identity delegates permissions to a
child role, `validatePermissionBoundary` (`backend/src/lib/casl/boundary.ts`) must
reject any grant that is *broader* than what the grantor holds. For secret paths
expressed as globs (`/prod/**`, `/apps/*`), that reduces to one set-containment
question, answered by `isGlobSubsetOfGlob` in
[`backend/src/lib/casl/glob-subset.ts`](backend/src/lib/casl/glob-subset.ts):

> **is every secret path matched by `subsetGlob` also matched by `parentGlob`?**

A false *yes* here is a privilege escalation — a child role that can read paths
the parent never could. The module's own `glob-subset.test.ts` documents a real
such bug that a previous implementation had (the old picomatch fallback let
`/@(public|shared)/*` appear to "contain" `/public/**`). The current
implementation is deliberately **sound, not complete**: it reasons only about
globs whose segments are literals, `*`, or `**`, and returns `false` for anything
else. So the only property worth proving is the security-relevant one — **it never
over-approves a containment claim**.

## Setup

**Prerequisites:** [Dafny](https://github.com/dafny-lang/dafny) ≥ 4.11, Node.js ≥ 18.

```sh
git clone https://github.com/midspiral/LemmaScript.git ../LemmaScript
cd ../LemmaScript/tools && npm install
```

**Re-run the verification** (from this repo's root):

```sh
npx tsx ../LemmaScript/tools/src/lsc.ts regen --backend=dafny \
  backend/src/lib/casl/glob-subset.ts
```

Expected: `Dafny program verifier finished with 12 verified, 0 errors`.

The production behavior is untouched — the existing unit tests still pass:

```sh
cd backend && npm run test:unit -- glob-subset
```

## What's Verified

### `segmentMatch` — glob set-containment is sound

`segmentMatch` is the recursive core of `isGlobSubsetOfGlob`: it compares two
parsed segment sequences (`literal` / `*` / `**`) and decides containment.

The proven theorem, over the matching semantics `glmatch` defined in
`glob-subset.dfy`:

> `segMatchSpec(parent, subset, pi, si)` ⟹
> ∀ concrete path. `glmatch(subset, si, path)` ⟹ `glmatch(parent, pi, path)`

i.e. if the algorithm returns `true`, then **every concrete secret path matched
by the `subset` glob suffix is also matched by the `parent` glob suffix** — the
exact non-escalation guarantee. `glmatch` is the standard segment-glob semantics:
`**` matches any number (≥ 0) of path segments, `*` matches exactly one, a literal
matches an identical segment.

Termination of the recursion is proven too (`decreases (|parent| − pi) +
(|subset| − si)`), as is the loop invariant for the "every remaining parent
segment is `**`" case.

### How the proof is structured

`segmentMatch` contains a loop, so LemmaScript emits it as a Dafny `method` —
which can't be named in specifications and can't carry in-body proof hints. The
proof therefore uses a **refinement decomposition**, all machine-checked:

1. **`segmentMatch` ≡ a pure spec.** The method carries
   `//@ ensures \result == segMatchSpec(parent, subset, pi, si)`, where
   `segMatchSpec` is a pure Dafny mirror of the algorithm (hand-written in the
   `.dfy`). Dafny discharges the equality from the recursive postconditions plus
   two loop invariants — pure boolean refinement, no semantic reasoning.
2. **The pure spec is sound.** `SegMatchSpecSound` proves `segMatchSpec ⟹ ⟨the
   containment property above⟩` by induction over `glmatch`, with two helper
   lemmas (`EmptyMatchesAllGlobstar`, `GlobstarAbsorbSubset`).
3. **Compose.** `SegmentMatchSound` takes the method's certified result and the
   soundness lemma and concludes the end-to-end guarantee.

### Trust boundary

The guarantee holds relative to two named assumptions, stated inline in
`glob-subset.dfy`:

- **`glmatch` faithfully models picomatch** on the literal/`*`/`**` fragment.
  picomatch is the matcher Infisical actually runs; `glob-subset.ts` only *decides
  containment* over parsed segments. That fragment is the **only** input
  `parseSegments` accepts — every other metacharacter (`?`, `[…]`, `{…}`, extglob,
  intra-segment `*`) makes it return `null`, so containment is conservatively
  `false`. The trust surface is exactly this one predicate.
- **`parseSegments` is outside the model** (`//@ extern`): it wraps a regular
  expression, which LemmaScript does not model. It is the regex/parse boundary;
  `segmentMatch`, which carries all the containment logic, is fully verified.

## File Structure

```
backend/src/lib/casl/
  glob-subset.ts        ← Original source + //@ annotations on segmentMatch (only)
  glob-subset.dfy       ← Proof: glmatch semantics, segMatchSpec, soundness lemmas
                          (hand-written, additions-only over the generated base)
  glob-subset.dfy.gen   ← Generated Dafny (regeneratable from the .ts)
  glob-subset.test.ts   ← Existing unit tests (unchanged; still pass)
LS_CANDIDATES.md        ← Survey of further verification targets, tiered by impact
```

## Not yet verified

[`LS_CANDIDATES.md`](LS_CANDIDATES.md) surveys further targets and the natural
next steps. In short:

- **`boundary.ts` operator algebra** (`areOperatorsDisjoint`, `isOperatorsASubset`)
  — the layer above this one, comparing CASL `$EQ`/`$NEQ`/`$IN`/`$GLOB`
  constraints, with picomatch lifted as a contract. The same refinement pattern
  applies to its loop-bearing helpers.
- **`literalPrefix` / `haveDisjointLiteralPrefixes`** — the deny-rule disjointness
  heuristic. Currently blocked: their bodies compare a string element to a
  single-character string literal (`glob[i] === "*"`), which LemmaScript does not
  yet translate (Dafny string-indexing yields a `char`, not a length-1 string).
