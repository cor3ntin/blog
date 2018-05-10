---
title: "A C++ Hello World And the Cute Heartless Rainbow"
date: 2018-01-24
---

This is Part two of a series wherein we build a “Hello World” application. If you are late to the party, I encourage you to check [part 1]({{< ref "helloworld_p1.md" >}}) first.

{{< figure src="dragon.png" title="Hic Sunt Arcūs" >}}

So, our Boss came in to check on our progress. They were starting to wonder why it takes a whole day to port a 3 lines application to a new system. But the real reason of their visit was to ask for a new feature. While our “Hello world” business is booming, the marketing department thinks the lack of graphical UI is hurting sales.

See, nobody escapes software bloat.

Still. Eager to make the boss forget the time it takes to setup MSVC on Linux, I went above and beyond and completed a full rewrite of our app.

```cpp
#include <QMessageBox>
#include <QApplication>

int main(int argc, char** argv) {
    QApplication a(argc, argv);
    QString message = "Hello, World !!!!!!!";

    QMessageBox::information(nullptr, "Hello World v2.0", message,
                             QMessageBox::Yes | QMessageBox::No);
    return 0;
}
```

Of course, we also provide a small nifty build file.

```
import qbs 1.0

QtGuiApplication {
    name  : "helloworld-gui"
    files : ["helloworld.cpp"]

    Depends {
        name : "Qt.widgets"
    }

    Group {
        name: "app"
        fileTagsFilter: "application"
        qbs.install: true
        qbs.installDir: "bin"
    }
}
```

The Linux version is still building fine. Duh.

{{< figure src="term1.png" title="The author likes to pretend all Linux machines are born with Qt And QBS installed. Just play along." >}}

{{< figure src="stupiddialog.png" title="In case of nuclear disaster, click “no”" >}}

Let’s enthusiastically build the Windows version:

{{< figure src="term2.png" title="Not so Qt." >}}

