---
title: "Shared Library Hot Reloading (Part 1)"
date: 2025-10-13T14:40:57-04:00
draft: true
toc: true
images:
tags:
  - cpp
---

A long, long time ago, when watching [Handmade Hero](https://hero.handmade.network/), somewhere in
the first 10-20 or so episodes, Casey shows how to do code hot reloading. That is, having your
program running, change your code, recompile and have the application immediatelly pick up the
changes.

{{<admonition note>}}
Unreal has something similar with Live++. But that is not the same thing. Live++ is a much more
sophisticated, black magic approach than the one described here.
{{</admonition>}}

Until that time, I always thought it was some voodoo, dark magic that was beyond the comprehension
of mere mortals like me. I was right (somewhat) on the first part, but not on the second. And while
I have been using the  technique successfully for the longest part, there are still some edge
corners that I've yet to master.

## The Setup

To better comprehend the technique, it's important to have a bit of understanding about how shared
libraries work and are loaded by OS. I will use the Windows/DLL nomenclature, but the process is
relatively similar on Linux.

When programs are compiled, the code is divided into modules (or libraries). One module will be the
program itself, where the `main` function is. But outside specific environments, like embedded
systems, no program is complete by itself. It requires other "external" code for it to function,
some provided by third party libraries (eg. SDL, Boost, etc.) and other comes from the OS.

When a program is being compiled, it has to declare which libraries it needs. This is sometimes
referred as "compiling or linking against" that library, the latter being more correct. Linking to a
library can be done statically or dynamically.

### Static Linking

Static linking means that the compiled code of that library will be "bundled in" the main program.
Outside from some metadata, from the point of view of the OS, it's as if that code was written for
that program all along.

This doesn't mean that the static library is compiled at the same time as the program, though it
often is. In either way, the compiler needs the headers and either the source or the precompiled
library at the moment of compilation, otherwise it cannot "bundle it in".

### Dynamic Linking

Dynamic linking is different in the sense that instead of bundling the code in, the compiler creates
"stubs" for the functions and symbols needed when using the library, and associated metadata to
track. The idea is that this library will be loaded _at runtime (dynamically)_, and the stubs
completed then. At compile time, the compiler needs the headers and some symbol metadata to be to
correctly create the stubs, but it does not need the generated code.On runtime, the OS will find the
shared libraries and fill those stubs.

A separate compilation step, which does not (and normally is not) done in the same machine/time,
creates the shared library: the actual .dll file). They hold a lot of data, but in a gist they are
the machine instructions, data pages (precompiled  read-only data, like tables, would go in there),
plus a lot of metadata.

The most relevant for the dynamic linking, in the case of DLLs, are the `Export Directory`, in which
there is a `Export Address Table`. In a gist, it is a table that maps function names to `Relative
Virtual Addresses`. These are fancy names for offsets from a known location, normally what would be
the load address of the shared library. This is needed because the library does not know where it
will be loaded in memory, so the best it can do is to have an offset that can be "patched" later on.

{{<admonition note>}}
This idea is fairly old (at least 70s), and it comes from optimizations when memory was much more
limited. So instead of each process having a copy of the same library in memory, they could all
refer to the same one and be shared.
{{</admonition>}}

### Static vs Linking

There are pros and cons to each approach, though one big driver for shared libraries, limited
memory, is no longer a big concern. So in many environments, like cloud computing, they compile
everything statically since it's much less error prone to have one big executable rather than
making sure that all the dynamic libraries are correctly deployed.

Having said the above, OSes use dynamic libraries **A LOT**. You can see which modules are loaded on a
process by using a debugger. As an example, these are the loaded modules for a simple, yet
non-trivial program:

The format of the list is:

`(DLL Name, DLL filepath, Mapped Address Begin, Mapped Address End, Associated PDB file, PDB Load result)`

{{<admonition important>}}
No need to read the list, this is just to show how many they are.
{{</admonition>}}

