---
layout: post
title: GHC optimizing a "Programming Horror"
tags:
  - Haskell
  - GHC
  - compilers
  - optimizations
  - programming
---

# One Day on Twitter...

...I saw this in my feed:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">fun post showing how LLVM optimizes a very bad is-even function <a href="https://t.co/t8cH8W0NOI">https://t.co/t8cH8W0NOI</a> <a href="https://t.co/l2Nx7PKUJX">pic.twitter.com/l2Nx7PKUJX</a></p>&mdash; John Regehr (@johnregehr) <a href="https://twitter.com/johnregehr/status/1428027708450099202?ref_src=twsrc%5Etfw">August 18, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

And also a reply showing that rustc (based on LLVM) can optimize that that into a 3 (x86_64) instruction sequence.

For those who didn't read the [blog post](https://blog.matthieud.me/2020/exploring-clang-llvm-optimization-on-programming-horror/), 
it's about Clang/LLVM magically optimizing a "stupid" isEven function.
That function basically counts from zero to the input number, starting with `even = true` and
negating that flag on each iteration.
Note that this function is not correctly handling negative numbers and the compiler sees that
as undefined behavior.

There are two things going on here:

1. The compiler "recognizing" the intent (is input "even", i.e., `n rem 2 == 0`, where "rem" is the remainder") and reducing the code. 
2. Emitting optimized machine code

What I mean with 2. is, that we don't actually need a division[^1] here - we only need to look at the least significant bit (LSB). 
If the LSB is "1", the number is not even, if it is "0" even.

So naturally, I was wondering what GHC would do with such a function.

# My Horrible "isEven"

This is how I would write the function from the posting in Haskell:

~~~ haskell
isEven :: Int -> Bool
isEven n = foldl' (\t _ -> not t) True [0, 1..n-1]
~~~

This function is not undefined for negative numbers - it's simply incorrect, because it will always return `True`.

Unfortunately GHC does **not** [reduce this](https://gcc.godbolt.org/z/EEW7czo4z) down to three instructions (not even with -fllvm).

The version with a recursive helper function is even worse.
I didn't really expect it to work such magic anyway.
Truth be told, when I read that tweet, my first instinct was to check the generated code for a
_sensible_ _isEven_ function.

# A sensible "isEven"

I think, one might write a function like this:

~~~ haskell
isEven :: Int -> Bool
isEven n = n `rem` 2 == 0 
~~~

That even works for negative numbers.

But looking at the generated assembly, I can see that the general optimized routine for division was emitted (using bit shifts, see footnote 1[^1]):

~~~ nasm
        movq 7(%rbx),%rax
        movq %rax,%rbx
        shrq $63,%rbx
        movq %rax,%rcx
        addq %rbx,%rcx
        andq $-2,%rcx
        subq %rcx,%rax
        testq %rax,%rax
~~~

You can see the full output [here](https://gcc.godbolt.org/z/7MnrMhfrf).

But all we need is:

~~~ nasm
        movq 7(%rbx),%rax # Load 'n' into register
    	testb $1,%al      # Test if LSB equals to 1
        # Some jump or return the result
~~~

# Digging into GHC

When you look into the generated STG code (with `-ddump-stg`), you can see this:

~~~
        case remInt# [ipv_s1uz 2#] of {
          __DEFAULT -> GHC.Types.False [];
          0# -> GHC.Types.True [];
        };
~~~

I.e., a case expression on the result of `remInt#`.
If you are very familiar with GHC, you will recognize the "hash" symbol ('#') as an indication for dealing with [unboxed values](https://ghc.gitlab.haskell.org/ghc/doc/users_guide/exts/primitives.html).

`remInt#` is actually a "primop" - a "primitive operation" built into GHC.

You can grep around the GHC source a bit to find that `remInt#` is represented by `IntRemOp` and trace that down to `MO_S_Rem` for signed integers and `MO_U_Rem` for unsigned integers.

We can find the optimization we are looking for in [GHC/Cmm/Opt.hs](https://gitlab.haskell.org/ghc/ghc/-/blob/ad28ae41206e3a8b483b067a8e3333df445010b6/compiler/GHC/Cmm/Opt.hs#L357),
where constant folding etc. is performed for these machine operations ("MachOps", that is what the "MO" in "MO_S_Rem" stands for: Machine Operation - Signed Remainder).

Long story short: the unsigned case, "MO_U_Rem", works as expected.
For powers of 2, it boils down to `x rem y == AND x,(n-1)`.
But "MO_S_Rem" handles the more generic case and uses a helper function for "QuotRem",
i.e., the operation to get the quotient **and** the remainder.
That function always performs the bit shift etc.

I simply created a special case for the divisor being exactly 2.
You can find my [MR here](https://gitlab.haskell.org/ghc/ghc/-/merge_requests/6403).

Now, will this make your program magically super fast all of a sudden?
No, definitely not. Unless you are doing `rem 2` **a lot** in hot loops.
But, you know, baby steps.

# Conclusion

Performing optimizations on a program at large is **hard**.
I can't fix that on my own in an hour after seeing some tweet,
but I **can** add some tiny arithmetic optimizations :D

If you spot any mistakes, omissions or want to provide feedback, comments, questions - you can reach me on [Twitter](https://twitter.com/cptwunderlich), or via mail with my username at gmail.

![readers](https://visitor-badge.glitch.me/badge?page_id=cptwunderlich.opthorror.iseven2021)
        
[^1]: Divisions are extremely slow on CPUs and a common optimization for divisions by powers of two is to use [bit shifting instead](https://en.wikipedia.org/wiki/Division_by_two#Binary).
