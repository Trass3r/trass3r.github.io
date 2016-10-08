---
layout:     post
title:      minimal VS2013 platform toolkit for VS2015
date:       2015-08-20 23:31:18
summary:    minimal VS2013 platform toolkit for VS2015
categories: "Visual Studio"
thumbnail: cogs
tags:
 - VS
---

So following http://stackoverflow.com/a/30104415 and
assuming a VS2015 installation do the following:

```bat
cd D:\packages
for %X in (vc_compilerCore86 vc_compilerCore86res vc_compilerx64x86res vc_compilercore vc_compilerx64nat vc_compilerx64natres vc_compilerx64x86 vc_librarycore vc_librarycore86) do msiexec /a J:\packages\%X\%X.msi TARGETDIR="<path>" 
msiexec /a J:\packages\vc_libraryDesktop\x64\vc_LibraryDesktopX64.msi TARGETDIR="<path>" 
msiexec /a J:\packages\vc_libraryDesktop\x86\vc_LibraryDesktopX86.msi TARGETDIR="<path>" 
xcopy /S "<path>\Program Files\MSBuild\Microsoft.Cpp\v4.0\V120" "C:\Program Files (x86)\MSBuild\Microsoft.Cpp\v4.0"
```

```registry
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\Microsoft\VisualStudio\12.0\Setup\VC]
"ProductDir"="<path>\\Program Files\\Microsoft Visual Studio 12.0\\VC\\"

[HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\Microsoft\VisualStudio\12.0\Setup\VS]
"ProductDir"="<path>\\Program Files\\Microsoft Visual Studio 12.0\\"
```
