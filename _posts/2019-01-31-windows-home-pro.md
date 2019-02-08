---
title: Windows 10 Home like a Pro
categories: coding
tags: windows, docker
---

Contrary to popular belief Windows Home is not that limited compared to the Pro edition.
It's actually possible to install missing elements like Hyper-V or the local group policy editor. Here's how.

## Hyper-V

Open a command prompt as Administrator and run:
```bat
dir /b %SystemRoot%\servicing\Packages\*Hyper-V*.mum > hyper-v.txt
for /f %i in ('findstr /i . hyper-v.txt 2^>nul') do dism /online /norestart /add-package:"%SystemRoot%\servicing\Packages\%i"
del hyper-v.txt
dism /online /enable-feature /featurename:Microsoft-Hyper-V-All /LimitAccess /ALL
```

## gpedit.msc

The dism trick also works for the missing Local Group Policy Editor:

```bat
dir /b %SystemRoot%\servicing\Packages\Microsoft-Windows-GroupPolicy-ClientExtensions-Package~3*.mum > gpedit.txt 
dir /b %SystemRoot%\servicing\Packages\Microsoft-Windows-GroupPolicy-ClientTools-Package~3*.mum >> gpedit.txt

for /f %i in ('findstr /i . gpedit.txt 2^>nul') do dism /online /norestart /add-package:"%SystemRoot%\servicing\Packages\%i" 
```

## Bonus: Docker

The Docker setup insists you can't install it on a Home Edition because of 'missing' Hyper-V.
Temporarily changing `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\EditionID` from `Core` to `Pro` will fix that problem and installation succeeds.
