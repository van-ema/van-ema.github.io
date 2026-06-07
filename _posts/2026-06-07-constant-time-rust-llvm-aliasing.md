---
layout: post
title: When Constant-Time Rust Meets LLVM Alias Metadata
date: 2026-06-07 20:01:10+0200
description: How Rust aliasing facts can let LLVM change fixed-load constant-time code into selected-address-load code.
tags: rust llvm cryptography constant-time
categories: Rust
---


Compilers rewrite programs all the time.

Rust code becomes MIR, MIR becomes LLVM IR, LLVM runs optimization passes, and
eventually machine code comes out. The usual contract is simple: the optimized
program should compute the same result as the original one, only faster or
smaller.

Constant-time cryptography asks for one more thing.

It is not enough that the program returns the right value. It also matters which
addresses the CPU touches while computing that value. Two executions can return
the same answer and still behave differently in the cache.

That creates an interesting question:

Can Rust code look constant-time at the source level, but compile into a binary
whose memory access pattern depends on a secret?

The investigation starts from a standard constant-time selection idiom: load
both candidate values first, then let the secret choose only between values
already in registers. Then aliasing is made relevant, the optimized assembly is
checked, and the timing behavior is measured.

The code and artifacts for the experiments live in the
[`ct-rust-verifier`](https://github.com/van-ema/ct-rust-verifier) repository.

## Starting From The Desired Shape

A common constant-time trick is to load both possible values, then select one of
the already-loaded values in registers:

```rust
fn ct_select_u8(choice: u8, a: u8, b: u8) -> u8 {
    let mask = 0u8.wrapping_sub(choice & 1);
    (a & !mask) | (b & mask)
}
```

The important property is not just "no branch". It is also "same memory access
pattern".

If `choice` is secret, this source shape is fine:

```rust
let av = *a;
let bv = *b;
ct_select_u8(choice, av, bv)
```

Both pointers are loaded every time. The secret only chooses between values that
are already in registers. At source level, this is the shape the experiment
wants to preserve.

But there is another shape that is not fine:

```text
selected = choice ? a : b
load *selected
```

That can be branchless too. On AArch64, for example, the address selection can
use `csel`, a conditional select instruction. But this version loads only one
address. If one address is cache-hot and the other is cache-cold, timing can
reveal the secret choice.

The source-level difference looks small, but the machine-level memory-access
pattern is not the same. The rest of the investigation is about whether LLVM can
legally move from the first shape to the second.

## Making Aliasing Matter

The test case uses this source-level pattern:

```rust
let av = *a;
*out = 0;
let bv = *b;
ct_select_u8(choice, av, bv)
```

There are two loads and one store. The store is there because it makes the
optimizer care about whether `out` can overlap with `a` or `b`. If overlap is
possible, the compiler has to be conservative around the store. If overlap is
ruled out, the compiler has more freedom.

This gives a simple strategy: keep the source access shape the same, but change
what aliasing facts are available to the optimizer.

The first version keeps raw pointers:

```rust
pub unsafe fn raw_interleaved_select(
    choice: u8,
    a: *const u8,
    b: *const u8,
    out: *mut u8,
) -> u8 {
    let av = *a;
    *out = 0;
    let bv = *b;
    ct_select_u8(choice, av, bv)
}
```

The second version first converts the raw pointers into Rust references:

```rust
pub unsafe fn unsafe_ref_interleaved_select(
    choice: u8,
    a: *const u8,
    b: *const u8,
    out: *mut u8,
) -> u8 {
    let a_ref = &*a;
    let b_ref = &*b;
    let out_ref = &mut *out;

    ref_interleaved_select(choice, a_ref, b_ref, out_ref)
}
```

The helper receives references and performs the same interleaved access pattern:

```rust
fn ref_interleaved_select(choice: u8, a: &u8, b: &u8, out: &mut u8) -> u8 {
    let av = *a;
    *out = 0;
    let bv = *b;
    ct_select_u8(choice, av, bv)
}
```

At the Rust source level, both versions still look like fixed memory access:
load `a`, store to `out`, load `b`, then select in registers.

At this point there is no result yet. Both Rust snippets still read like the
same fixed-access algorithm. The result appears only after optimization.

## First Result: The Assembly Shape Changes

The optimized assembly is where the first finding appears.

The raw-pointer version keeps both loads:

```asm
ldrb    w8, [x1]
strb    wzr, [x3]
ldrb    w9, [x2]
tst     w0, #0x1
csel    w0, w8, w9, eq
ret
```

The reference version selects the address first, then loads once:

```asm
tst     w0, #0x1
csel    x8, x1, x2, eq
ldrb    w0, [x8]
strb    wzr, [x3]
ret
```

This is the transform the experiment is looking for:

```text
load a; load b; select value
```

becomes:

```text
select address; load selected address
```

If `choice` is secret, this changes the side-channel behavior of the program.
The source-level constant-time argument says "both addresses are loaded"; the
binary does not do that in the reference-based version.

## Why Rust Semantics Matter

The assembly difference points back to Rust semantics.

The raw-pointer version and the reference version are not equivalent inputs to
the optimizer. Forming references tells the compiler more about the memory being
accessed.

When Rust lowers references to LLVM IR, it can attach facts such as:

- `noalias`
- `nonnull`
- `dereferenceable`
- `readonly`
- `writeonly`
- `alias.scope`

These facts are useful. They are part of why Rust can produce good optimized
code. They also mean that unsafe reference or slice construction can become part
of the constant-time story, even when the source code still looks branchless and
fixed-access.

For example, an `&mut T` carries a strong exclusivity promise. If LLVM knows
that `out` cannot alias `a` or `b`, then the store to `out` cannot affect the
loads from `a` or `b`. That gives the optimizer more room to rewrite the memory
operations.

From LLVM's point of view, the selected-address version is functionally
equivalent. It returns the same value. The optimization can be legal under the
ordinary language and IR rules.

The catch is that constant-time code has an extra rule: the memory access shape
must not depend on secrets.

LLVM is not optimizing for that rule unless the compilation model gives it a way
to represent and preserve it.

## Second Result: The Difference Is Measurable

The assembly result gives a concrete hypothesis: if the binary loads only the
selected address, then cache state should make the secret choice measurable.

The timing setup is simple:

- the fixed class always selects a cache-hot byte;
- the random class randomly selects the hot or cold byte;
- before each sample, a large buffer evicts cache state;
- only the hot pointer is warmed;
- a Welch t-test compares the two classes.

If the code always loads both pointers, both classes should do the same hot and
cold work. If the code loads only the selected pointer, the random class should
be slower.

That is exactly what the measurement shows.

| Target | Samples/class | Mean fixed | Mean random | Welch t | Result |
|---|---:|---:|---:|---:|---|
| `unsafe-ref-interleaved` | 10000 | 18.220 | 99.448 | -52.133 | distinguishable |
| `raw-interleaved` | 10000 | 144.341 | 155.492 | -0.887 | not distinguishable |
| `volatile` control | 10000 | 131.229 | 223.733 | -1.010 | not distinguishable |

Using the usual Dudect-style threshold of `|t| > 4.5`, the unsafe-reference
variant is clearly distinguishable. The raw-pointer and volatile controls are
not.

This is the second positive result. In this benchmark, alias-bearing reference
construction changes the optimized access pattern, and that change is
measurable.

## The Important Point About The Compiler

This result does not require LLVM to be obviously wrong. LLVM is allowed to use
the alias facts it receives, and the optimized function still computes the
right value.

The constant-time issue is about a property outside ordinary value semantics:

> In constant-time code, unsafe reference or slice construction can communicate
> alias facts that are invisible in a source-level constant-time review, and
> those facts can matter at the assembly level.

That is the security-relevant part. The compiler preserves the answer. It also
changes the way the answer is loaded from memory.

## Generalizing The Pattern

The minimal example explains one instance of the mechanism. The next question is
whether it depends on one carefully chosen function, or whether it appears
across a broader family of Rust constructs.

The taxonomy reproduces the same kind of access-shape change across several
source-facing categories:

- `&mut` exclusivity;
- shared references combined with a separate write path;
- mutable slice reconstruction;
- unchecked mutable indexing;
- integer-to-pointer round trips followed by reference formation;
- C/LLVM-style alias contracts such as `restrict`, `noalias`, and
  `alias.scope`.

The common thread is not a particular syntax trick, but the fact that the
optimizer receives more information about which pointers cannot overlap.

The strongest signal is the promise that "these pointers do not overlap".
Metadata such as `noalias` and `alias.scope` carries more weight than weaker
facts like `nonnull` or `readonly` on their own.

## Taking It To Real Code

After the taxonomy, the next step is to ask whether the same ingredients appear
in real Rust crypto and constant-time crates.

The early real-world scan covers:

- `subtle`
- `curve25519-dalek`
- `crypto-bigint`
- `base16ct`
- `base32ct`
- `base64ct`

The scan looks for unsafe reference or slice reconstruction, unchecked indexing,
raw pointer conversions, and similar patterns. For interesting source hits, the
analysis then moves down the stack:

```text
source pattern -> LLVM alias facts -> optimized assembly -> timing
```

The point is not to declare every unsafe pattern suspicious. The point is to
find cases where Rust source, LLVM metadata, and final assembly tell the same
story.

The scanner used for this is the
[`cross_layer_detector`](https://github.com/van-ema/ct-rust-verifier/tree/main/cross_layer_detector/detector).
Its current rules and output summaries are also checked in under
[`cross_layer_detector/results`](https://github.com/van-ema/ct-rust-verifier/tree/main/cross_layer_detector/results).

The strongest real-world-derived case comes from a `crypto-bigint` byte-slice
reconstruction pattern. In an extracted fixed-access selection shape, it
reproduces the same selected-load transform and timing leakage:

```text
primary run: abs(t) = 14.872
repeat run:  abs(t) = 18.925
```

This is the bridge from the minimal example to real code. A source pattern from
a cryptographic crate can reproduce the same alias-driven transform in a focused
benchmark, and the transform remains timing-visible. The extracted reproducer
lives under
[`real_world/extracted/phase2_cases`](https://github.com/van-ema/ct-rust-verifier/tree/main/real_world/extracted/phase2_cases),
with the classification notes in
[`real_world/results/confirmed_findings.md`](https://github.com/van-ema/ct-rust-verifier/blob/main/real_world/results/confirmed_findings.md).

## Scaling The Investigation

The scan then expands to 30 pinned Rust crypto and security crates on
x86_64 Linux.

The expanded corpus is pinned in
[`real_world/corpus/manifest.csv`](https://github.com/van-ema/ct-rust-verifier/blob/main/real_world/corpus/manifest.csv).

The detector finds many optimized-code patterns worth reviewing:

- 368 cross-layer transform rows;
- 34 selected-pointer-load rows;
- 17 unique selected-pointer-load crate/symbol pairs;
- many LLVM alias facts, including `noalias`, `alias.scope`, and `!noalias`.

This makes the cross-layer part of the work much more concrete. The detector is
not just finding unsafe source snippets. It is finding optimized code shapes
where source patterns, LLVM metadata, and assembly line up.

At this point the investigation has a useful queue: real optimized crate
artifacts containing the selected-load codegen shape, often with LLVM alias
facts nearby.

Manual triage then answers the security question:

```text
Where does the selector come from?
```

The highest-priority selected-load rows fall mostly into two buckets:

- `crypto-bigint` boxed integer and modular arithmetic paths;
- `elliptic-curve` development mock-curve code.

The reviewed `crypto-bigint` selected loads are driven by public length,
precision, or zero-padding decisions. For example, a loop over limbs may choose
between an actual limb and a static zero limb when one operand is shorter:

```rust
let &a = lhs.limbs.get(i).unwrap_or(&Limb::ZERO);
let &b = rhs.limbs.get(i).unwrap_or(&Limb::ZERO);
```

That can compile into a selected address load. Structurally, it matches the
pattern under investigation:

```asm
cmp     ...
csel    selected_ptr, real_limb, zero_limb, ...
ldr     value, [selected_ptr]
```

If the selector is public operand length, the selected-load shape is still
useful detector evidence. It shows the codegen pattern exists in real crate
artifacts, even when the selector itself is not secret.

The `elliptic-curve` hits are in development mock-curve code. Some of those
are useful as regression tests for the detector, but they are not production
curve arithmetic findings.

If the selector comes from a secret, the finding becomes security-sensitive. If
it comes from public length, format state, parser state, allocation state, or a
fixed field parameter, it is evidence for the compiler pattern and the detector,
but not a timing finding by itself. The expanded triage table is in
[`real_world/results/expanded_triage.csv`](https://github.com/van-ema/ct-rust-verifier/blob/main/real_world/results/expanded_triage.csv),
and the expanded run is summarized in
[`reports/expanded-real-world-evaluation.md`](https://github.com/van-ema/ct-rust-verifier/blob/main/reports/expanded-real-world-evaluation.md).

## What The Systematic Search Found

The systematic search produces two useful results.

First, the selected-address-load shape is not limited to the tiny reproducer. It
appears in optimized artifacts from real Rust crates. That matters because it
shows the compiler pattern is not just a lab construction.

Second, source-only analysis is far too weak for this problem. Many source
patterns look interesting but do not produce the final access shape. Some final
assembly patterns are real selected loads, but their selectors come from public
state such as length, precision, formatting, parser state, allocation state, or
fixed public field parameters. Those are still useful findings for the detector
and for understanding the optimizer, but they are not timing vulnerabilities by
themselves.

At the moment, the systematic investigation has not confirmed a real upstream
crate vulnerability. The positive finding is narrower and still important:
Rust-level aliasing semantics can affect the memory-access shape that a
constant-time implementation relies on, and that effect can be observed in both
controlled experiments and real optimized crate artifacts.

## Constant-Time in Rust

For constant-time Rust, the practical rule should not be "never use unsafe" or
"never use references". That is too broad to help.

A better rule is:

> When a memory access pattern is part of the constant-time argument, review the
> optimized assembly for that access pattern, especially if unsafe code creates
> references, slices, or alias-separated views around the data.

In practice, this means:

- Watch `&mut` and reconstructed slices in constant-time selection paths.
- Be careful when a source-level argument depends on "load both sides before
  selecting".
- Check whether LLVM IR contains `noalias`, `alias.scope`, or related alias
  metadata on the relevant pointers.
- Check whether assembly still loads both addresses, or whether it selects an
  address and loads once.
- Classify each selected load by selector source: secret selectors are the
  security-sensitive ones.
- Keep small assembly regression tests for the access shapes you rely on.

Raw pointers and volatile operations are not general constant-time strategies.
In this benchmark family, raw-pointer forms avoided the specific alias facts
that enabled the selected-address transform.

The important thing is not the syntax. It is the contract you give the
optimizer.

## Conclusions

The core finding is specific:

> Alias metadata from unsafe Rust can let LLVM legally rewrite fixed-load
> constant-time-looking code into selected-address-load code. If the selector is
> secret, that can become a timing leak.

The current evidence includes one confirmed extracted real-world-derived timing
case, a small taxonomy of alias-driven transforms, and real crate artifacts
where the same selected-load codegen shape appears. That is enough to make the
mechanism worth taking seriously.

Constant-time security lives in the binary, not just in the source. Rust gives
developers strong tools for writing safe and fast code, but unsafe code can also
give the optimizer strong promises. When those promises interact with a
constant-time argument, the final access pattern needs to be checked.

At the moment, no upstream crate vulnerability has been confirmed. That should
be future work, not a reason to ignore the mechanism. The next step is to
investigate upstream call paths where this transform is reachable with a secret
selector, extract more real-world-derived reproducers, and measure them under
controlled timing tests.

The compiler follows the rules it is given. The security lesson is that
constant-time reviews need to follow the data all the way down.
