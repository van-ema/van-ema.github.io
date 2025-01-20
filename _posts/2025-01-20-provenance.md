---
layout: post
title: Pointers provenance is real
date: 2025-01-20 11:12:00-0400
description: Provenance and Rust
# tags: formatting
categories: Rust
---
### Pointers are not integers

and Rust made it very clear to everyone.

I may understand if you've never heard about pointers provenace.
Maybe you are a 10yoe programmer and still you never needed to know what provenance is. 
Similar to apparent forces in Physics, Provenance exists, just because we can see it. Therefore, we must define so that the world can work as it actualy does.
So let's start from the beginning. How do we know provenance exixsts and where to find it?

## Provenance in Rust

The statement 'pointers are just integers' is refuted by the [counterexample](https://godbolt.org/z/ce4bjqjbM) illustrated in [RFC3559 of Rust])https://rust-lang.github.io/rfcs/3559-rust-has-provenance.html).
So we now know that provenance exists but we don't actually have a mean to use it when we code. isn't it?
Rust 1.84 stable version introduce APIs to manipulate pointers and specify provenance explicitly. 
Rust documentation states:

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