```text
advapi32.dll,C:\Windows\System32\advapi32.dll,7FFC5D0A0000,7FFC5D152000,advapi32.pdb,PdbFail
assimp.dll,C:\Users\crist\_bazel_crist\3m6uftxs\execroot\_main\bazel-out\x64_windows-dbg\bin\kandinsky\assimp.dll,7FFB9BBF0000,7FFB9C40A000,,NoPdb
bcrypt.dll,C:\Windows\System32\bcrypt.dll,7FFC5A370000,7FFC5A396000,bcrypt.pdb,PdbFail
bcryptprimitives.dll,C:\Windows\System32\bcryptprimitives.dll,7FFC5B020000,7FFC5B0B9000,bcryptprimitives.pdb,PdbFail
cfgmgr32.dll,C:\Windows\System32\cfgmgr32.dll,7FFC5A0D0000,7FFC5A127000,cfgmgr32.pdb,PdbFail
clbcatq.dll,C:\Windows\System32\clbcatq.dll,7FFC5C4F0000,7FFC5C598000,CLBCatQ.pdb,PdbFail
combase.dll,C:\Windows\System32\combase.dll,7FFC5BF40000,7FFC5C2C4000,combase.pdb,PdbFail
CoreMessaging.dll,C:\Windows\System32\CoreMessaging.dll,7FFC56C20000,7FFC56D46000,CoreMessaging.pdb,PdbFail
CoreUIComponents.dll,C:\Windows\System32\CoreUIComponents.dll,7FFC53920000,7FFC53C03000,CoreUIComponents.pdb,PdbFail
crypt32.dll,C:\Windows\System32\crypt32.dll,7FFC5A520000,7FFC5A697000,crypt32.pdb,PdbFail
cryptbase.dll,C:\Windows\System32\cryptbase.dll,7FFC59B80000,7FFC59B8C000,cryptbase.pdb,PdbFail
cryptnet.dll,C:\Windows\System32\cryptnet.dll,7FFC52B70000,7FFC52BAB000,cryptnet.pdb,PdbFail
cryptsp.dll,C:\Windows\System32\cryptsp.dll,7FFC59B90000,7FFC59BAC000,cryptsp.pdb,PdbFail
DataExchange.dll,C:\Windows\System32\DataExchange.dll,7FFC1C540000,7FFC1C59A000,DataExchange.pdb,PdbFail
dbghelp.dll,C:\Windows\System32\dbghelp.dll,7FFC57C80000,7FFC57EC1000,dbghelp.pdb,PdbFail
devobj.dll,C:\Windows\System32\devobj.dll,7FFC5A130000,7FFC5A15D000,devobj.pdb,PdbFail
directxdatabasehelper.dll,C:\Windows\System32\directxdatabasehelper.dll,7FFC57540000,7FFC575A0000,directxdatabasehelper.pdb,PdbFail
drvstore.dll,C:\Windows\System32\drvstore.dll,7FFC529F0000,7FFC52B6C000,drvstore.pdb,PdbFail
dwmapi.dll,C:\Windows\System32\dwmapi.dll,7FFC575D0000,7FFC57606000,dwmapi.pdb,PdbFail
DXCore.dll,C:\Windows\System32\DXCore.dll,7FFC57620000,7FFC57667000,DXCore.pdb,PdbFail
dxgi.dll,C:\Windows\System32\dxgi.dll,7FFC572B0000,7FFC573E3000,dxgi.pdb,PdbFail
gdi32.dll,C:\Windows\System32\gdi32.dll,7FFC5B120000,7FFC5B14B000,gdi32.pdb,PdbFail
gdi32full.dll,C:\Windows\System32\gdi32full.dll,7FFC5A6A0000,7FFC5A7D2000,gdi32full.pdb,PdbFail
glew32.dll,C:\Users\crist\_bazel_crist\3m6uftxs\execroot\_main\bazel-out\x64_windows-dbg\bin\kandinsky\glew32.dll,6D610000,6D68B000,C:\Users\Nigel Stewart\Downloads\glew-2.2.0\glew-2.2.0\bin\Release\x64\glew32.pdb,PdbFail
glu32.dll,C:\Windows\System32\glu32.dll,7FFBCA3A0000,7FFBCA3CD000,glu32.pdb,PdbFail
gpapi.dll,C:\Windows\System32\gpapi.dll,7FFC59950000,7FFC59977000,gpapi.pdb,PdbFail
imagehlp.dll,C:\Windows\System32\imagehlp.dll,7FFC5CCA0000,7FFC5CCC0000,imagehlp.pdb,PdbFail
imm32.dll,C:\Windows\System32\imm32.dll,7FFC5B980000,7FFC5B9B0000,imm32.pdb,PdbFail
kernel.appcore.dll,C:\Windows\System32\kernel.appcore.dll,7FFC59370000,7FFC5938A000,Kernel.Appcore.pdb,PdbFail
kernel32.dll,C:\Windows\System32\kernel32.dll,7FFC5C750000,7FFC5C819000,kernel32.pdb,PdbFail
KernelBase.dll,C:\Windows\System32\KernelBase.dll,7FFC5AB00000,7FFC5AECC000,kernelbase.pdb,PdbFail
main.exe,C:\Users\crist\_bazel_crist\3m6uftxs\execroot\_main\bazel-out\x64_windows-dbg\bin\kandinsky\main.exe,7FF775820000,7FF776AD4000,C:\users\crist\_bazel_crist\3m6uftxs\execroot\_main\bazel-out\x64_windows-dbg\bin\kandinsky\main.pdb,PdbLoaded
msasn1.dll,C:\Windows\System32\msasn1.dll,7FFC59CA0000,7FFC59CB3000,msasn1.pdb,PdbFail
mscms.dll,C:\Windows\System32\mscms.dll,7FFC53410000,7FFC534D8000,mscms.pdb,PdbFail
msctf.dll,C:\Windows\System32\msctf.dll,7FFC5CF30000,7FFC5D08F000,msctf.pdb,PdbFail
msvcp140.dll,C:\Windows\System32\msvcp140.dll,7FFC32F50000,7FFC32FDD000,D:\a\_work\1\s\binaries\amd64ret\bin\amd64\\msvcp140.amd64.pdb,PdbFail
msvcp140d.dll,C:\Windows\System32\msvcp140d.dll,7FFC4E100000,7FFC4E1E1000,D:\a\_work\1\s\binaries\amd64ret\bin\amd64\\msvcp140d.amd64.pdb,PdbFail
msvcp_win.dll,C:\Windows\System32\msvcp_win.dll,7FFC5A990000,7FFC5AA33000,msvcp_win.pdb,PdbFail
msvcrt.dll,C:\Windows\System32\msvcrt.dll,7FFC5D210000,7FFC5D2B9000,msvcrt.pdb,PdbFail
ntdll.dll,C:\Windows\System32\ntdll.dll,7FFC5D300000,7FFC5D566000,ntdll.pdb,PdbFail
ntmarta.dll,C:\Windows\System32\ntmarta.dll,7FFC59490000,7FFC594C5000,ntmarta.pdb,PdbFail
nvgpucomp64.dll,C:\Windows\System32\DriverStore\FileRepository\nvlt.inf_amd64_aadc33586c05acfa\nvgpucomp64.dll,7FFC4FF70000,7FFC5215E000,nvgpucomp64.pdb,PdbFail
nvoglv64.dll,C:\Windows\System32\DriverStore\FileRepository\nvlt.inf_amd64_aadc33586c05acfa\nvoglv64.dll,7FFB9F310000,7FFBA1F00000,nvoglv64.pdb,PdbFail
nvspcap64.dll,C:\Windows\System32\nvspcap64.dll,7FFC1B690000,7FFC1B961000,C:\dvs\p4\build\sw\rel\gfclient\rel_03_27\shadowplay2\proxy\win7_amd64_release\nvspcap64.pdb,PdbFail
ole32.dll,C:\Windows\System32\ole32.dll,7FFC5C830000,7FFC5C9CF000,ole32.pdb,PdbFail
oleacc.dll,C:\Windows\System32\oleacc.dll,7FFC28D20000,7FFC28D9D000,oleacc.pdb,PdbFail
oleaut32.dll,C:\Windows\System32\oleaut32.dll,7FFC5B9C0000,7FFC5BAA0000,oleaut32.pdb,PdbFail
opengl32.dll,C:\Windows\System32\opengl32.dll,7FFBCA760000,7FFBCA86C000,opengl32.pdb,PdbFail
powrprof.dll,C:\Windows\System32\powrprof.dll,7FFC590C0000,7FFC5911E000,powrprof.pdb,PdbFail
profapi.dll,C:\Windows\System32\profapi.dll,7FFC5A3A0000,7FFC5A3CF000,profapi.pdb,PdbFail
rpcrt4.dll,C:\Windows\System32\rpcrt4.dll,7FFC5C5A0000,7FFC5C6B6000,rpcrt4.pdb,PdbFail
rsaenh.dll,C:\Windows\System32\rsaenh.dll,7FFC592D0000,7FFC5930A000,rsaenh.pdb,PdbFail
SDL3.dll,C:\Users\crist\_bazel_crist\3m6uftxs\execroot\_main\bazel-out\x64_windows-dbg\bin\kandinsky\SDL3.dll,7FFBFB620000,7FFBFB877000,C:\temp\SDL3-3.2.2\VisualC\SDL\x64\Release\SDL3.pdb,PdbFail
sechost.dll,C:\Windows\System32\sechost.dll,7FFC5CBF0000,7FFC5CC96000,sechost.pdb,PdbFail
setupapi.dll,C:\Windows\System32\setupapi.dll,7FFC5BAB0000,7FFC5BF36000,setupapi.pdb,PdbFail
SHCore.dll,C:\Windows\System32\SHCore.dll,7FFC5B880000,7FFC5B96F000,shcore.pdb,PdbFail
shell32.dll,C:\Windows\System32\shell32.dll,7FFC5B150000,7FFC5B87D000,shell32.pdb,PdbFail
shlwapi.dll,C:\Windows\System32\shlwapi.dll,7FFC5C6C0000,7FFC5C729000,shlwapi.pdb,PdbFail
test_251013_190732.0574755.dll,C:\cdc\projects\kandinsky\temp\game_dlls\test_251013_190732.0574755.dll,7FFBF10C0000,7FFBF1359000,C:\users\crist\_bazel_crist\3m6uftxs\execroot\_main\bazel-out\x64_windows-dbg\bin\apps\std\std_shared.pdb,PdbLoaded
TextInputFramework.dll,C:\Windows\System32\TextInputFramework.dll,7FFC31AC0000,7FFC31C0D000,TextInputFramework.pdb,PdbFail
threadpoolwinrt.dll,C:\Windows\System32\threadpoolwinrt.dll,7FFC20950000,7FFC20966000,threadpoolwinrt.pdb,PdbFail
twinapi.appcore.dll,C:\Windows\System32\twinapi.appcore.dll,7FFC4D350000,7FFC4D58C000,twinapi.appcore.pdb,PdbFail
ucrtbase.dll,C:\Windows\System32\ucrtbase.dll,7FFC5AED0000,7FFC5B01B000,ucrtbase.pdb,PdbFail
ucrtbased.dll,C:\Windows\System32\ucrtbased.dll,7FFC29C80000,7FFC29E9F000,ucrtbased.pdb,PdbFail
UIAutomationCore.dll,C:\Windows\System32\UIAutomationCore.dll,7FFC1E9D0000,7FFC1EE04000,UIAutomationCore.pdb,PdbFail
umpdc.dll,C:\Windows\System32\umpdc.dll,7FFC590A0000,7FFC590B4000,UMPDC.pdb,PdbFail
user32.dll,C:\Windows\System32\user32.dll,7FFC5C2D0000,7FFC5C49A000,user32.pdb,PdbFail
uxtheme.dll,C:\Windows\System32\uxtheme.dll,7FFC571F0000,7FFC5729F000,uxtheme.pdb,PdbFail
vcruntime140.dll,C:\Windows\System32\vcruntime140.dll,7FFC357F0000,7FFC3580E000,D:\a\_work\1\s\binaries\amd64ret\bin\amd64\\vcruntime140.amd64.pdb,PdbFail
vcruntime140_1.dll,C:\Windows\System32\vcruntime140_1.dll,7FFC35A60000,7FFC35A6C000,D:\a\_work\1\s\binaries\amd64ret\bin\amd64\\vcruntime140_1.amd64.pdb,PdbFail
vcruntime140_1d.dll,C:\Windows\System32\vcruntime140_1d.dll,7FFC54A30000,7FFC54A3F000,D:\a\_work\1\s\binaries\amd64ret\bin\amd64\\vcruntime140_1d.amd64.pdb,PdbFail
vcruntime140d.dll,C:\Windows\System32\vcruntime140d.dll,7FFC4E020000,7FFC4E04E000,D:\a\_work\1\s\binaries\amd64ret\bin\amd64\\vcruntime140d.amd64.pdb,PdbFail
version.dll,C:\Windows\System32\version.dll,7FFC52BB0000,7FFC52BBB000,version.pdb,PdbFail
win32u.dll,C:\Windows\System32\win32u.dll,7FFC5A960000,7FFC5A987000,win32u.pdb,PdbFail
Windows.StateRepositoryCore.dll,C:\Windows\System32\Windows.StateRepositoryCore.dll,7FFC3E8E0000,7FFC3E8FA000,Windows.S tateRepositoryCore.pdb,PdbFail
windows.storage.dll,C:\Windows\System32\windows.storage.dll,7FFC580E0000,7FFC58936000,Windows.Storage.pdb,PdbFail
winmm.dll,C:\Windows\System32\winmm.dll,7FFC46830000,7FFC46866000,winmm.pdb,PdbFail
winsta.dll,C:\Windows\System32\winsta.dll,7FFC58BF0000,7FFC58C57000,winsta.pdb,PdbFail
wintrust.dll,C:\Windows\System32\wintrust.dll,7FFC5A490000,7FFC5A510000,wintrust.pdb,PdbFail
WinTypes.dll,C:\Windows\System32\WinTypes.dll,7FFC5A7E0000,7FFC5A954000,WinTypes.pdb,PdbFail
wldp.dll,C:\Windows\System32\wldp.dll,7FFC59C40000,7FFC59C9D000,wldp.pdb,PdbFail
wtsapi32.dll,C:\Windows\System32\wtsapi32.dll,7FFC56F00000,7FFC56F16000,wtsapi32.pdb,PdbFail
```

