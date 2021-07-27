---
layout: post
title: SSA transformation for GHC's native code generator (Part 3)
tags:
  - Haskell
  - GHC
  - compilers
  - ssa
  - register_allocation
  - programming
---

# Another Try

In the [first]({% post_url 2021-04-27-ssa_for_ncg_part1 %}) [two]({% post_url 2021-04-30-ssa_for_ncg_part2 %}) parts of this series,
I described my motivations for bringing SSA transformation to NCG, what SSA form is and an evaluation of my implementation.

My original implementation used the classic Cytron et al. algorithm. The reason for that was, that a) there was plenty of literature
on that (e.g. in "Engineering a Compiler") and b) that GHC already had functions to calculate the dominators etc.
To be honest, I did look at the Braun et al.[^1] paper (which is probably the most popular one these days), but I found it a bit confusing.
While the base algorithm is quite simple, there are plenty of improvements and additions they describe.

Alas, as my implementation was just way too slow to be practical (up to 30% increase in compile times), I set out to reimplement it with
the Braun et al. algorithm.

# SSA à la Braun et al.

The basic algorithm is quite simple. Looking at all blocks, all instructions in those blocks, we "remember" the new value for any definition
and we look up the current value for any use.
The look-up starts out local, checking if there is a definition in the current block and recurses into the block's predecessors if there is none.
If we only have one predecessor, we can simply walk up until we find a definition or multiple predecessors.
Otherwise we have to place Phi-nodes, bc. we have merging control-flow.

Phi nodes are placed "defensively", to mark already visited blocks and their arguments may be added later (incomplete Phis).
After adding a Phi's arguments, it's checked for "triviality" and removed if it is trivial.
A Phi node is trivial, if it doesn't merge at least two distinct values. The argument list is cleaned of any duplicates
and the declaring Phi itself (a possible self-reference). If only one distinct value remains, they Phi node can be replaced with that value.
For example: `p1 = φ(p1, v2)` is trivial and each occurrence of `p1` can be replaced with `v2`.
Whereas `p2 = φ(v1, p1)` actually merges two distinct values and therefore is not trivial.

Incomplete Phis are added for blocks, which are not yet "sealed". I'll simply quote the paper[^1] here, section 2.3,
as this is a very clear explanation:

> We call a basic block sealed if no further predecessors will be added to the block.
> As only filled blocks may have successors, predecessors are always filled. Note
> that a sealed block is not necessarily filled. Intuitively, a filled block contains
> all its instructions and can provide variable definitions for its successors.
> Conversely, a sealed block may look up variable definitions in its predecessors as all
> predecessors are known.

