---
layout: post
title: Rust MIR Instrumentation
date: 2025-12-14 11:12:00-0400
description: How to inject calls inside MIR
# tags: formatting
categories: Rust
---
# Instrumenting Rust MIR with a Custom Compiler Driver

Rust’s Mid-level Intermediate Representation (MIR) is one of the most powerful—but least documented—extension points in the compiler. If you want to observe reference creation, track memory, or build dynamic analyses, MIR is usually the right layer.

This post explains **how to instrument Rust MIR in practice**, focusing on three concrete questions:

1. **Which rustc APIs to use**
2. **How to build a custom rustc driver**
3. **How to link a runtime crate safely (even under LTO)**

All examples are based on a working multi-crate workspace.

---

## 1. Which rustc APIs to Use

### Nightly and `rustc_private`

MIR instrumentation is not available on stable Rust. You must use nightly and opt into rustc’s internal APIs:

```rust
#![feature(rustc_private)]

extern crate rustc_driver;
extern crate rustc_interface;
extern crate rustc_middle;
extern crate rustc_mir_transform;
extern crate rustc_span;
```

### The `optimized_mir` Query

Rustc exposes MIR through *queries*. The most useful query for instrumentation is `optimized_mir`, because it runs **after MIR is built and optimized**, but **before** codegen to LLVM.

In a custom driver you can override this query via `Config::override_queries`:

```rust
_config.override_queries = Some(|_session, queries| {
    queries.optimized_mir = CUSTOM_OPT_MIR;
});
```

Your hook has the shape:

```rust
const CUSTOM_OPT_MIR: for<'tcx> fn(tcx: TyCtxt<'tcx>, def: LocalDefId) -> &'tcx Body<'tcx> =
    |tcx, def| {
        let mut body = (rustc_interface::DEFAULT_QUERY_PROVIDERS.optimized_mir)(tcx, def).clone();

        // Your MIR instrumentation goes here.
        MyOptimizationPass.run_pass(tcx, &mut body);

        tcx.arena.alloc(body)
    };
```

The important pattern is:

1. Call the default provider (`DEFAULT_QUERY_PROVIDERS.optimized_mir`) to get rustc’s MIR.
2. Clone it (you need an owned `Body` to edit).
3. Modify it (insert statements/terminators, add locals, etc.).
4. Allocate it in the compiler arena and return `&'tcx Body<'tcx>`.

If you pick an earlier query (like `mir_built`) you’ll see less optimized MIR; if you pick a later stage you may miss the chance to inject cleanly before codegen.

---

## 2. How to create a custom rustc driver

Rust no longer supports compiler plugins, so the standard approach is to **replace `rustc`** with your own binary that embeds rustc and overrides queries.

The basic structure is:

```rust
struct CompilerCallbacks;

impl rustc_driver::Callbacks for CompilerCallbacks {
    fn config(&mut self, config: &mut rustc_interface::Config) {
        config.override_queries = Some(|_session, queries| {
            queries.optimized_mir = CUSTOM_OPT_MIR;
        });
    }
}

fn main() {
    let mut callbacks = CompilerCallbacks;
    rustc_driver::run_compiler(&std::env::args().collect::<Vec<_>>(), &mut callbacks);
}
```

In practice you usually want a **Cargo subcommand** wrapper (`cargo-instrument-mir`) that runs `cargo build` but sets:

- `RUSTC=/path/to/instrument-mir`

This makes the workflow feel like a normal Cargo command:

```bash
cargo instrument-mir -p hello --release
```

---

## 3. How to link a runtime crate safely (even under LTO)

MIR instrumentation usually needs a **runtime crate** to record events (log, track pointers, update global state, etc.). The compiler pass injects calls, but those calls must resolve and the runtime must not be stripped by the linker.

### 3.1 Make rustc able to see the runtime

If your driver injects calls like `runtime::__injected_hook(...)`, you must ensure rustc can resolve the `runtime` crate. A simple way is to add `-L` and `--extern` when your driver invokes rustc:

```rust
let runtime_path = "/path/to/workspace/target/release";

args.push("-Zunstable-options".to_string());
args.push(format!("-L{runtime_path}"));
args.push(format!("--extern=force:runtime={runtime_path}/libruntime.rlib"));
```

Notes:
- `-L` tells rustc where to find `libruntime.rlib`.
- `--extern=force:runtime=...` makes the crate available even if it is not referenced from source code.

### 3.2 Keep the runtime from being dead-stripped (especially with LTO)

With LTO enabled, the linker can remove crates and symbols that appear unused. Since the runtime is only referenced by injected MIR, it can look “unused” from the perspective of the normal Rust source.

A robust pattern is to export a C ABI hook and **force a reference to it**:

```rust
#[no_mangle]
pub extern "C" fn __injected_hook(addr: usize) {
    println!("__injected_hook called for address: 0x{:x}", addr);
}

macro_rules! force_runtime {
    ($sym:path) => {
        #[used]
        static _FORCE_RUNTIME: fn(usize) = $sym;
    };
}

force_runtime!(__injected_hook);
```

This does two things:
- `#[no_mangle]` + `extern "C"` gives you a predictable symbol.
- `#[used]` prevents the symbol from being dropped during optimization and linking.

### 3.3 Use a stable hook ABI

Don’t pass typed references like `&T` to the runtime hook. It’s brittle and can lead to invalid IR (especially under LTO). Instead, compute a stable representation in MIR (for example, a `usize` address) and pass that.

On recent nightlies with strict provenance, the cast you want in MIR is typically:

```rust
CastKind::PointerExposeProvenance
```

That converts a pointer-like value into an integer address in a way that rustc/LLVM accept.

---

## References

- [Writing a custom `rustc` driver](https://jyn.dev/rustc-driver/)