Most of them are Windows libraries (note how the debug symbols are not there :sad:). Actually, from
all the nonsense, only the following are "my code":

```text
assimp.dll,C:\Users\crist\_bazel_crist\3m6uftxs\execroot\_main\bazel-out\x64_windows-dbg\bin\kandinsky\assimp.dll,7FFB9BBF0000,7FFB9C40A000,,NoPdb
glew32.dll,C:\Users\crist\_bazel_crist\3m6uftxs\execroot\_main\bazel-out\x64_windows-dbg\bin\kandinsky\glew32.dll,6D610000,6D68B000,C:\Users\Nigel Stewart\Downloads\glew-2.2.0\glew-2.2.0\bin\Release\x64\glew32.pdb,PdbFail
main.exe,C:\Users\crist\_bazel_crist\3m6uftxs\execroot\_main\bazel-out\x64_windows-dbg\bin\kandinsky\main.exe,7FF775820000,7FF776AD4000,C:\users\crist\_bazel_crist\3m6uftxs\execroot\_main\bazel-out\x64_windows-dbg\bin\kandinsky\main.pdb,PdbLoaded
SDL3.dll,C:\Users\crist\_bazel_crist\3m6uftxs\execroot\_main\bazel-out\x64_windows-dbg\bin\kandinsky\SDL3.dll,7FFBFB620000,7FFBFB877000,C:\temp\SDL3-3.2.2\VisualC\SDL\x64\Release\SDL3.pdb,PdbFail
test_251013_190732.0574755.dll,C:\cdc\projects\kandinsky\temp\game_dlls\test_251013_190732.0574755.dll,7FFBF10C0000,7FFBF1359000,C:\users\crist\_bazel_crist\3m6uftxs\execroot\_main\bazel-out\x64_windows-dbg\bin\apps\std\std_shared.pdb,PdbLoaded
```

