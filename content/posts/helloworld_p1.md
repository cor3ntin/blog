---
title: "A C++ Hello World And A Glass Of Wine, Oh My !"
date: 2018-01-22
---

```cpp
#include <iostream>

int main() {
    std::cout << "Hello, World\n";
}
```

Nothing to remove, nothing to add.

This is the proper “***Hello World***” in C++. All the others *Hello World* are **[wrong](https://www.youtube.com/watch?v=6WeEMlmrfOI)**.
But this is not where I rant about how using namespace std; crystallizes everything messed up with the teaching of C++. Another time perhaps.

Today we are gonna be compiling that *hello world* so that it can be executed on a **target system.**

But first, let me tell you a few things about me. I use Linux for fun and profit. I happen to think it is the best system. For me. As a developer. Sometimes, I look down on developers using Windows, wondering how the eff they manage to get anything done by clicking on things. And it is likely that a vim user on Arch looks down on me for using Ubuntu. Nobody’s perfect.

Anyway, let’s fire up a terminal

    # sudo apt-get install g++
    # g++ -o helloworld helloworld.cpp
    # ./helloworld
    Hello, World!
    #

Nice, That’s simple, let’s go home and have a beer 🍻 !

But then, enters my boss. They are in the business of selling software to people who use Windows. I try to show them that I can make a cow speak and that I can resize my terminal so we should obviously move all our business to Linux at once, they say something incomprehensible about market shares and apparently, they can resize their command prompt too.

{{< figure src="mrrobot.png">}}

After looking down at each other for a while like we are stuck in an Escher painting, I grudgingly remember that I’m in the business of making my clients happy, and so we are going to port our *hello world* application to Windows. Our boss doesn’t care what environment we use to create that ground breaking app, and they don’t have an issue with me continuing working on the Linux version at the same time, so I’ve decided to develop that application for Windows, **on** Linux; what could possibly go wrong ?

Besides, it will then be far easier to set up a build farm and continuous integration. You could even have your CI provision fresh docker containers on the fly to build the windows app in a controlled and fresh environment. While I tend to think that Dockers is a bit of a cargo cult, Using Docker along Jenkins is actually something that makes a lot of sense. And if you like your sysadmin, don’t force them to deal with Windows servers.

We should strive to make your application as portable and platform agnostic as possible, so having a windows version of our application may actually make our code better. That’s what I try to tell myself.

As it turns out, Microsoft is nice enough to offer a compiler for windows called msvc, and I have the feeling that msvcis a better choice on windows than g++since that’s the compiler the whole ecosystem is designed around. And hopefully Microsoft knows their own tools, formats and calling convention best. I never went the extra mile to benchmark that though, and you will find proponents of either approach on internet. But, [The MSVC team agrees with me.](https://blogs.msdn.microsoft.com/vcblog/2017/03/07/msvc-the-best-choice-for-windows/) Shocker.

Anyway, for now, let’s stick to that.

    # apt-get install msvc
    E: Unable to locate package msvc

Surprisingly, that doesn’t work. You can’t blame a guy for trying. But to explain why, let me tell you how a compiler works.

A compiler opens a file, transforms the content of that file into something that can be executed, and writes that out to some other file. Sometime you have more than one source file, so you need a linker which is a program which opens a bunch of files and writes an executable down. An executable is a file, nothing magical about it. Sometimes you need libraries. A library is a file. And you mostly likely need tons of headers which are… you get it, files. plain old boring files. The executable is then loaded by another executable which is also a file, it’s files all the way down. Ok, maybe not, Plan 9 has more files.

To be clear, compilers are extremely complex pieces of engineering, especially C++ compilers and you should offer a cookie to all the compiler writers you meet. However, from a system-integration point-of-view, they are as trivial as it gets. Most compilers don’t even bother with threads. They let the build system deal with that. Which is unfortunate since most build systems have yet to learn how to tie their shoe laces.

Anyway…here is the list of kernel facilities you need to write a compiler:

* Opening, Reading and Writing files

* Reading directories content

* Allocating memory

You may therefore be thinking that this is a reasonable enough list, and so, you wonder why msvc isn’t available on Linux. Sure, having msvc build Linux/ELF applications would be a huge and probably pointless undertaking, but all we want is to build an application *for* Windows, and surely Microsoft would make it easy as possible for me to do so, right?

But there is this thing. Windows is an ***“ ecosystem “***. It means they want to sell their OS to both their users and their developers, sell their tools to developers, and make sure nobody learn about that other OS legends speak of. So if you want to build a windows application, you need Windows. Crap.

Fortunately, someone rewrote Windows on Linux and called that wine. Probably because they had to be very drunk to even think about doing it. It took merely 15 years for wine to reach 1.0. But it’s in 3.0now, so maybe we can use it ? There are minor miracles in the open source community and WINE is certainly one of them.

For a very long time, MSVC was bundled with Visual Studio. Which mean that if you wanted to compile a C++ app on windows using Qt creator, CLion, Eclipse or notepad++, you still had to have Visual Studio. Ecosystem and all that.

Things are better now, you can install the “build tools” such that you will only need to install about 5GB of… stuffs. Let’s do that.

Oh, apparently the compiler is distributed as an executable which then downloads stuffs you didn’t asked for over the internet. Maybe it’s better than a 40GB zip ?

    # wine vs_BuildTools.exe
    The entry point method could not be loaded

Are you surprised? My dreams are crushed. We **do** need a windows to do some windows development ( spoiler : it gets better ).

Let’s fire up a VM. If you want to follow along, I recommend that you use a new or cloned Windows 10 VM. We are gonna install a lot of weird things and it will be next to impossible to clean up after ourselves. Afterwards you may [get rid of the VM](https://www.youtube.com/watch?v=z0wK6s-6cbo).

{{< figure src="vs1.png" title="Isn’t he cute ? Maybe I should send my resume to Oracle" >}}

Once that’s done, [we can go and download the VS build tools](https://www.visualstudio.com/downloads/#build-tools-for-visual-studio-2017).

Scroll down until you get carpal tunnel. The build tools is the second to last item. Download that.

I had a few issues launching the installer. I think they try to contact a server that managed to get itself in a list of ad servers so my DNS blocked it. Don’t ask.

{{< figure src="vs2.png" title="That took way more than a minute" >}}

Installers with loading screens, it’s perfectly normal. Once it’s done, it loads the main UI, slowly and painfully but then we get to check boxes. I’m having a blast.

{{< anim src="components.webm" title="Shopping list.">}}

You won’t need the static analysis tools, but they get checked when you install a compiler no matter what. That’s fine.

We need a C++ toolset — the thing everybody else calls a toolchain. I’m not sure what is newer, v141 or 15.4 v14.11. Use a die ?

We also need a C runtime too, that’s handy. I’m not sure whether we need the CRT or the URT so, we will just install both. The URT/CRT is nice though. Before it came into existence, everything was much, much harder.

Finally, we will probably need to use some windows features so we should get the Windows SDK. Apparently that depends on some C# components. And some JS libraries, *obviously. *To be clear, you can’t do anything remotely useful without the Windows SDK, better get it now.

{{< figure src="vs4.png" title="WIN32_LEAN_AND_MEAN" >}}

Time to have a coffee pot while Visual Studio gets into every recess of your hard drive. At some point though, it’s done so you can go do some bicycle with the butterflies. Nice.

{{< figure src="vs5.png" title="Improving my editorial line with pretty pictures, courtesy of Microsoft" >}}

A butterfly isn’t enough to make me want to ditch Linux, so let’s see if what we have just installed can be used without a Windows box.

Copy the following outside of your VM:

* C:\Program Files (x86)\Windows Kits\10\Include

* C:\Program Files (x86)\Windows Kits\10\Lib

* C:\Program Files (x86)\Microsoft Visual Studio\2017\BuildTools\VC\Tools\MSVC

The first two paths are the Windows SDK, the last one is the toolchain, containing the compiler, the linker, the STL and the VC runtime libraries, for all architectures.

In my installation I have URT files all over the places, So I guess that if you install the Windows 10 SDK you actually get the CRT so you don’t need to activate it separately when you select the components to install. Compared to only a few years ago the situation is much better. Microsoft has been busy.

{{< figure src="vs6.png" title="Copy all the things !" >}}

I put everything in a folder called windows, I have the msvc compiler with the runtime, the STL and the redistribuables on one side, and the windows 10 SDK in a separate folder. I didn’t keep any information about which version of the SDK or the toolchain I was using, you may want to do that more properly.

In the Windows Sdk there are some useful binaries & dll likerc.exe Put them alongside in msvc2017/bin/Hostx64/x64 where the toolchain binaries are located, including cl.exe.

If you are not familiar with windows development:

* cl.exe is the compiler

* link.exe is the linker

* rc.exe is a tool to deal with resources files, including icons and manifests

You may need various other tools if you have to deal with drivers, cab files, MSI installers, etc.

The whole thing is about **2.9GB**. About half what we had to install on the windows VM.

{{< figure src="term1.png" title="The road so far" >}}

Let’s have some wine again 🍷. Visit [https://wiki.winehq.org/Ubuntu](https://wiki.winehq.org/Ubuntu) and [https://github.com/Winetricks/winetricks](https://github.com/Winetricks/winetricks) to make sure your wine setup is up to date. I will be using wine-3.0.

Then, install the redistribuable for VC 2017. The process is mostly automatic. We will use a dedicated wine prefix to keep everything kosher. What that means is that it’s actually easier to have multiple msvc installations under wine that it is on windows.

```sh
WINEPREFIX=windows/vs-buildtools2017-wine64 WINEARCH=win64 winetricks vcrun2017
```

Then in your windows folder, create a bin64 folder in which you can write a small bash script with the following content.

```sh

#!/bin/bash
set -ex
DIR="/home/cor3ntin/dev/cross-compilers/windows/bin64"
export WINEPREFIX="$DIR"/../vs-buildtools2017-wine64
export WINEARCH=win64
export WINEDEBUG=-all
PROGRAM=$(readlink -f "$0")
PROGRAM=$(basename "$PROGRAM")

ARGS=( "$@" )
x=0;
while [ ${x} -lt ${#ARGS[*]} ]; do
    if [[ "${ARGS[$x]}" ==  '/'* ]] && [ -e "${ARGS[$x]}" ]; then
        ARGS[$x]=$(sed 's/\//\\/g' <<< "${ARGS[$x]}" )
    fi
    x=$((x + 1))
done

wine "$DIR"/../msvc2017/bin/Hostx64/x64/$PROGRAM ${ARGS[@]}
```

That script will first set up wine to use our prefix . Then we do Linux -> windows path separator translation ( / to \ ) before forwarding the arguments to the actual windows PE binary running on wine.

We will use a tool called [shc](https://github.com/neurobin/shc) to convert that wrapper into a proper elf executable. Otherwise we may have issues down the road. Another solution would be to write a C++ wrapper instead of bash. the shc has a few drawbacks, starting with the need for a hard coded install path.

    shc -f cl-wine.sh -o cl.exe
    shc -f cl-wine.sh -o link.exe

You can create a bin32 folder in the same manner, just changing the last line to wine "$DIR"/../msvc2017/bin/Hostx64/**x86**/$PROGRAM

To have a x86 target compiler. I’m not sure why you need two sets of separate binaries to support another architecture, but we do. Finally, the x86 linker may complain about missing libraries so we are gonna create some symlinks.

    cd windows/msvc2017/bin/Hostx64/x86/
    for x in $(ls ../x64/ms*.dll); do ln -s $x .; done

{{< figure src="term2.png" title="We are going places !" >}}

One last thing before we can do some serious work. We need to delete vctip.exe as it doesn’t work. It’s a telemetry tool so we don’t need it. It’s located in windows/msvc2017/bin/Hostx*/**. If you don’t follow that step you will encounter weird stack traces.

Time to build our Hello World application ! It’s actually straightforward

```sh
windows/bin64/cl.exe                   \
  /nologo /EHsc                        \
  test/helloworld.cpp                  \
  /I windows/msvc2017/include/         \
  /I windows/sdk_10/include/ucrt/      \
  /link                                \
  /LIBPATH:windows/msvc2017/lib/x64/   \
  /LIBPATH:windows/sdk_10/lib/um/x64   \
  /LIBPATH:windows/sdk_10/lib/ucrt/x64 \
  /out:helloworld.exe
```

We are building an executable that depends on the compiler headers ( including the STL ), the C runtime, and some windows libs such as kernel32.lib.

{{< figure src="term3.png" title="🎉 😎 🍻!" >}}

For completeness, here is the x86 build

```sh
windows/bin32/cl.exe                   \
  /nologo /EHsc                        \
  test/helloworld.cpp                  \
  /I windows/msvc2017/include/         \
  /I windows/sdk_10/include/ucrt/      \
  /link                                \
  /LIBPATH:windows/msvc2017/lib/x86/   \
  /LIBPATH:windows/sdk_10/lib/um/x86   \
  /LIBPATH:windows/sdk_10/lib/ucrt/x86 \
  /out:helloworld.exe

```

Truth is, the whole endeavor is reasonably simple, perhaps more so than using windows proper. No messing about with vcvarsall.batand all your favorites tools such as perl, git, python, sed, the terminal, zsh…are there and work properly.

## 🔨 Build System

We got cl.exeworking on linux, yeah ! Before we go further, we should add that alien toolchain to a nice, modern build system. As I’m not in the mood to deal with the hotmess that is cmake, we will be using **QBS**, my favorite build system.

Setting up qbs to use our wine/msvc compiler should be easy…

QBS can detect toolchains automatically, however, there are a few issues. First the tools assumes MSVC only exists on windows so some code paths are disabled away. I think this could be fixed in a few hours, it would merely require implementing the CommandLineToArgv function in a portable manner.

However, there is something to be said about tools being too clever. QBS attempts to parse vcvars.bat at an assumed location. That’s one file we happily got rid of.

Reality check, we are not going to get any sort of automatic detection. We don’t really need to. We can set up the toolchain manually and treat it as a separate thing from the msvc-on-windows-proper. Detection isn’t really an issue since all we have are a couple of include directories and library paths.

I’ve started to push some files to GitHub, it’s very much a work in progress. Debug builds are completely broken at the moment. It’s a module which offers some understanding of our wine toolchain. It mostly disables all probing and assumes everything is already configured properly.
[cor3ntin/qbs-wine-toolchain](https://github.com/cor3ntin/qbs-wine-toolchain)

So we have to do all the work of setting the profile manually. But if our endeavor proved anything, it’s that even a toolchain as hairy as VC++ can be reduced to a handful of variables ( compiler, linker, tools path, includes, defines, library paths). So, h[ere is my QBS profile configuration](https://gist.githubusercontent.com/cor3ntin/85eda9db218bd194f235b8098226d185/raw/4f0f951c5c8b78c77152acfb54dfe0e36c15d9de/qbs_config_x64.conf).

And finally, we can write a small qbs build script

```js
import qbs 1.0

CppApplication {
    name  : "helloworld"
    files : ["helloworld.cpp"]

    Group {
        name: "app"
        fileTagsFilter: "application"
        qbs.install: true
        qbs.installDir: "bin"
    }
}
```

Which we can then run and *Voilà !*

{{< figure src="term4.png" title="Bonsoir Eliot." >}}

runner.sh is a [small script](https://gist.github.com/cor3ntin/ecd136ad792f54dd7feb89cb094dcf75) that sets up the wine prefix before launching the freshly built windows executable. Nothing too fancy.

So here you have it. A Microsoft compiler, wrapped in a bash script compiled to ELF, building 64 bits PE executables, driven by a modern build system run on Linux. That’s pretty satisfying.

Our hypothetical Boss is knocking on the door. See you in [part 2]({{< ref "helloworld_p2.md" >}}).
