---
title: Reducing C/C++ code
categories: coding
---

Once in a while I run into internal compiler errors.
To provide a nice minimal test case with the bug report I still use [DustMite](https://github.com/CyberShadow/DustMite/wiki).
It was actually written for D code but works good enough for C++ as well and is easier to setup than more sophisticated tools like [C-Reduce](http://embed.cs.utah.edu/creduce/).
Just get the [LDC](https://github.com/ldc-developers/ldc/releases/) compiler and compile DustMite with `ldc2 -O3 -release -g dustmite.d splitter.d`.

Then put all your test sources in one folder.
Sometimes it makes sense to get a preprocessed version of your code first (`-E` or `/EP` option).

Create a test script outside this folder with the test commandline that reproduces the error:
```sh
cl -c -O1 test.c 2>&1 | "C:\Program Files\Git\usr\bin\grep.exe" -q "internal error"
```
Here I'm using the grep utility Git for Windows provides.

Now we can let DustMite work its magic: 
`dustmite --split "*.{c,cpp,h,hpp}:d" test ..\make.bat`.
Make sure to run it from a proper environment, like the Native Tools Command Prompt for msvc.

It will take a while but when it's all done you should have a reduced test case.
Some post-processing may still be necessary as DustMite only uses a simple splitter and can't do advanced transformations that would require knowledge about C++.