Two of them are written by me (main.exe and test_251013_190732.dll), while the other are
dependencies.

{{<admonition note>}}
Note how the PDB (debug symbols) path is *hardcoded* into the entries, to the point that glew32.dll
has `C:\Users\Nigel Steward\...` in the path, because that is where the pdb was at the moment of
compiling the shared library (thus creating the metadata).

Unless you control the whole environment, that path is not useful at all and has always been an
awful way of managing and finding debug symbols for debuggers.
{{</admonition>}}

## Program Initialization

OSes, when starting the program, will go through the whole list of dependencies above and load/patch
the dynamic library. This is sometimes called the `Runtime Linker`. When the OS loads a dynamic
library, it roughly goes through the following steps:

1. Find and reads the file into memory.

    The OS has specific paths and conventions to where to look for these files and in what order.
    There are mechanism to instruct the OS where to find (like LD_LIBRARY_PATH on Linux).

    Not finding this file will fail the start of the program. Windows will issue an image like this:

    {{<figure src="images/posts/dll_not_found.png" >}}

2. Parse the metadata, headers and all the information needed.

    Here it will also do the matching between the symbols imported (needed) by the program and the
    ones. If it cannot not, this is where you get the corrupted error. Normally this happens when
    there is a mismatch of versions (aka. you have an old .dll). But on Linux is very common to
    have very old system libraries (aka. trying to run your program in too old a Linux version).

