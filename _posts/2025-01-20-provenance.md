---
layout: post
title: (Rust) Pointers provenance is real
date: 2025-01-20 11:12:00-0400
description: Provenance and Rust
# tags: formatting
categories: Rust
---
### Pointers are not integers

and Rust made it very clear to everyone.

Have you ever heard of pointer provenance? If not, don’t worry—you’re not alone. Even if you’ve been programming for 10 years, you might have never needed to think about it. But here's the thing: provenance is a bit like those "apparent forces" in physics—it exists because we can observe its effects. And since it’s there, we need to define it to make sense of how things really work under the hood.

So, where do we even start? How do we know pointer provenance exists, and how can we find it? Let’s break it down.
## Provenance in Rust

The claim that "pointers are just integers" doesn’t hold up, as demonstrated by [counterexample](https://godbolt.org/z/ce4bjqjbM) and explained in [RFC3559 of Rust])https://rust-lang.github.io/rfcs/3559-rust-has-provenance.html).
But here’s the catch — while provenance exists, we haven’t really had a way to interact with it directly in our code. That changes with Rust 1.84. The stable release introduces new APIs that let developers manipulate pointers and explicitly define their provenance.

As the Rust documentation state

    It is undefined behavior to offset a pointer across a memory range that is not contained in the allocated object it is derived from

Rust 1.84 introduces `wrapping_offset` to create a pointer that points outside its provenance (dereferencing the pointer is still UB!)
However the LLVM IR for `offset` and `wrapping_offet` is quite similar. Why then `offset` leads to undefined behavior even when the result is not deferenced? Let's look at the [LLVM IR here](https://godbolt.org/z/3Mz7serhx)

with `offset`:
```
  store i64 6, ptr %count.dbg.spill.i2, align 8
; call core::ptr::const_ptr::<impl *const T>::offset::precondition_check
  call void 
  @"_ZN4core3ptr9const_ptr33_$LT$impl$u20$$BP$const$u20$T$GT$6offset18precondition_check17h058d8998d9a55876E"(ptr %_6, i64 6, i64 1) #20, !dbg !2498
  %_0.i4 = getelementptr inbounds i8, ptr %_6, i64 6, !dbg !2500
  ```

with `wrapped_offset`:
```
 store i64 6, ptr %count.dbg.spill.i2, align 8
  #dbg_declare(ptr %count.dbg.spill.i2, !2351, !DIExpression(), !2354)
  %9 = getelementptr i8, ptr %_6, i64 6, !dbg !2355
```

and what LLVM LangRef says about Gep instruction?
    The result value of the getelementptr may be outside the object pointed to by the base pointer. The result value may not necessarily be used to access memory though, even if it happens to point into allocated storage

and again
    The getelementptr instruction may have a number of attributes that impose additional rules. If any of the rules are violated, the result value is a [poison value](https://llvm.org/docs/LangRef.html#poisonvalues). 

and therefore and out-of-bound Gep which is not used afterwards is not UB.

Further readings:
- [Strict provenance in Rust](https://doc.rust-lang.org/nightly/std/ptr/index.html#strict-provenance)
- [Pointers Are Complicated blog post by Ralf's Jung](https://www.ralfj.de/blog/2018/07/24/pointers-and-bytes.html)