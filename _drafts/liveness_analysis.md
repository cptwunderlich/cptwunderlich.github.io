---
layout: post
title: Liveness Analysis in GHC and beyond
tags:
  - Haskell
  - GHC
  - compilers
  - ssa
  - liveness
  - programming
---

# Working on Liveness Analysis in GHC

If you haven't seen my other posts on here, I've been working on bringing
[SSA-form into NCG]({% post_url 2021-04-27-ssa_for_ncg_part1 %}).
So far, I've implemented (and [re-implemented]({% post_url 2021-07-27-ssa_for_ncg_part3 %})
SSA transformation and destruction (out-of-SSA transformation).
Which means, I transform out-of-SSA and reuse the existing compiler passes as they are.
That alone didn't really improve register allocation the way I hoped,
but I'm working towards an SSA-based graph coloring register allocator.
For that, I need liveness information _on SSA-form_.

# What is 'Liveness' Anyway?

A variable is said to be 'live' at a program point p, if its value will be needed at a later point in the program,
i.e., there is a path from program point p, where the variable is live to another point q,
where the variable is used and q is executed after p.
(That also works for loops, the body is executed after the header, even if we then return to the header)

This information is needed for several things, e.g., you can use it to detect uninitialized variables,
for dead store elimination and most notably for register allocation.
It is vital for register allocation, because registers are a limited, precious resource and
you want to know which values need to be kept in registers and when those become free again.

Let's start by clarifying some confusing terminology.
Program _Variables_ can be variables used by the programmer (e.g., in Java `int x;`),
but the compiler will also generate temporary variables while messing with your code.
In most imperative languages, a variable is by default mutable and can be changed,
or redefined.
This is mostly useful at a "higher level" in your compiler and you can, e.g.,
detect uninitialized variables.

Of course, Haskell doesn't have mutable variables, but as your program gets lowered to an
intermediate representation, which is closer to the machine your program will run on,
it will deal with _virtual registers_ and stack memory.
A _virtual register_ or _vreg_ may also be defined multiple times,
but using SSA construction and destruction to perform "web analysis", a vreg should
represent a "web", so values that are directly connected by dataflow.
Here some pseudo-code to illustrate this:

    int x := 1
    int y

    if z >= 0 {
        x := x + 1
        y := 10
        use y
    } else {
        x := x - 1
        y := 20
        use y
    }

    use x

When we translate this, we can use one vreg for `x`, because we want to assign real register to vregs 1-to-1
and we need the value of `x` for those operations.
`y` on the other hand has two completely disjoint definitions and uses. We can use two different vregs.
Using the same would constrain us to use the same real register and might block that for longer than necessary.

