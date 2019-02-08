---
title: Embedded development with CMake
categories: coding
tags: embedded
---

Using CMake for embedded development is actually not that hard.
The main thing you need is a proper [toolchain file](https://github.com/vpetrigo/arm-cmake-toolchains)
which sets the necessary variables like `CMAKE_C_COMPILER`.

Set it via the cmake commandline or with:
```cmake
set(CMAKE_TOOLCHAIN_FILE arm-gcc-toolchain.cmake)
```

This should also work in Visual Studio when using its builtin CMake support (i.e. not the CMake VS generator).

The rest should be similar to a normal CMake project setup. You may want to adjust the executable suffix:

```cmake
set_target_properties(${ProjectName} PROPERTIES SUFFIX .out)
```

And you'll need to specify your linker script. That's straightforward but to get the dependencies right it's a good idea to create a helper function:

```cmake
function(target_linker_script target script)
	target_link_libraries(${target} PRIVATE -T "${script}")

	# relink on script change
	get_target_property(_savedDeps ${target} LINK_DEPENDS)
	string(APPEND _savedDeps " ${script}")
	set_target_properties(${target} PROPERTIES LINK_DEPENDS $<TARGET_PROPERTY:INCLUDE_DIRECTORIES> ${script})
endfunction()
```

## Convenience functions

This should be enough to successfully create the firmware in ELF form.
Usually you need to convert that to some special format and you also want to get a quick overview of how much memory is used.
The following convenience functions will do the job as post-build commands:

```cmake
function(print_firmware_size target)
	add_custom_command(TARGET ${target} POST_BUILD COMMAND ${CMAKE_SIZE_UTIL} -A "$<TARGET_FILE:${target}>")
endfunction()

function(run_objcopy target suffix type)
	add_custom_command(TARGET ${target} POST_BUILD
		COMMAND ${CMAKE_OBJCOPY} -v -O${type} "$<TARGET_FILE:${target}>" "$<TARGET_FILE_DIR:${target}>/${target}${suffix}"
	)
endfunction()
```

Here's my full [template](https://gist.github.com/Trass3r/3b70ee33d9b1319bbe78de6bdb09c9dc) showing how all those functions can be used.
