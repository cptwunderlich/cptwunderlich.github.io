---
layout: post
title: SSA transformation for GHC's native code generator (Part 2)
tags:
  - Haskell
  - GHC
  - compilers
  - ssa
  - register_allocation
  - programming
---

# SSA based Live Range Discovery

In the [first part]({% post_url 2021-04-27-ssa_for_ncg_part1 %}), I explained the motivation
to bring SSA-transformation to GHC's NCG.
Here, I want to look at the implementation challenges, decisions and results.

# SSA in the Backend

I've used pseudo-code in the first part, but generally, you'll have some intermediate representation (IR) in your compiler, which is in SSA.
For example, a three address code (TAC), where each instruction represents an operation,
taking two _input_ arguments and one _output_ argument, e.g., `IADD x, y, z` -> `z = x + y`.

Modern compilers usually use multiple IRs, adapted to different compilation phases and tasks.
They may or may not be in SSA.
GHC has been around for a while and has several compilation phases and IRs.
I won't describe them here in detail[^1], but they are Core, STG, C--.
C-- is a low-level, imperative language with an explicit stack.
NCG performs instruction selection on C-- and spits out machine instructions for the selected
platform.
So at this point, we're not dealing with an IR anymore[^2].

![GHC's Intermediate Representations](/assets/posts/ssa-ncg/GHC-Pipeline.png)

There are actually quite a few challenges when trying to implement SSA-form on machine instructions.

## Pre-colored Registers

Sometimes, the ISA or ABI requires you to use certain registers in specific places.
For example, the x86_64 calling convention requires the passing of function arguments in
specific registers.
Or think about special purpose registers, like the x86_64 stack registers `RBP` and `RSP`.

Instruction selection already emits these physical register references where necessary.
Obviously, we can't promote these to SSA variables and start renaming them willy-nilly.
In this case, i.e., for live range discovery, we can simply get a way with ignoring them all together, since we are not moving any code[^3].
(FYI, this will be a common theme in the following subsections.)

## Modifying and 2-Address Instructions

Most real world ISAs have the annoying property of not being neat, regular and simple.
x86 has many instructions that modify an operand (i.e., in **and** out operands).
Let's say `%v1` denotes a virtual register number 1, then `inc %v1` _increments_ `%v1`,
or `v1 := v1 + 1`.
Another example is the two-address addition instruction: `add %v1, %v2` is `%v2 = %v1 + %v2`.

This is clearly not in SSA, as we are producing a value but we can't assign a new name!
One may introduce an intermediate representation to handle this case, constraining the
in-out operand pair together somehow - but again, in our use case, we can ignore this!

If we only care about live ranges having unique vreg names, this is OK because the in-out operand
pair belongs together anyway. There is no way we could assign different registers here.

## Conditional Instructions

Some architectures have [conditional (or predicated) instructions](https://en.wikipedia.org/wiki/Predication_(computer_architecture)), which will only execute,
if a specific condition is met, normally some status bits must be set in a status register.
For example, x86_64 offers the `cmov`, or conditional move instruction.
As an example, `CMOVE %v1, %v2` is equal to `if (x == y) { v2 := v1 }`.

32-Bit ARM can basically make every instruction predicated, whereas AARCH64 dialed that down a bit.

I'm not going to discuss their benefits or drawbacks, but since NCG supports architectures with these instructions, we need to take them into account.
Again, I'm choosing the most simple route. I consider them non-kill definitions and don't rename their target.

# Implementation

Let's get down to business. The ticket tracking the implementation can be found [here](https://gitlab.haskell.org/ghc/ghc/-/issues/19453).

## Overview

Instruction Selection -> Liveness Analysis -> SSA Construction -> SSA Destruction -> Register Allocation

# Evaluation

# Conclusion


If you spot any mistakes, omissions or want to provide feedback, comments, questions - you can reach me on [Twitter](https://twitter.com/cptwunderlich), or via mail with my username at gmail.

[^1]: I recommend [Takenobu's GHC Reading Guide](https://github.com/takenobu-hs/haskell-ghc-reading-guide) for an overview.
[^2]: Well, there are still a few meta-instructions, like SPILL and LOAD, but all-in-all it's pretty close to assembly.
[^3]: I think one could promote them to a distinct class of SSA variables and store a mapping to the original register.

*[ABI]: Application Binary Interface
*[GHC]: Glasgow Haskell Compiler
*[IR]: Immediate Representation
*[IRs]: Immediate Representations
*[ISA]: Instruction Set Architecture
*[LR]: Live Range
*[LRs]: Live Ranges
*[NCG]: GHC's Native Code Generator
*[SSA]: Static Single Assignment
*[vreg]: Virtual Register