A `value` is - well, exactly that. Instead of looking at variables or vregs, we can look at distinct _values_.
This is what we do in SSA transformation, where we speak of _SSA values_ (I've also read _SSA variables_).
The point is, that there must be exactly one definition for an SSA value.
So in the above example, there are three definitions of `x`.
Those are three different values and in SSA form, they are usually depicted using subscripts 
(x<sub>0</sub>, x<sub>1</sub>, x<sub>2</sub>).
If you haven't read my other posts, let me add here, that I have implemented a sort of "pseudo-SSA" in NCG,
because at that level, we're dealing with (almost) assembly instructions.
Intel x86_64 instructions - unfortunately - love to modify their operands, i.e., you have 2-address instructions
with source and source AND destination.

Lastly, what's a _live range_? A live range describes from which program points to which your variable/vreg/value is _live_.
For simplicity, I will just write "vreg" from now on, because that is what I'm working with in NCG.
A live range isn't necessarily contiguous, e.g., those if-else blocks above are not executed sequentially, but are alternatives.
So a live range may be composed out of several _live intervals_[^1].

![control flow graph for if](/assets/posts/liveness/if.png)

Here we can see, that `x` is live across the whole program,
but `y` is only live from its definition to the last use in the right alternative branch.

# Live Range Representations

Many textbooks (and papers) talk about _live sets_ (or _liveness sets_).
For a given basic block, Live<sub>In</sub> contains all the vregs that are live at the beginning
of the block  ("available at the first instruction") and
Live<sub>Out</sub> the ones that are live at the end of a block.

You could simply use this, but, e.g., for register allocation, the "resolution" of this
is too imprecise.
Generally, you want to know which values are live at each instruction.
One could take the Live<sub>In</sub> set and simply walk over each instruction in the block
to get the liveness there, but you don't want to do that repeatedly.

Another way to represent liveness is a dedicated live range data structure.
For each vreg, you store a list of live intervals, which are half-open intervals
of program points.
In the above example, we have a consecutive numbering of instructions and
the live range of `x` is \[0,3), \[3,5) and \[5,6).

In NCG, instructions are annotated with liveness information directly, in addition to per-block Live<sub>In</sub>-sets.
So each instruction carries the following three sets: liveBorn, liveDieRead, liveDieWrite.
That is, which regs are 'born' or defined and which ones are _read_, respectively _written_ for the last time.
Live ranges that "reach" a successor block, i.e., are live-in, extend within the local block to the branch
instruction to that successor block.
So that live interval "dies" at that branch instruction and is in the Live<sub>In</sub> set of the branch target.
I think of 'born' and 'die' as 'start' and 'end' of a live interval segment.

To summarize, a live interval **starts** at
* the instruction where the vreg is defined
* the top of a BB if it is Live<sub>In</sub>

and it ends at
* the last instruction to use the vreg
* a branch instruction to a BB where the vreg is Live<sub>In</sub>

If you want to take a look at NCG's liveness representation, you can use the GHC flag `-ddump-asm-liveness`.
It will look something like this:

    {% raw %}
    ==================== Liveness annotations added ====================
    sat_s9Uz_entry() { //  [R1]
            { [(cjxI,
                sat_s9Uz_info:
                    const 1;
                    const 16 :: W32;
                    const _ujy5_srt-sat_s9Uz_info;)]
              # entryIds         = [cjxI]
              # liveVRegsOnEntry = [(cjxI, {%r1}), (cjxJ, {%vI_V0}),
                                    (cjxK, {%vI_V0}), (cjxL, {%vI_V0}), (cjxM, {%vI_V0})]
              # liveSlotsOnEntry = LM (UM (fromList []))
            }
        [NONREC
            cjxI:
                    movq %rbx,%vI_V0
                        # born:    %vI_V0
                        # r_dying: %r1
                         
                    leaq -40(%rbp),%vI_V1
                        # born:    %vI_V1
                         
                    cmpq %r15,%vI_V1
                        # r_dying: %vI_V1
                         
                    jb _blk_cjxJ
                         
                    jmp _blk_cjxK
                        # r_dying: %vI_V0
    {% endraw %}

(Note: for real regs, this prints the actual register name in the instructions,
but the liveness info uses the internal name, which simply assigns a number to each reg.
So "rax" is "r0", "rbx" is "r1" etc.)

Register identifiers start with a percent sign "%" and then either "r" for real reg
or "v" for virtual reg.
After that comes the type, e.g., "I" for "Integer" and "D" for "Double", 
followed by an underscore and a unique name.

So in the snippet above, we can see that `r1` was live-in for block `cjxI`
and dies in the first instruction (last time read).
`vI_V0` is born in that instruction and dies at the last branch instruction, 
because it is live-in to block `cjxK` (see `liveVRegsOnEntry`).

`vI_V1` is a temporary, which is used only once right after it was "born".

Looking at the liveness info for your program, you might realize that liveness
annotations are missing for some real regs.
That is, because it is only recorded for _assignable_ registers.
This excludes, e.g., rsp and rbp, which are the C stack pointer
and the Haskell stack pointer respectively,
or also rip, the instruction pointer register.[^2]


TODO: LR representation: Numbering of instructions, two labels per instruction, order





// TODO

ssa values vs. real regs. LLVM comment regarding real regs interesting!
rsp and rbp never born/die?? -> makes sense, not allocatable!

what happens at function calls.
procedure arguments.

# Classic Fixed-Point Algorithm

# Engineering Live Range Representations

While textbooks mostly talk about the general algorithms used,
clever engineering of data structures makes a big difference for the performance 
of real world compilers.
Beyond mere asymptotic time-complexity, cache locality is an important factor.
I.e., you want compact representations and avoid lots of indirection.
Of course, this is a bit harder in Haskell, since we have less control over how things
are layed out in memory and there are lots of indirections by default.

Since I'm mostly focused on the code generator and register allocation in particular,
we'll look at two tasks: Building liveness information and constructing the 
[interference graph](https://en.wikipedia.org/wiki/Register_allocation#Graph-coloring_allocation).

## Liveness Sets and Annotations

To construct liveness sets and annotated instructions, we need to perform one of the construction
algorithms.
For either one, we need to go over each block **both** forwards and backwards.
The backward pass is necessary, to identify dying vregs.
So we'll have at least 2 iterations over the code (singly linked lists) 
and lots of set operations.
How much memory do we use?
The iterative algorithm in GHC useses one set for "live regs at this instruction"
and a map (IntMap) of Live<sub>In</sub> sets.
I wonder how this compares to the "traditional" bitvector representation with one
bit per variable.
My implementation of by-use path exploration needs a worklist (a map),
the current live set and the map of Live<sub>In</sub> sets. 

To construct the interference graph, you need to iterative over the code,
update the Live<sub>In</sub> set as you process the "born" and "dies" annotations
and add conflicts for any regs of same type[^3] live at the same time.
GHC just collects all register conflicts in a bag and build the graph from that.

One thing that is more difficult with this representation is finding the start
(or ends) of any given live range.

## Live Range Data Structure

One can represent this by using a map from register to a list of live intervals.
Instead of that list, one could also use yet another map from basic block to an interval.

As you perform one of the construction algorithms, you'll have to look-up the reg
(or create an entry if none exists), then find (or create) the interval for the current
block.
If you only deal with SSA values, your live range will only have one "definition interval"
and optional "live-in intervals".
I've taken this idea from [Cranelift's representation](https://github.com/bytecodealliance/wasmtime/blob/535b3a47ee4a4c5cf2c57d4b7296d64481a4afad/cranelift/codegen/src/regalloc/liverange.rs#L163).
Storing the definition point and end point, but an "optional" list of further intervals
is a clever optimization, since most live ranges are very short and live only within one block
(at least in SSA form programs).

In Haskell, it could look something like that:

~~~Haskell
type ProgramPoint = Int

data LiveInterval = LI BlockId ProgramPoint
                    -- ^ Start   End

data LiveRange = LR { lrReg         :: Reg
                    , lrDefBlock    :: BlockId
                    , lrDefStart    :: ProgramPoint
                    , lrDefEnd      :: ProgramPoint
                    , lrInIntervals :: [LiveInterval] }

type LiveRanges = UniqFM Reg LiveRange
~~~

This is bare-bones, without deriving any of the typeclasses you might want.
E.g., you'll definitly want `Eq`, but also comparing and ordering might make sense.
Also, this is not as efficient and compact as we'd like.
So some strictness annotations and `{-# UNPACK #-}` pragmas might be in order.
(Or using some type of array for `lrInIntervals`, but using arrays in GHC is a pain,
see [18927](https://gitlab.haskell.org/ghc/ghc/-/issues/18927) and
[20299](https://gitlab.haskell.org/ghc/ghc/-/issues/20299).)
Note that a `ProgramPoint` being a plain int is very efficient,
but of course not a great abstraction.
It relies implicitly on a specific numbering scheme, e.g.,
how do you include Phi-nodes, if they are block parameters and not part of the instruction stream.

Now, one of the biggest issues with that format for GHC is,
that (in my implementation), only vregs are in SSA form.
Some code is emitted using real registers directly and they may have multiple
definitions.

One advantages of this representation is that we can update live intervals
independently of our current position in the code.
If you look at [Cranelift's implementation](https://github.com/bytecodealliance/wasmtime/blob/8ebaaf928dc206543fc4ca8f42aa175f6d4e1eab/cranelift/codegen/src/regalloc/liveness.rs),
(which I would highly recommend!) they only need one pass over the code
(plus the upward path exploration).
When encountering a definition, you create a LR whist begins and ends
at that instruction.
As you encounter another use of that value, you update the 'end' position.
When processing a successor and finding out, that that value is live-in,
you can go back and extend that original LR to the branch leading to this successor.
But this means that you either need to store which instruction took you here 
(AFAIK [Cranelift does that](https://github.com/bytecodealliance/wasmtime/blob/03077e0de9bc5bb92623d58a1e5d78b828fd1634/cranelift/codegen/src/flowgraph.rs#L35)),
or go back and search for it.
As you insert or extend new intervals, you have to iterate over the existing
intervals of a live range.
In practice, this shouldn't be too bad and Cranelift "compresses" intervals,
i.e., if you have several "adjacent" intervals, that span the whole block
(they are live-in and live-out) you can merge them into one.

Anyway, I thought Cranelift's design is very interesting and
exceptionally well documented.
It even contains comparisons (and thus explanations) to LLVM.
If you're interested, you should [start here](https://github.com/bytecodealliance/wasmtime/blob/8ebaaf928dc206543fc4ca8f42aa175f6d4e1eab/cranelift/docs/regalloc.md#liveness-analysis)
and then check out the other links I added throughout this section.

But what about constructing the interference graph?
One has to iterate over all LRs and compare only the definition interval of the current LR
against all the intervals of the other LRs.
(but non-SSA LRs need to compare all intervals)

You do need an efficient way to map program points back to your instructions.
When calculating register pressure[^4] this is a downside,
as you want to know how many regs are live at the current instruction.

# Liveness on SSA-Form

When computing liveness for general variables, you may have multiple definitions
and last uses
In [SSA-form](https://en.wikipedia.org/wiki/Static_single_assignment_form) though, 
you only have one single definition.
So at any given program point p in block b, looking at a use of a variable v,
you don't know where you'd find another definition.
In the same block, in multiple predecessors, or sucessesors.
In SSA, the definition **has to be** either:

* a Phi-function at the top of the block
* regular instruction in the same block
* coming from a straight line of predecessors

That last one is important.
If you can use a value, it has to come from all incoming paths (otherwise it would be undefined on some path)
and whenever you have merging data-flow, SSA inserts a Phi-node.
So you'll never have to multiple paths or get caught in loops.

But SSA-form also raises some questions.
How should we treat Phi-functions?
As a refresher, the meaning of Phi-functions is that of _parallel copies_.
So they are "executed" _along the edge_ from a predecessor to the current block
and "all done" by the time we reach the blocks first instruction.
Another way to think about it, is that for the Phi-function
p<sub>0</sub> = &phi; (v<sub>0</sub>, v<sub>1</sub>), the value for p<sub>0</sub>
will be selected by the path taken.
That means, coming from the "first" predecessor, it sets p<sub>0</sub> = v<sub>0</sub>.

So what does that mean for liveness?
Phi _arguments_ are **not** considered live-in to the block, as the selection happens
"along the edge".
The value defined by the Phi is considered to be live-in.

The [cranelift regalloc README](https://github.com/bytecodealliance/wasmtime/blob/535b3a47ee4a4c5cf2c57d4b7296d64481a4afad/cranelift/docs/regalloc.md#liveness-analysis)
actually gives a terrific explanation[^5]:

> A starting point for a live range can be one of the following:
>
>   * The instruction where the value is defined.
>   * The EBB header where the value is an EBB parameter.
>   * An EBB header where the value is live-in because it was defined in a dominating block.
>
> The ending point of a live range can be:
>
>   * The last instruction to use the value.
>   * A branch or jump to an EBB where the value is live-in.

Quick explanation: An EBB is an extended basic block and an EBB parameter is a Phi node.

# Algorithms

Research report https://hal.inria.fr/inria-00558509v2/document

#
## GHC's Implementation

## SSA-based Liveness Analysis

## LLVM, Cranelift and others

### LLVM LiveVariables Pass

This is considered the "old" representation.

The [header](https://github.com/llvm/llvm-project/blob/f10d4cfc237bf778d659f645eaf5c4ecb094148b/llvm/include/llvm/CodeGen/LiveVariables.h) 
gives an overview and shows that `runOnMachineFunction` is the entry point
and this is the [implementation](https://github.com/llvm/llvm-project/blob/dc3fbe293f1a1c1068e2cd27151fb373798fdfb6/llvm/lib/CodeGen/LiveVariables.cpp).

On machine instructions. Seems to be BB-wise, e.g., similar to by-use algo.
Interesting comment about handling real regs: treated as block local only
and ignoring not register allocatable regs (stack pointer etc.).

A structure with liveness information for a variable is maintained.
The instruction in which a variable is 'killed' ('dies') is updated as the uses in a block
are encountered.
So this is one of the notable differences to NCG, where we have to annotate an instruction and
we can't easily go back to update this.

### LLVM LiveIntervals Pass

[header](https://github.com/llvm/llvm-project/blob/c4cd573b3247509da6d76222a87265f9efd6ad02/llvm/include/llvm/CodeGen/LiveIntervals.h) and [implementation](https://github.com/llvm/llvm-project/blob/main/llvm/lib/CodeGen/LiveIntervals.cpp).

This constructs live _intervals_ and corresponds to the 'by-var' algorithm in the research report.

Live "interval" is a bit of a misnomer, it corresponds to a live range containing multiple intervals,
but in LLVM terminology it's a "Live Interval" containing "segments".
[Talk](https://youtu.be/IK8TMJf3G6U?t=788)

## LibFirm

LibFirm is an SSA based backend from KIT (Karlsruhe Institute of Technology) and also where
Hack et al. implemented their SSA based graph coloring register allocator.
You can find their implementation of [liveness analysis here](https://github.com/libfirm/libfirm/blob/20c5dd7d55f6e3fc9b7bfbfdc34bc342d03aa798/ir/ana/irlivechk.c).

The comment at the top of the file says, that this is _"Liveness checks as developed by Benoit Boissinot, Fabrice Rastello and myself (Hack)"_,
so I assume this is the algorithm from ["Fast liveness checking for ssa-form programs"](https://doi.org/10.1145/1356058.1356064).

I love that the top comment also says: _"This algo has one core routine check_live_end_internal()"_, but that there is no such routine in the whole repository.
Of course, comments and code getting out of sync happens easily, if one isn't careful.
Anyway, I find C code with _C style identifiers_ (e.g., `lv_chk_bl_xxx`) at times frustrating to read...

## SPIRV-Tools

Uses loop nesting forest based approach:

[test](https://github.com/KhronosGroup/SPIRV-Tools/blob/de69f32e8962ecc4e604bfc125da41d00a7a1ca8/source/opt/register_pressure.h)

[A Non-iterative Data-Flow Algorithm for Computing Liveness Sets in Strict SSA Programs](https://link.springer.com/chapter/10.1007%2F978-3-642-25318-8_13)
..or the research report above.

# GHC Strangeness

Some weirdness regarding physical registers I observed in the old liveness analysis:

    {% raw %}
    # entryIds         = [c4nm]
    # liveVRegsOnEntry = [(c4nm, {%r4, %r14}), (c4np, {%r4, %r14}),
                          (c4nq, {%r4, %r14})]
    # liveSlotsOnEntry = LM (UM (fromList []))

    c4nm:
            addq $24,%r12
                 
            cmpq 856(%r13),%r12
                 
            ja _blk_c4nq
                 
            jmp _blk_c4np
                # r_dying: %r4 %r14
    {% endraw %}


[^1]: I've seen the terms "live range" and "live interval" used synonymously, but most papers and implementations
    seem to use the notion that "a live range consists of live intervals" and I'm also using it in that sense.

[^2]: The use of real regs might be a bit confusing coming from a more "traditional" compiler background.
    Haskell compiles down to the [STG-Machine](https://gitlab.haskell.org/ghc/ghc/-/wikis/commentary/compiler/generated-code) 
    and you can check the source how STG registers are mapped to real regs (e.g., in MachRegs.h and CodeGen.Platform.h).

[^3]: Integer vs. Float/Double registers. E.g., on x86_64, integers (and memory addresses) are put into general purpose registers,
    like %rax, whereas floating point values are put into the SIMD registers, like %xmm0.
    You'll want to build separate interference graphs for different register classes.

[^4]: "[Register pressure](https://en.wikipedia.org/wiki/Instruction_set_architecture#REGISTER-PRESSURE)" basically means "how many real register do I need at this point in the program".
[^5]: Their documentation in general is superb (at least in the codegen part I looked at), so kudos!

*[BB]: Basic Block
*[GHC]: Glasgow Haskell Compiler
*[LR]: Live Range
*[LRs]: Live Ranges
*[NCG]: GHC's Native Code Generator
*[SSA]: Static Single Assignment
*[vreg]: Virtual Register

