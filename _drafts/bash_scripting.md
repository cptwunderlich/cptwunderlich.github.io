---
layout: post
title: A Unit Test Framework in Bash
tags:
  - bash
  - programming
#cover_url: https://s2.banana.moe/unsplash_colle/romain-vignes-53940.jpg
---

# bash scripting a unit testing framework

## Introduction

A while ago, I created a script at work. Creating a new project for a client used to be arduous task, including several steps and thus opportunities for mistakes. A script to automate this, asking the user a few simple questions, was the perfect solution.

But sure enough, the script became more and more complex. New features were added, to create new projects for our different products, using different databases, an update mechanism, a settings file, so the user doesn't have to edit the script itself, etc. etc.
At some point, the script was long and complicated and I decided to break it up. Because testing all the possible combinations of choices to make (what project, what database, etc.), unit testing the script seemed like a good idea. Since it was "just a simple bash cript", I started writing some tests manually.
After 9 hours of implementing my own unit testing framework, I realized, this is getting out of hand. But since it was a lot of fun and I learned a lot, I'd like to continue it as a side project. Here, I'd like to document some of the neat things I've learned about bash scripting and I want to show parts of the "evolution" of the code (including interesting dead-ends).

Mind you, this is not intended as an introduction to bash scritping.

## functions
Functions in bash are a bit weird. They aren't quite like their relatives in "normal" programming languages.

## Splitting a string

As a software developer, I'm used to working with languages which give me basic operations on strings. Searching for ways to split a string surfaced a few different approaches. Notably:

    IFS="." read -ra result <<< "$in_var"

This will split the text in $in_var, using the delimiter IFS (Internal Field Separator - in this case a dot character), by using the built-in "read". The option "-a" means: read input as array, starting with index 0. "-r" means: don't treat backslash as escape character. See the manual [^1]

Well, I didn't like that very much. It's so much to type and using `read` doesn't really communicate what I want to do. So I hid the line inside a function "strsplit".

But turns out, there is another way. It's even less descriptive, but at least terse. You can use variable expansion/substring replacement[^2]:

    result=(${my_input//\./ })

So, what's going on here? The parens `(...)` are the array constructor[^3]. Withing our variable substitution `${}`, we first have the name of our input variable. The double slash '//' means global substitution, instead of just once. Then comes the pattern to be replaced and after the slash the replacement. Since everything separated by spaces within the parens will become an array element, using this on strings that may contain spaces results in a lot of trouble.
Replacing `1 1.2 2.3 3` will not give you `("1 1", "2 2", "3 3")`, but instead `("1", "1", "2", "2"...)` etc.
In that case, you'll have to use the approach using `read`.

## Enumerating all functions

This one was interesting. I wanted to enumerate all test functions in my script, so they can be called automatically, just like in many "real" unit testing frameworks (although sometimes you need an annotation, etc.).

Now this is really a "bash-ism" and not portable to other shells. While the more standard `declare -f` will give you all defined functions including their bodies, bash's `declare -F` will give you an output like "declare -f quote".

For example, finding all function names starting with "test_" can be done with
<TODO>

## Writing an assert function

<TODO>

## Mocking functions

### Basic mocking

It is very simple. You can just overwrite commands as you wish.

    mock_exit()
    {
        exit()
        {
            test_debug "Exit called"
        }
    }
    
    unmock_exit()
    {
      unset -f exit
    }

Now you can overwrite exit by calling `mock_exit` and remove it with `unmock_exit`.

### Mocking read

This, I thought at first, is a bit more tricky.

<TODO>

## Log functions

<TODO>

[^1]: http://tldp.org/LDP/Bash-Beginners-Guide/html/sect_08_02.html
[^2]: http://www.tldp.org/LDP/abs/html/parameter-substitution.html#EXPREPL1
[^3]: http://tldp.org/LDP/Bash-Beginners-Guide/html/sect_10_02.html
