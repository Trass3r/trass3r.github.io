---
title: Advanced PlatformIO
categories: coding
tags: embedded
---

Arduino has really done a great job at opening up the embedded world to more people.
But its "IDE" is barely usable for anything more than a Hello World.
I came across PlatformIO and decided to give it a try. It supports a lot of platforms and also various frameworks for each of them.

It is straightforward to get a basic setup.
After setting up VSCode in the nice [portable mode](https://code.visualstudio.com/docs/editor/portable) (unfortunately it's harder to get PlatformIO portable as well) and installing the PIO extension you can use the wizard to generate a basic Blinky project.

### Advanced usage

Now let's get our hands dirty.

First we enable verbose mode to see what's going on.
Since the `pio run` invocation is hard-coded we set [force_verbose](http://docs.platformio.org/en/latest/userguide/cmd_settings.html#setting-force-verbose) in the PIO console:
`pio settings set force_verbose true`

This prints all the compiler invocations and also the output binary sections in every detail.

Let's tune the compiler warnings for our own code since the default `-Wall` is barely enough. This can be done in the main ini file:

```ini
src_build_flags = -Werror -Wextra -Wno-unused-parameter -Wduplicated-branches -Wduplicated-cond -Wlogical-op -Wnull-dereference -Wconversion -Wfloat-conversion -Wsign-conversion -Wshadow -Wsuggest-final-types
```

But oops, some of these warnings aren't even implemented since an ancient version of gcc is in use (5.4).
There have been major improvements since then, especially for LTO (see Jan Hubička's [blog](https://hubicka.blogspot.com/2018/12/firefox-64-built-with-gcc-and-clang.html?showComment=1545127908297#c7585538255098898297) for interesting details on that).
Also `-isystem` is broken for C++ in arm-gcc up to version 8.

A nice feature is that the `platform` configuration parameter also accepts a github url.
So we can use a fork of the platform repo to make any desired adjustments:

```ini
platform = https://github.com/[username]/platform-teensy.git
```

Now, VSCode does not seem to pick up gcc errors and warnings by default, so let's fix this by selecting the Build task in Terminal->Configure Tasks... and set up a matcher:
```json
	"problemMatcher": {
		"base": "$gcc",
		"fileLocation": ["absolute"]
	}
```

That's about the extent of changes you can do in the ini file.

### More advanced usage

As soon as you need something just slightly more advanced like *compiler* flags to be used only in the link stage (not talking about `-Wl` linker flags) you need to resort to a separate python script. This works since PIO uses SCons under the hood:
```ini
extra_scripts = extra_script.py
```

But that is useful for a lot of things so it's always good to have one for non-trivial projects:

```py
Import("env", "projenv")

env.Append(
  LINKFLAGS=[
    "-save-temps",
  ]
)
```

In there you can do all sorts of fancy things like trying to persuade the system to properly use `-isystem`:

```py
projenv.Append(
    CXXFLAGS = [v for e in env["CPPPATH"] for v in ("-isystem", e)]
)
```

or printing the biggest code symbols in your firmware:
```py
env.AddPostAction("buildprog",
    env.Action("arm-none-eabi-nm -ClS --radix=d --size-sort $BUILD_DIR/${PROGNAME}.elf | tail -n10"))
```
```sh
00002540  00000310 t delay                       pins_teensy.c:1207
00007196  00000318 T main                        main.cpp:33
00001940  00000356 t systick_isr                 EventResponder.cpp:338
00000000  00000444 T _VectorsFlash               kinetis.h:5858
536838656 00000444 B _VectorsRam
00007516  00000558 t ResetHandler                mk20dx128.c:690
00002852  00000786 t _init_Teensyduino_internal_ pins_teensy.c:520
536839100 00000864 B usb_buffer_memory
00003728  00003380 t usb_isr                     usb_dev.c:863
```

You could even try to replace gcc with clang but that's a bit more involved as you would need to setup all the include paths etc.
But here's a start:

```py
projenv['CC'] = 'clang'
projenv['CXX'] = 'clang++'
env['CC'] = 'clang'
env['CXX'] = 'clang++'
env['LINK'] = 'arm-none-eabi-g++'
```

### Conclusion

Once it's all set up properly PlatformIO is quite usable as an embedded IDE.
Of course for more professional use you may want to reach for CMake directly.