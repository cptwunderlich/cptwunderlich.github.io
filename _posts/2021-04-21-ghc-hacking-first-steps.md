---
layout: post
title: Hacking on GHC - First Steps
tags:
  - Haskell
  - GHC
  - compilers
  - programming
---

# Hacking on GHC - Getting Started

## Intro

For quite a long time I've been interested in compilers and to deepen my superficial knowledge of Haskell. Since I'm a hands-on learner, the latter requires an interesting project.
Getting started with contributing to an existing compiler on the other hand, is a daunting task, as these are usually large and complex projects.

As I was searching for a suitable project, I found [AndreasK's blog post on GHC backends](https://andreaspk.github.io/posts/2019-08-25-Opinion%20piece%20on%20GHC%20backends.html). Digging deeper, I read his article on GHC's [Graph Coloring Register Allocator](https://en.wikipedia.org/wiki/Register_allocation#Graph-coloring_allocation) and a ticket with [improvement ideas for -fregs-graph](https://gitlab.haskell.org/ghc/ghc/-/issues/16243).

Since I'm very interested in code generation and I've learned a bit about register allocation before, I thought that'd be a perfect place to start!

You can find some other (older) blog posts on how to get started in the [GHC Wiki](https://gitlab.haskell.org/ghc/ghc/-/wikis/contributing#advice), but I encountered many issues along the way.
Especially since some things are in a bit of a transition phase, e.g., the Make-base build vs. Hadrian-build.
That's why I've decided to write about my experience - maybe someone will find the solution to a problem here or get motivated to start hacking on GHC!

Note that this is written from a Ubuntu Linux on x86_64 perspective.

## Set-Up

First things first. You need a very recent version of GHC and other dependencies to build the latest GHC. The packages in your distribution's repository may be way to old (GHC 9.1 requires at least 8.10.x, AFAIK).
Luckily, there is a [PPA by Herbert V. Riedel](https://launchpad.net/~hvr/+archive/ubuntu/ghc) - the page also links to sources for Debian Stretch, Mac OS and WSL repositories.
Otherwise you might want to look at [GHC Up](https://www.haskell.org/ghcup/).

The following commands will add the PPA and install everything needed from the repositories (note that I'm not choosing GHC 9, as there are still some things incompatible with it, e.g., Haskell Language Server):

    sudo add-apt-repository ppa:hvr/ghc
    sudo apt update
    sudo apt install build-essential autoconf git ghc-8.10.4 cabal-install-3.4

And now to checkout out the multi-module Git project into the current folder:

    git clone --recurse-submodules https://gitlab.haskell.org/ghc/ghc.git

Add GHC/Cabal to your path:

    echo -e '# For GHC\nPATH="/opt/ghc/bin:$PATH"' >> ~/.profile
    source ~/.profile

Now we need to install some Haskell packages, namely Happy & Alex (lexer and parser generators):

    cabal update
    cabal install happy
    cabal install alex
    # Add '/home/<username>/.cabal/bin' to your ~/.profile
  
Don't forget to add the path to .cabal/bin to your PATH in .profile and source it!
Check the versions to make sure everything is working:

    cabal --version
    ghc --version
    alex --version
    happy --version

For building the docs you'll need Sphinx:

    sudo apt install python3-sphinx

If you want to use/work on the LLVM-backend, you'll also need to install the right LLVM development packages.
Check the latest release notes to see which LLVM versions are supported.

### IDE

Now, I know that the "true Scotsmen" use emacs, or _maybe_ vi, but despite having programmed for many years, I never gotten over that initial learning bump. That time investment just doesn't seem justifiable to me. I prefer an IDE or at least a rich code editor.

VS Code is a popular and robust choice for this, so that's what I'm using for Haskell.
Install VS Code, e.g., via the Ubuntu store, or download it from [here](https://code.visualstudio.com/download).

There are plenty of infos out there on how to install and configure VS Code, so I'll stick to the Haskell specifics.
You'll only need to install one extension:
Go to File -> Preferences -> Extensions and search for "Haskell". You'll find "Haskell - Haskell language support powered by...", or alternatively, here the [direct link](https://marketplace.visualstudio.com/items?itemName=haskell.haskell).
This will also install the required extension "Haskell Syntax Highlighting".

I can't stress enough just how great the [Haskell Language Server](https://github.com/haskell/haskell-language-server) is.
When I started, I tried manually compiling _ghcide_ etc. and could barely get it to work. From the first release of Haskell LS, I could see and feel the improvements with every release.
Give it a try, even with emacs or vim.

In case Haskell LS still ends up having difficulties, it can help to hit <kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>P</kbd> and search for "Restart Haskell LSP Server" and run that.

Another very useful command is "Trim Trailing Whitespace" - GHC's CI has a linter that will fail your MR if you have trailing whitespace!
You can run the lint rules locally with `./hadrian/build lint:compiler` or `lint:base`, but you will need `hlint` on your PATH!

Opening the project with VS Code is pretty straight forward - just go to File -> Open Folder and open your checkout directory of GHC.

### HLint

Speaking of [hlint](https://github.com/ndmitchell/hlint) - you can simply install it with cabal.
You might want to add your own .hlint.yaml to a subdirectory in GHC and add or ignore some rules.

### Git

GHC's Git project contains many submodules. Sometimes, when you want to get a branch completely up-to-date with upstream,
you'll want to [update all submodules recursively](https://git-scm.com/book/en/v2/Git-Tools-Submodules#_pulling_upstream_changes_from_the_project_remote):

     git pull --recurse-submodules

## Build & Test

Many of the things I'm writing here can be found on the nifty cheatsheet [ghc.dev](https://ghc.dev/) - make sure to check it out.
One thing I would add though: no need to run `./boot && ./configure`, as Hadrian can do that with the `-c` option.

But I'm getting ahead of myself. Let's start with the basics and change into the GHC directory with your favorite terminal emulator (e.g., in VS Code).
Be aware that building GHC takes quite some computing resources and time, depending on the chosen optimizations.

Make sure to read the [Hadrian README](https://gitlab.haskell.org/ghc/ghc/-/tree/master/hadrian).
Basically, there are different build "Flavours" with different optimizations or additional configuration (e.g., profiling builds).
You can find an overview of the flavours [here](https://gitlab.haskell.org/ghc/ghc/-/blob/master/hadrian/doc/flavours.md).
GHC compilation is done in stages. Confusingly, sometimes you'll see _stage0_ and _stage1_, other times it starts with _stage1_. Anyway, the first stage is the bootstrap compiler, compiled with the GHC on your path. The next stage is compiled with the bootstrap compiler, containing all the latest GHC changes.

Running `./hadrian/build -cj --flavour=Quick` will run the _configure_ step (`-c`), build in parallel (`-j`) and produce a "quick" build (`-O0`).
Note that you don't need to pass `-c` every time and that you can use `--freeze1` to avoid rebuilding the bootstrap compiler (if you haven't changed anything relevant).

Hadrian has a clean command (`hadrian/build clean`), there is also `hadrian/ghci` to load GHC into a GHCi session and you can run the hlint rules with `hadrian/build lint:base` and `hadrian/build lint:compiler`.

After your build has finished, verify it by running `./_build/stage1/bin/ghc --version`.

### Read

Reading the READMEs of [GHC](https://gitlab.haskell.org/ghc/ghc), [Hadrian](https://gitlab.haskell.org/ghc/ghc/-/tree/master/hadrian) and [ghc.dev](https://ghc.dev/) gives you some quick and hands-on infos and commands.

To get the big picture view of GHC, there are also quite a few resources around!

[Takenobu's GHC reading guide ](https://github.com/takenobu-hs/haskell-ghc-reading-guide) is highly recommended, as well as their [GHC illustrated](https://github.com/takenobu-hs/haskell-ghc-illustrated).

Stephen Diehl also wrote about GHC, [this is the first of a three-part](https://www.stephendiehl.com/posts/ghc_01.html) blog post series.
Be aware that some of the folder/module structures have changed, since that was written in 2016.

The Haskell Wiki is a double edges sword. While there is lots of extremely useful information, much of it is out-of-date and sometimes it's hard to tell them apart.
Either way, you'll want to look at the [GHC Commentary](https://gitlab.haskell.org/ghc/ghc/-/wikis/commentary) and the [reading list](https://gitlab.haskell.org/ghc/ghc/-/wikis/reading-list), which contains papers and other theoretical sources.

## Get connected!

Now that you can compile GHC and have some reading material, you want to get in touch with the community.

Get yourself a [Gitlab account](https://gitlab.haskell.org/), so you can participate on tickets and merge requests.
For the latter, you might want to create a fork of GHC for your account and add that as a [git remote](https://git-scm.com/book/en/v2/Git-Basics-Working-with-Remotes) locally.

Subscribe to the [GHC mailing list](http://www.haskell.org/mailman/listinfo/ghc-devs), to stay informed and get new ideas for projects!

AFAIK, most of the communication happens on IRC, so pop in at [#GHC on freenode](https://kiwiirc.com/nextclient/chat.freenode.net/#ghc) - I have to say, people on their have been very friendly and extremely helpful!
There is also a [#Haskell-docs](https://kiwiirc.com/nextclient/chat.freenode.net/#Haskell-docs) channel, where you can find support for your fervor to make Haskell documentation _excellent_.

You might also want to check out this Wiki page for [new contributors](https://gitlab.haskell.org/ghc/ghc/-/wikis/contributing#advice).

## What to work on?

This is probably the hardest part. It very much depends on your knowledge and interests.
If you have an area of interest, search for that on the bug tracker or ask around on IRC.

Making GHC compile times **faster**, even by a tiny bit, or reducing memory consumption, will always be welcomed by the community.
If you have any experience with LLVM, I'm sure that there are also lots of ways to improve and speed-up the LLVM-backend.

Documentation is also _vital_ and you can contribute by fixing mistakes, clarifying or adding descriptions.
One really simple and fast way to dip your toes into GHC development is this ticket on documenting yet [undocumented command line flags](https://gitlab.haskell.org/ghc/ghc/-/issues/18641).
All you need is `grep` and some detective work.

## Developer's Life

Usually, developing not only means writing code and compiling it, but debugging, profiling, benchmarking.
I wanted to touch on these topics here too, briefly, to help you start out.

### Debugging

Let's say you found yourself a project and you're working hard at it.
Sooner or later, you'll need to debug things. Unfortunately, there aren't an awful lot of resources on this.

As mentioned above, there is a way to load GHC into GHCi. To be honest, I haven't used this yet, as it didn't seem to practical for working on the code generator.

This [reference of debug flags](https://downloads.haskell.org/~ghc/9.0.1/docs/html/users_guide/flags.html#compiler-debugging-options) got quite a lot of use though.
You can dump intermediate steps along the way to figure out, where things went awry.

In lieu of a proper debugger, GHC.Driver.Ppr with it's `pprTrace` and friends - where `ppr` stands for "pretty print". GHC uses the `SDoc` type for string representations of data structures. I.e., this is the good old "printf-debugging".

When you have to debug a crashing output binary, you'll also want to use GDB and possibly the reverse debugger [rr](https://rr-project.org/).

In some cases I have used ad-hoc scripts to visualize or search for interesting data in my traces.
Banging my head against the desk and crying profusely have not proven efficient techniques, despite my repeated tries.

### Profiling

To enable [cost center profiling](https://downloads.haskell.org/~ghc/9.0.1/docs/html/users_guide/profiling.html), you'll have to compile GHC and libs with profiling support.
Hadrian offers [Flavour Transformers](https://gitlab.haskell.org/ghc/ghc/-/blob/master/hadrian/doc/flavours.md#flavour-transformers) for that.
Note the remark for the `profiled_ghc` transformer - you'll want to also use `no_dynamic_ghc` and the flavour _"Quick"_ won't cut it.
E.g., use something like:

    ./hadrian/build -j --flavour="default+profiled_ghc+no_dynamic_ghc"

You can then use the RTS flags, like [+RTX -xc](https://downloads.haskell.org/~ghc/9.0.1/docs/html/users_guide/runtime_control.html#rts-flag--xc).

Check out the above linked GHC documentation for more on profiling and make sure to install Matthew Pickering's [eventlog2html](https://mpickering.github.io/eventlog2html/) for some graphical representation of profiling eventlogs.

Here just quickly two recipes I found myself using quite a bit:

Since GHC is quite big and I was working on a single module, it made sense to restrict the output to that module, using `-hm`.
The `-hy` option will give you a heap profile broken down by type and `-hc` breaks down time profiles by cost-center stack.

Here the type example:

    ghc -O2 -dcmm-lint -dasm-lint -ddump-asm-ssa Main.hs +RTS -hm"GHC.CmmToAsm.SSA" -hy -l-au

And cost-centers:

    ghc -O2 -dcmm-lint -dasm-lint -ddump-asm-ssa Main.hs +RTS -hm"GHC.CmmToAsm.SSA" -hc -l-au

Using eventlog2html to visualize it:

    /home/ben/.cabal/bin/eventlog2html ghc.eventlog --bands 100 --include-trace-events && firefox ghc.eventlog.html

### Benchmarking

GHC's [nofib benchmark suite](https://gitlab.haskell.org/ghc/nofib) come with a whole slew of of benchmarks.

While there is a submodule in the GHC tree, I've been advised to create a separate checkout of nofib.
Like GHC, it has an old Make-based build and a new Shake-based one. I'll only describe the latter here.

To start a full run of all benchmarks, simply use the `nofib-run` tool, like this:

    # Optional `cabal update`
    cabal new-run -- nofib-run --compiler=/absolute/path/to/your/ghc --compiler-arg="-O2" --output=my-test

Note that the path to your GHC, used for compiling the tests, has to be an absolute path!
You can add any number of compiler args using the `--compiler-arg` flag.
`--output` is the name of the result folder.
To run tests "t"-times, use the `-t` switch.
There is a `-j` option for parallel runs, but this only makes sense if you want to stress test your compiler
to find e.g. crashes, as parallelization makes runtimes less deterministic and reproducible.
Refer to `cabal new-run -- nofib-run --help` for other options.

The Make-build only runs certain benchmark groups by default, AFAIK omitting `gc` and some other.
Using Shake, it will run **all** of them though. I haven't figured out how to get the same behavior for both,
but you can simply specify a list of tests to run.

All tests come with three different "speed" settings: SLOW, NORMAL, FAST
This is supposed to enable benchmarking with different runtimes and also help normalize runtimes across benchmarks.
That being said, runtimes have become quite disparate...

Results from different runs can be compared with `nofib-compare`, e.g., to compare baseline to your optimization:

    cabal new-run -- nofib-compare ./_make/baseline ./_make/my_opt

`nofib-compare` also supports different output formats (CSV, ascii, markdown, latex).

For some serious measurements, you might want to install `perf`, which should be part of the `linux-tools-generic` package (or the specific one for your kernel).
That way, you get data from your CPUs hardware counters, which is more "objective" than pure wall-time.
To (temporarily) allow access to more CPU events, you have to allow that: `echo "0" | sudo tee /proc/sys/kernel/perf_event_paranoid`
Cache misses are not part of the default event set used by nofib - for some reason - but you can include them explicitly with these
arguments to nofib-run: `--perf --perf-arg="-ecache-misses,cycles,instructions"`

Benchmarking is generally an arcane art. You'll want to limit any interferences, like programs running in background
and you'll **definitely** want to deactivate CPU frequency scaling and on-demand overclocking ("turbo boost") - otherwise you'll never have reproducible results.

To set the "performance" CPU governor:

    sudo cpupower frequency-set -g performance
    
(Temporarily) deactivating "turbo boost":

    echo "0" | sudo tee /sys/devices/system/cpu/cpufreq/boost

(only tested this with my AMD chip, but should work for all CPUs. [See docs](https://www.kernel.org/doc/Documentation/cpu-freq/boost.txt))

Last but not least, nofib may sometimes break with the latest changes from GHC head.
Currently I can't use the latest GHC release (9.0.1) for nofib-run, so I have to use 8.10.x.
Sometimes packages may become incompatible, but there is a hackage overlay with pre-releases, called [head.hackage](https://gitlab.haskell.org/ghc/head.hackage).

## Conclusion

I hope this provided a gentle introduction to GHC development for you all and that some of you will contribute to this great project.
Maybe this saves someone time that I spent scratching my head.

If you spot any mistakes, omissions or want to provide feedback, comments, questions - you can reach me on [Twitter](https://twitter.com/cptwunderlich), or via mail with my username at gmail.

