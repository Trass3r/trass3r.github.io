---
title: Improving code readability with clang-tidy
categories: coding
tags: cpp
---

## Introduction

Ever had to deal with typical legacy pre-C99 code declaring all variables at function start? It's no piece of cake.

The scope in which they are actually used is not obvious and it's bug-prone on top as they could be modified later by accident. You need to scan the whole function and build up a huge context in your mind just to figure out what's going on. The compiler has to do the same thing of course to optimize stack usage which unnecessarily adds to the compilation time.

So I finally decided to test if clang could help with this. Here is an example of the AST for a simple function: <https://godbolt.org/z/BYj86H>

The following AST matcher can be used in clang-query to find all uninitialized function variables and their usages (DeclRefExpr): 

```c
let localUndefinedVar varDecl(unless(hasInitializer(anything())), unless(isInstantiated()), hasLocalStorage(), unless(parmVarDecl()), isDefinition())
let refToUninitVar declRefExpr(hasDeclaration(localUndefinedVar))
```

The next step would be to walk the AST - especially the scopes which are represented by compound statements - and find the [lowest common ancestor](https://en.wikipedia.org/wiki/Lowest_common_ancestor) of all variable references. That should yield the target location for the variable declaration.

But of course reality is more complicated. There are scopes which are not represented by a CompoundStmt like `if` without braces or your usual `case` statement which wouldn't even allow declarations without additional braces. Also you do want to merge the declaration and its initialization (e.g. move it into a for loop init-statement).

## readability-localizing-variables

It was at this point that I found a pre-existing [project](https://github.com/piotrdz/clang-tools-extra/commits/master) mentioned on the [mailing list](https://lists.llvm.org/pipermail/cfe-dev/2016-February/047164.html) in 2016. It's a pity this got abandoned since it was quite advanced already.

I [ported](https://github.com/llvm/llvm-project/compare/master...Trass3r:localize) it to the current master and gave it a try.
It actually works quite well for most basic cases but still has a few rough edges some of which could be addressed by drawing inspiration from the [readability-isolate-declaration](https://clang.llvm.org/extra/clang-tidy/checks/readability-isolate-declaration.html) check:

- It can't deal with multiple declarations in one statement properly. This can be mitigated by running `clang-tidy -checks=-*,readability-isolate-declaration ...` first.
- It didn't preserve the original declaration wording, e.g. if macros are present.
- When moving multiple variables into the same new location they seem to step on each other's feet.
  The isolate declarations check aggregates the replacements and then creates only one fix-it hint.
- Relatedly, when moving variables into a switch case scope it correctly creates braces if they didn't exist before (awesome). But if multiple variables are moved it somehow creates a second nested scope and produces invalid code.
- It doesn't support duplicating variables like a loop index variables that is re-used in several loops.
- Sometimes it fails to find the innermost scope.
- It doesn't work properly with gcc inline asm blocks.
- Comments belonging to a declaration aren't moved which can create confusion.
- Formatting could be better. Sometimes empty lines are left behind.

I added the clang-tidy [binary](https://github.com/Trass3r/llvm-project/releases/tag/v1) I used for testing.
The command-line should be something like `clang-tidy -checks=-*,readability-localizing-variables --fix --format-style=file test.c --`.