I went to download Qt on [https://www.qt.io/download](https://www.qt.io/download). That site is getting worse by the day. The Qt company tries to dissuade people to get the open source version. But if you go to [https://download.qt.io/](https://download.qt.io/) you have all the tarballs and installers.

Speaking of installers, they do not offer a 64 bits version for Windows. in 2018. We may have to restrict ourselves to 32 bits builds and extract that with WINE. Except it doesn’t work because the Qt installer framework has no silent mode ( apparently, you can script a silent mode in 100 lines of JavaScript ) and uses some Windows methods unsupported by WINE.

Nevermind. I will build Qt myself to prove how awesome my WINE toolchain is. And it will be 64 bits. I’ll get promoted and the intern who has 4 PHD will bring me cappuccinos. Maybe people will even use my name as a verb ( If you don’t [Godbolt](https://godbolt.org/) yet, you should definitively check it out !).

Except. Qt still uses qmake to build itself. And qmake exists only to make cmake look cool and modern. There is an [ongoing effort ](http://code.qt.io/cgit/qt/qtbase.git/log/?h=wip/qbs2)to build Qt with qbs and while this is very exciting, it may be a bit too cutting edge, even for this blog.

So, we are stuck with qmake .qmakeis a build system generator which unfortunately conflates toolchains and build systems. I did attempt to create configuration for my wine toolchain, it was actually rather simple, and it did generates some Makefiles with the proper commands. But they were Makefiles for nmake which is a make-like tool for windows albeit with a format that is not quite compatible with the original make. I tried using nmake (which works fine) but then all the subsequents cl.exe / link.execalls happen in the wine environment which means it executes the actual cl.exe rather than our wrapper and so our crude slashes-to-backslashes transformation script never gets run and the compilation fails because cl.exe assumes anything starting with `/` is an option. And we can’t get nmake to call our fake cl.exe wrapper since nmake is a windows process and windows doesn’t know about ELF.

It is left as an exercise to the reader to calculate how many millions of dollars Windows using `\` as a path separator and is costing the industry.

The solution would be to patch qmake. Which even the maintainer of qmake actively avoid to do. So let’s not do that.

I’m very sorry, but we will go back to our Windows VM and build Qt from there.
> # That should be simple, right ?

Of course, the VC build tools are unable to set up their environment properly.

How many Microsoft Engineers does it take to set up a build environment ? Certainly quite a few, since I was able to find about 1900 lines of batch scripts serving that purpose, in 20 or so files. There are probably more.

I manage to get my environment sorted in 3 lines. Remember, regardless of how complex your build system is, it boils down to a handful of variables. A compiler doesn’t needs much.

```sh
set "INCLUDE=C:\Program Files (x86)\Microsoft Visual Studio\2017\BuildTools\VC\Tools\MSVC\14.12.25827\include; %INCLUDE%"
set "LIB=C:\Program Files (x86)\Microsoft Visual Studio\2017\BuildTools\VC\Tools\MSVC\14.12.25827\lib\x64\;%LIB%"
set "PATH=%PATH%;C:\Users\cor3ntin\Documents\qtbase-everywhere-src-5.10.0\gnuwin32"
```

After that, it was a matter of downloading Qt 5.10 setting up a x64 build environment ( I used vcvars64.bat + the three lines above, but you can set up PATH manually and not bother with vcvars.bat at all).

Make sure perl is installed and in the PATH.

For the purpose of this article, I only need Qt Base (Core, Gui, Network… ) so that’s what I’m building. Building QtWebEngine - which uses chromium - is a bit more involved.
```sh
configure -release -opensource -confirm-license \
          -platform win32-msvc -nomake examples -nomake tests
nmake
nmake install
```
Once Qt has done building, copy the folders back to your linux host. You need at least bin, include, lib, plugins . put them in a new directory. I called mine qt5_10base-x64 .

I had issues with the includes referencing src. So you can run perl bin\syncqt.pl -copy -windows -version 5.10.0 -outdir cpy from the Qt directory on windows and use the include folder generated in cpy . Even then I had to copy a few files manually ( qconfig.h, q{core,gui,widgets,network,xml}-config.h) from the src folder to their respective include folder. Definitively a bit fiddly but eventually you will get there.


{{< figure src="term3.png" title="My Linux is slowly morphing into a Windows Box. I expect Clippy to appear at any moment now" >}}

So now we have Qt. But how can we tell QBS to actually use it ?

The profile system of QBS is one of the things that make it great. You can set up compiler toolchain globally and switch from one to another effortlessly.

However to works with Qt, qbs needs a complete set of modules per profile. The gcc profile we used earlier is made of 160 qbs files setting each Qt library and component.

Fortunately, all you need to do is to call this handy tool.
```sh
qbs-setup-qt -h
This tool creates qbs profiles from Qt versions.

Usage:

    qbs-setup-qt [--settings-dir <settings directory>] --detect

    qbs-setup-qt [--settings-dir <settings directory>] <path to qmake> <profile name>

    qbs-setup-qt -h|--help

The first form tries to auto-detect all known Qt versions, looking them
up via the PATH environment variable.

The second form creates one profile for one Qt version.
```

Except we can’t call that tool from linux because the built qmake is a windows application expected to run on Windows. Hopefully, someday we will get rid of qmake entirely, but for now, back to our Windows machine.

We first [install a windows build](https://doc.qt.io/qbs/installing.html) of qbs and run the tool.

```sh
qbs-windows-x86_64-1.10.0\bin\qbs-setup-qt.exe \
    --settings-dir . bin\qmake.exe msvc14-x64-qt
```
That should create a qbs folder with the proper Qt configuration. Move that folder to your linux machine in the qt5_10base-x64 folder created earlier.

If you open on of the .qbs file, say 1.10.0/profiles/msvc14-x64-qt/modules/Qt/gui/gui.qbs , you will notice a reference to a path. For some reason, in mine it’s /usr/local/Qt-5.10.0 . I guess I messed up somewhere since we should have a windows path. In any case we need to transform that path to the actual locations of Qt on our Linux machine, which is easy to do, just use sed.

```sh
find -name "*.qbs" -exec sed -i -e \
    's\#/usr/local/Qt-5.10.0\#/home/cor3ntin/dev/cross-compilers/windows/qt-5.10-msvc-x64\#g' {} \;
```
We then need to modify our qbs.conf to use those qt modules. Edit the QBS file created in part one to reference them:
```sh
qt-project\qbs\profiles\msvc14-x86_64\preferences\qbsSearchPaths=\
    /home/cor3ntin/dev/Qt/qbs-wine-toolchain,/home/cor3ntin/dev/cross-compilers/windows/qt5_10base-x64/qbs/1.10.0/profiles/msvc14-x64-qt
```
I realize it’s all a bit tricky. Eventually though, we can run qbs and it compiles our Qt-based hello world for Windows.

{{< figure src="term4.png" title="$?==0" >}}

It won’t run because it can’t find the Qt dll. You can either copy the dll to the build directory, or make sure they are in the PATH. The windows/wine PATH that is.

The only way I found to do that is to run regedit — make sure to do it from the appropriate WINEPREFIX and add the location of the Qt’s bin directory to Path which is under `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Environment`.

If you know a way to script that away, please let me know.

{{< figure src="regedit.png" title="Using regedit on Linux. What would I not do for my readers ?" >}}


We also need to put a qt.conf file next to our built helloworld.exe binary to tell Qt where to load the plugins from. In my case
```ini
[Paths]
Prefix=/home/cor3ntin/dev/cross-compilers/windows/qt5_10base-x64/
Plugins = plugins
```
And that’s it ! It wasn’t too hard, was it ?

{{< figure src="hello1.png" title="I made a clicky thing !" >}}

### And So, Microsoft Studio Visual C++ Compiler is the greatest thing ever !

A recent study conducted on two preschoolers shows that millennials like rainbows. I was immediately asked to made our app more rainbow-y. Ambiguously labeled buttons are apparently the horsemen of the pretend apocalypse so we will also get rid of them.

What we need to do is iterate over our string an inject some html markup around each character so that they are render in different colors.

Iterating over some container will require us to use range-v3. There is *literally* no other way.

{{< figure src="rainbow1.png" title="Making WordArt Great Again." >}}


Isn’t that great ?

But, and I’m sure you you will be utterly flabbergasted to learn that msvc is not capable of compiling range-v3. I know, it’s a total blow, and a complete let down.

### Microsoft Studio Visual C++ Compiler Defeated.

There is absolutely nothing we can do. It’s not like there exists a [msvc-compatible port of range-v3](https://github.com/Microsoft/Range-V3-VS2015)or [some other way](http://en.cppreference.com/w/cpp/language/range-for) to transform our string. Please stop saying I purposefully made up a contrived story just so I could use ranges and defeat msvc with rainbows. That would be *mean*.

The only reasonable thing is to ditch msvc and replace it by something else. But what ? Mingwdoesn’t understand the MSVC headers and libs, icc is quite expensive, Cfrontis no more maintained.

By now, you certainly know where I’m coming at, don’t you ? clang-cl ! If you didn’t know about clang-cl, it’s a drop in replacement for cl.exe except it’s actually clang, so it will make you all warm and fuzzy inside.

Clang was designed properly and it’s an amazing tool for a lot of reasons, but most importantly:

* It’s runs on all major OSes

* It doesn’t activate OS-specific features at compile time, but at runtime.

* Which means it will let you target anything from anywhere.

* It offers a driver ( a command line interface, if you like ) compatible with that of msvc

* It’s easy to build ( and if you ever attempted to build GCC, you know how hard building a compiler can be.)

* It’s open-source and its designed isn’t actively constrained by political motives.

### So, let’s use Clang !

If you don’t have clang follow the documentation of your Linux distribution, it should come with clang-cl by default ( it’s merely a symbolic link to clang ).

Or if you are like me, checkout out and build the trunk, [it’s full of goodness](https://clang.llvm.org/cxx_status.html) !

Beside being a drop in replacement, there are a few things we need to do. The QBS toolchain module we wrote for wine does not quite work since it transforms slashes to backslashes.

I did copy the toolchain modules and fixed some other details. That will end up on GitHub soon.

Creating a QBS profile for clang-cl is straight forward, copy the one from wine and change the toolchain name from msvc to clang-cl and point the toolchainInstallPath to somewhere containing clang-cl and lld-link .

Oh, didn’t I mention lld-link ? It’s a drop in replacement for link.exe. lld is also a replacement for ld on unix, and it’s much, much faster than ld and gold so you should [check it out and use it](https://lld.llvm.org/)!

We are not quite done yet.

{{< figure src="monkey.jpeg" title="Case consistency at Microsoft is a serious matter" >}}

Microsoft Windows is designed around case insensitive file systems. Which I’m sure looked like a good idea at the time.

However, unless you are masochistic enough to run your Linux machine on FAT, chances are your file-system is case sensitive. And that is an issue for us.

How bad it is, really ? Well…
<pre>
**A**cl**UI**.**L**ib
ahadmin.lib
wdsClientAPI.**LIB
P**sapi.**L**ib
sensorsapi.lib
**S**etup**API**.**L**ib
</pre>
This is just a small selection of files. I don’t suppose there is anything they can do to fix that now, it’s too late.

We could attempt to fix the case issues by changing the filename of every library and header. But that will probably not work since third party libraries aren’t consistent either. So even if we attempt to fix our build files sooner or later we will run on a case issue.

So, the “proper” (for some definition of proper) solution would be to trick the Linux kernel into being case insensitive. And to that we need to use a case-insensitive system. Fortunately, a file can be a file system, so we will create a file large enough to hold the windows SDK, format it with a case-insensitive filesystem such as EXFAT and put the SDK there. Please note that NTFS is case sensitive, even if the Windows kernel is not.

```sh
dd if=/dev/zero of=win_sdk_10.fs bs=1024 count=$((1024 * 100 * 3))
mkfs.exfat win_sdk_10.fs
mv sdk_10 sdk_10.bak
mkdir sdk_10
sudo mount win_sdk_10.fs
mv sdk_10.bak/* sdk_10
```
You gotta love Linux.

And now, compiling with clang-cl works. However, running the program exposes a bug.

{{< figure src="term6.png" title="Our program compiles with clang and runs but it exhibits a nasty bug." >}}

And yet, the same executable copied to a virtual machine runs fine.

{{< figure src="rainbow21.png" title="Such greetings" >}}

I’m still not sure where the bug actually is. It appears to be either in range-v3 or my use thereof, and yet it seems odd that it would expose a different run-time behavior.

But that’s what’s great about having a bigger set of development environments, if you have bugs, the are more likely to get exposed. For one, this nasty graphical glitch made me realized I should handle white spaces as a special case.

Oh, and if you want to check what clang is actually doing, you can use Microsoft’s `dumpbin.exe` found in `msvc2017/bin/Hostx64/x64/` . This tool is equivalent of ldd and nm for unix.

{{< figure src="term7.png">}}

Interestingly, the clang artifact looks exactly like a msvc-produced one, albeit with few extra sections. Including the CRT and the vcruntime !

Here is the MSVC built binary.

{{< figure src="term8.png">}}

Right now ldd-link is [still experimental](https://lld.llvm.org/windows_support.html) and offers no support for debug info. clang-cl is [mostly done](https://clang.llvm.org/docs/MSVCCompatibility.html) except for some exceptions handling. You can mix and match `cl.exe` `clang-cl` `link.exe` and `lld-link`.

I can’t advise you to use clang-cl in production (yet, it’s getting there ) but it’s an amazing tool to add to your CI and development environment.

It’s all I have for you today, I hope you’ve learned something!

## [One more thing … See you in part 3]({{< ref "helloworld_p3.md" >}}).