3. Maps code section into the address space of the process.

   These pages will be most likely be marked as read-only, executable memory pages.

4. Perform the relocations:

    This is the "patching" mechanism, since this is the moment the OS knows where in the address space
    the memory sections are loaded.

5. Run any initialization code

    This is where things like global variables (and constructors in the case of C++) will be
    initialized.

{{<admonition important>}}
Note that initialization is *per shared object instance*. This will be important later on.
{{</admonition>}}

## Doing it manually

The above process is done by the OS on program startup, but there is nothing stopping us from doing
the same thing ourselves. Normally OSes will expose functionality to do the same thing. In Windows
this is [LoadLibrary](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-loadlibrarya).
That will search for the path, perform (roughly) the same steps as above and return a handle to
manage the lifetime of this library.

{{<admonition note>}}
If I can, I use a library like SDL, that provides an OS-agnostic call to load the libraries, like
[SDL_LoadObject](https://wiki.libsdl.org/SDL3/SDL_LoadObject).
{{</admonition>}}

Once we have the library loaded, we can perform queries into it, again using functionality exposed
by the OS.
On Windows, the function is [GetProcAddress](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getprocaddress)
(or [SDL_LoadFunction](https://wiki.libsdl.org/SDL3/SDL_LoadFunction)). This returns the pointer to
the function associated with that name. If you know the function signature, you can cast that
function pointer and call in into the library.

{{<admonition note>}}
See how we get the function name by string? This is where the `export "C"` nonsense comes from.

Because C++ could not get its shit together, each compiler "mangles" (aka. makes up) symbol names in
a different way, meaning that there is no good standard way of finding these functions. So people
just say to the C++ compiler: for the following functions, don't do your normal bullshit and export
the C names so we can query them in sane manner.
{{</admonition>}}

## Hot code reloading!

Now we have all the pieces to do hot code reloading. The main trick is to "split" your program into
two pieces: a bootstrap program and the "app dll". The bootstrap program is the actual `main`
executable you run, which knows where the app dll is (this can be as simple as receiving the path as
a cli argument). The job of this exe is then to initialize stuff, load the dll and call into it.
Most of the actual app logic is in the dll.

Then to do the "hot reload", we simply make the
bootstrap program listen to the dll file, constantly checking it for changes. Once it detects a
change, it can unload the current dll instance, reload the new one and call into it again. And
that's it: Hot code reloading! The process can be repeated any number of times.

Basically the steps are:
1. Copy the original DLL into a created temporary location.
2. Load that DLL, leaving the original one untouched.
3. Listens for changes on the original DLL.
4. On change, make a new copy into *another* temporary location.
5. Unload the first DLL.
6. Load the second DLL.
7. Start calling into the newly loaded.

TODO INSERT IMAGE

## Devil in the details

The overview above is true, but there are a lot of details that are critical to consider, which I
will talk in my next post, since why they do not change the flow, they are quite important.
Basically:

- Code architecture to aid with the process.
- Memory lifetime management.
- Global variable lifetime management.
- Some tips on handling with compilers/file timing.

### DLL management

The approach above of listening to the DLL, loading and unloading is not an OS utility, it must be
done by you. And there are a ton of details to make it work reliably.

1. DLL being locked.

    This is a Windows specific thing, but if a DLL is loaded on a program, the .dll file cannot be
    changed. Meaning that any recompilation will fail to write into that file.

    How is the compiler to write the new version? An approach is to make to make your build
    system/process generate a new filepath each time, but I find that a wrong approach, since your
    build system should not care/be aware that you're doing dynamic library shenanigans.

    I prefer to have my app, when loading the .dll, **copy that dll into a generated location and
    load that one instead**. This leaves the original .dll untouched and free for a next build. It
    also frees our program to do whatever it wants with them, since it is the creator/owner if these
    files.

    A gist of what I do in my own code is something like this:

```cpp
bool InitialGameLibraryLoad(PlatformState* ps, ...) {
    ...
    std::string new_path = std::format("{}\\test_{:%y%m%d_%H%M%S}.dll",
                                       temp_path.Str(),
                                       std::chrono::system_clock::now());

    // Copy the library to the new location.
    if (!SDL_CopyFile(ps->GameLibrary.Path.Str(), ps->GameLibrary.TargetLoadPath.Str())) {
        return false;
    }

    if (!LoadGameLibrary(ps)) {
        SDL_Log("ERROR: Re-loading game library");
        return false;
    }

    return true;
}
```

{{<admonition note>}}
Note that this is where the `test_251013_190732.dll` comes from! I likely should name it better,
but the rest is just the timestamp of when the copy is done. This ensures that I will not have
collisions when generating a new one.
{{</admonition>}}

    This leaves the DLL open for rebuilding. Now the app simply has to contantly check for changes
    on that file and repeat the process. The only thing that changes is that before we load the
    library, we need to unload the first.