The algorithm is capable of constructing SSA form as the CFG is being constructed.
In NCG, we don't need this, as instruction selection happens before and we have a complete CFG and filled blocks.
I'm still using these two states[^2], because you need to handle loop tail's edges to the loop head.
In that case, you won't have the definitions from the loop body when you are processing the loop head.
To avoid recursing into the whole loop body (and having to figure out when to stop, so you won't get an infinite recursion),
I simply consider blocks "not sealed" if not all of their predecessors are filled.
My definition of "filled" is, that all instructions have been processed to find definitions (i.e., the block has been "visited").

## Value Numbering

I did not mention anything about generating new SSA names for definitions in the previous section.
In fact, neither does the paper.
You see, the algorithm performs a [(global) value numbering](https://en.wikipedia.org/wiki/Value_numbering) as a natural side-effect.
That is because the algorithm looks for the current _value_ of a variable. This may be a simple value or an expression.
Think of it as the right hand side of an expression.

```
# source
x <- 2
y <- 2
z <- x + y
u <- x + 2

# result
v1 <- 2

v2 <- v1 + v1
v3 <- v2
```

GVN is an optimization and also the basis for further optimizations.

The paper seems to assumes some expressions like in an AST or some IR.
Unfortunately, things aren't as easy with assembly.

To make it short, there are several challenges:
You basically have to model the semantics of any instruction you want to consider for
all supported ISAs.
After all, you have `addq $2,%v1` and you need to know that this means `v1 <- 2 + v1`,
or that `xor %v1,%v1` means `v1 <- 0`, or multiplication/division by bit-shifting.
Further more, some instructions have implicit (result) operands, like status registers etc.
Most of the time, you won't simply add constants (immediate values) to virtual registers though.
You will have to deal with memory access, like memory access relative to the base pointer,
so you will need to keep track of the current offset.
Also complex addressing modes accessing memory, e.g., ModR/M-Addressing for x86: `mov (%esi,%ebx,4), %edx`.

All in all, it seemed too complex. For that reason, my implementation simply generates a new SSA name
for definitions of virtual registers.

## Removing Trivial Phis

As I wrote earlier, Phi nodes are placed "defensively" and are later checked for "triviality".
If a Phi node is deemed trivial, it can be replaced by its only argument value.
In turn, any Phi node having that trivial Phi as an argument could now become trivial:

```
p1 = φ(p1, v1)
p2 = φ(p1, v1)

# p1 is trivial: p1 -> v1
p2 = φ(v1, v1)

# p2 has become trivial: p2 -> v1
```

This is conceptually quite simple, but some things have to be considered when implementing this.
As this may remove previously inserted (and used!) Phis, there will be instructions using these Phis.
In the paper, it looks as if they are using def-use chains to find and rewrite all those uses.
We don't have those and computing them incurs an overhead.
Instead, I simply keep a mapping of all renamings (e.g., `{p2 -> p1, p1 -> v1, p3 -> v2}`)
and then perform a second rewriting pass over the code with the "canonized" names
(e.g., `{p2 -> v1, p1 -> v1, p3 -> v2}`).

This also necessitates keeping a Phi dependency graph, to keep track of all Phi users of a Phi node,
in case that node becomes trivial and all users have to be updated.

The whole procedure is quite imperative and "mutating" in nature, so I'm making extensive use of the
state monad. I'd actually like to see the state be shrunk somewhat, but I'll look into optimizations in the future.

# SSA Flavor and Optimizations

In my previous articles, I wrote that I was using "pruned" SSA - precomputing liveness to make sure to only
insert live Phi nodes.
The Braun et al. algorithm naturally produces pruned SSA form, as it will only insert Phi nodes "on-demand",
i.e., when it encounters the use of a variable coming from merging control-flow.

As for _minimality_, - _minimal SSA_ requires that Phi nodes for a variable are only placed in the first
block where multiple definitions of that variable meet for the first time - the algorithm produces
_minimal SSA_ only for [reducible control flow](https://en.wikipedia.org/wiki/Control-flow_graph#Reducibility).
Section 3.2 contains the description of a post-processing pass to remove any Phi nodes inserted because
of irreducible control flow.
So far, I have not implemented this though. Looking at some simple benchmarks, I could not see any excess
of Phi nodes. I might have to revisit this later though.

The paper also contains some extensions to remove the number of temporary Phi nodes inserted, in section 3.3.
It may be worth looking at these to improve performance, but I have not implemented them for now.

# Other Implementations

I looked at [SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools/blob/e0937d7fd1dbcb306c93d0806954a0dac08f48b9/source/opt/ssa_rewrite_pass.cpp)
and [Cranelift's](https://github.com/bytecodealliance/wasmtime/blob/3da677796b58fac7aedaa87b51c494da4e8df2a6/cranelift/frontend/src/ssa.rs) implementations
for inspiration.
(There is also [LLVM](https://llvm.org/doxygen/MemorySSAUpdater_8cpp_source.html), but the whole situation there is much more complicated
and I didn't really dive into that)

I found both implementations very readable, but my situation in Haskell somewhat different, so many things did not apply.

There is also [LibFirm](https://github.com/libfirm/libfirm), which contains the original implementation by the paper authors.
(Alas, it's written in C and I find navigating the code base quite challenging).

# Results

So, how did it turn out?
To test my implementation, I made sure that both versions are rebased on master.
Then I used "cabaltest" - just a simple script to compile Cabal with provided flags - for comparison.
There is a big caveat here: I only did one compile per configuration, but by repeating some of them, I saw quite the variability.
I'd say that you should add ~5% error bars mentally.

To my chagrin, I also found out, that my first version seems to perform particularly badly with -O2, but quite OK with -O1.
I did my last evaluation only with baseline (no optimizations) and -O2 - which was a bad choice anyway.
But it wasn't all for naught, because the new implementation is still faster and uses way less memory.

| Branch                                                   | Flags                | User Time | Time Change % | Allocated    | Alloc. Change % | Max residency | Max Res Change % |
| -------------------------------------------------------- | -------------------- | --------- | ------------- | ------------ | --------------- | ------------- | ---------------- |
| Asm-ssa cytron (7ffabc)                                  | '-O1                 | 199.77    |               | 176875852448 |                 | 531635592     |                  |
| Asm-ssa cytron (7ffabc)                                  | '-O1 -fssa-transform | 211.19    | 5.72%         | 222723463896 | 25.92%          | 539740512     | 1.52%            |
| Asm-ssa cytron (7ffabc)                                  | '-O2                 | 258.68    |               | 223571517608 |                 | 684983536     |                  |
| Asm-ssa cytron (7ffabc)                                  | '-O2 -fssa-transform | 318.62    | 23.17%        | 336322854056 | 50.43%          | 690025104     | 0.74%            |
| Asm-ssa-braun (b5b113)                                   | '-O1                 | 210.6     |               | 176875529800 |                 | 552021824     |                  |
| Asm-ssa-braun (b5b113)                                   | '-O1 -fssa-transform | 227.87    | 8.20%         | 180666914120 | 2.14%           | 547049824     | -0.90%           |
| Asm-ssa-braun (b5b113)                                   | '-O2                 | 225.9     |               | 223570497192 |                 | 688337672     |                  |
| Asm-ssa-braun (b5b113)                                   | '-O2 -fssa-transform | 249.13    | 10.28%        | 227890634696 | 1.93%           | 686895536     | -0.21%           |

By looking at this table you might think "hey! the new version is actually _slower_ at -O1".
I'm not convinced that it is, see my comment regarding the "error bars".
I've seen changes of 10 seconds or more across runs, so I'm inclined to say that they are about equal in terms of
speed for -O1, where the new version performs much fewer allocations.
Either way, I'll keep the new one anyway and rerunning the benchmarks would take quite a long time.
Mind you that the new implementation is absolutely not optimized and I'm sure that there are quite some gains to be had.

I also wanted to look at the number of Phi arguments, to see how I can optimize their representation (currently it's just a list).

| Impl | prog | count | min args | max args | avg args |
| ------ | ------ | ------ | ------ | ------ | ------ |
| old | real/small-pt | 216 | 2 | 5 | 2 |
| new | real/small-pt | 216 | 2 | 5 | 2 |
| old | spectral/simple | 345 | 2 | 4 | 2 |
| new | spectral/simple | 345 | 2 | 4 | 2 |
| old | shootout/fannkuch-redux | 257 | 2 | 7 | 2 |
| new | shootout/fannkuch-redux | 257 | 2 | 5 | 2 |

The "count" columns refers to the total number of Phi functions in the compiled modules.
Interestingly, they are the same for the three benchmarks I looked at from the "nofib" suite.
So there aren't any superfluous Phi nodes inserted by the new implementation (see _minimality_ above).
It's curious that the old implementation seems to insert _more_ arguments in the last program,
as indicated by the higher "max args" number.
Those may be duplicate elements, as the argument lists are de-duplicated for the triviality check
in the new implementations, but I have not verified this hunch.

# Conclusion

I'm somewhat satisfied with the new implementation, but I think there is still a lot of potential for improvement.
Especially performance can always be better.
I might have to look into the post-processing step to ensure minimality for irreducible control flow later,
but for now I have my eyes on _liveness analysis_.
But stay tuned, I might write about that soon.

For those who want to keep track of this effort, the [ticket](https://gitlab.haskell.org/ghc/ghc/-/issues/19453) is still the same
and [this is a new MR](https://gitlab.haskell.org/ghc/ghc/-/merge_requests/6170).

If you spot any mistakes, omissions or want to provide feedback, comments, questions - you can reach me on [Twitter](https://twitter.com/cptwunderlich), or via mail with my username at gmail.

[^1]: [Braun et al. - "Simple and Efficient Construction of Static Single Assignment Form"](https://link.springer.com/chapter/10.1007%2F978-3-642-37051-9_6)
[^2]: [SPIRV-Tools SSA rewrite pass](https://github.com/KhronosGroup/SPIRV-Tools/blob/e0937d7fd1dbcb306c93d0806954a0dac08f48b9/source/opt/ssa_rewrite_pass.cpp) seems to only use "sealed", but IIRC that differed from the paper's definition.

*[ABI]: Application Binary Interface
*[AST]: Abstract Syntax Tree
*[CFG]: Control Flow Graph
*[GHC]: Glasgow Haskell Compiler
*[IR]: Immediate Representation
*[IRs]: Immediate Representations
*[ISA]: Instruction Set Architecture
*[LR]: Live Range
*[LRs]: Live Ranges
*[NCG]: GHC's Native Code Generator
*[SSA]: Static Single Assignment
*[vreg]: Virtual Register
*[GVN]: Global Value Numbering
