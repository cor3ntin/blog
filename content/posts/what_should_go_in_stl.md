---
title: "A cake for your cherry: what should go in the C++ standard library?"
date: 2018-02-21
---

This is a reply to [Guy Davidson’s article “Batteries not included: what
should go in the C++ standard library?](https://hatcat.com/?p=16)”.

Over the past few years there has been a push to include a graphics
library into the C++ standard. It would be something a bit like cairo.
Or SDL. The proposal, in its current form, is [here](https://wg21.link/P0267)

In its current state, the library proposal can draw some shapes on a
pre-allocated surface, has some support for images, and there are of
courses projects to add text, maybe some input in the form of mouse /
keyboard handling.

The primary goal of the library seems to be teaching. The argument put
forward is that it’s cool and ludic for kids to have pretty blinky
pixies on screen. Of course there already exists libraries to do that
and more, but you see, C++ has no decent, idiomatic package manager, so,
of course the conclusion was reached by some prominent committee members
that the C++ standard should offer a 2D graphics library, out of the
box.

I do think this is a path that should not be pursued and that doing so
would be, at best, a waste of time. Let me tell you why.

#### But, first, some needed clarifications

Guy Davidson and others have, on all account, put a great amount of
work, time and energy in that proposal. The people pushing for that
proposal to be rushed through standardization are way more experts than
I will ever be.

I did not contribute anything to C++ so what will follow are just the
opinions of one guy.

I also want to make clear that I do not have a negative opinion of that
particular library. My issue is with the inclusion of a 2D painting
library, any painting library in the C++ standard, at this point in
time.

I hope I won’t be misconstrued

Anyway, let’s get to it.

### The C++ Standard Library is not a library

The C++ Standard is exactly that: A well specified document that
describes in the most detailed and unambiguous possible way what C++ is,
and how it works. The goal being that anyone can implement a C++
compiler for themselves by implementing that specification. However, it
happens that the specification is not specific enough, or implemented
not quite properly, or implemented opinionatedly and so various C++
compilers end up having some slight differences of behavior from one
implementation to the next. Sometimes it cannot be implemented at all
because the people doing the implementation and the people doing the
specification forgot to talk to each others.

Now, a big part of that specification describes the Standard Template
Library, a library that ships with every conforming compiler.

There exist at least 5 implementations of that specification, maintained
by as many entities. Some are open source, some are not. They each work
in a select subset of platforms and system. And even if they sit at the
very bottom of about any C++ program, they are, like any other library ,
subject to bugs.

In this context, what should, or should not be included in the C++
standard library is a very important question. What should come as
standard bundled with the compiler? What do most people need to be
productive with C++ ?

Guy’s article describes the positions one can have. Maybe we need about
nothing ? Maybe we need some vocabulary types ? Maybe containers ? Maybe
not ? Do we need filesystem support ? sockets ? json ? xml ? rpg making
tools ? sql ? html ? javascript vm ? 2d graphics ? 3d graphics ? soap ?
IPC ? windowing ? Should $\pi$ be defined ?
What about websockets ? ftp ? ssh ? VR ? AR ? crypto ? ssl ?
Do we need ssl but no other crypto ? Deep learning ? Sound ? 3d sound ?
Video Decoding ? gif ?

Clearly we need to draw a line.

Somewhere ?

Where ?

Let’s look at .Net. Or Java. When the STL is mentioned, comparing C++
and Java is customary. Java is cool, right? It has sockets and HTTP and
crypto and everything, basically.

But Java is mostly maintained by a single entity. So someone at Oracle
decides Java should have sockets and they implement it, there are
internal reviews and now Java has Sockets. Sometimes later, Google want
to have sockets using the same API, and before they can say “Ahead of
time”, they are sued for 9 billions USD.

Meanwhile, the C++ specification undergoes a long, painful process until
there is a vote and there is a majority consensus on every single
feature, every single method. Should that be called `data`?
`get`? “At Bloomberg we have experience using `data` on our 2
millions line code base” will say the guy working at Bloomberg. “We
noticed it’s faster to use type `get` on
EBCDIC keyboards” Will object the guy at IBM. “And we have a 3 millions
lines code base”.

I don’t have an opinion on which model is best. Benevolent dictatorship
obviously only works if the dictator is benevolent.

However, I will argue that democracy is unfit to the birth of a good
graphic library.

### The Committee Resources are limited

Even if sleep deprived proposal authors sweat blood, a big part of the
work and the voting takes places in week-long quarterly meetings where
people go through an ever growing pile of proposals. As the committee
learn to be more transparent, more people contribute, leading to more
work for people attending. There is little to no money in that work. At
best you can hope for someone to pay you the plane tickets to the
beaches of Florida, the green hills of Switzerland or the pools of
Hawaii at which the meeting happen. You will reportedly never see
neither the beaches, the hills nor the pools.

And because resources are limited and time is limited there is a need to
sort, prioritize and even discard proposal. [Directions for ISO
C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0939r0.pdf)
attempts to describes how that sorting and prioritizing should happen.

The question then becomes : can the committee spares the time to work on
a 2D graphics library and is that a priority ?

In its current form, which is limited to drawing shapes, the proposal is
about 150 pages long. It’s one of the biggest proposal submitted for the
next meeting.

It can only grow bigger. There is no end to the complexity of a “small
and simple graphics library”. Every second spend on that proposal will
not be on some other work. Of course people discuss proposals they have
an interested in and discussions happen in parallel. Still. There is
maybe one person in those meetings for every 200’000 c++ developers.

### Let’s draw a triangle

A 2D graphics is the complete opposite of what the Standardization
process is good at. Standardization is all about formalism, so it works
best at describing formal things, Math, Algorithms. The more reality
gets messy, the more it’s hard to describe it put it on paper and have
that paper serves as source of Truth for decades.

The first thing one need to do play with pretty pixes is to get a
“surface”. A canvas where pixels get drawn.

So hopefully you have a `surface` class
to which you give dimensions and that gets you a canvas on which to
paint.

But wait. On most desktop systems, if you want a surface you need to put
it in a window. It’s customary for windows to have titles so a graphics
API should probably handle that, right ?

You probably also want the window to have an icon. An icon is a file on
most system, the format of which is system specific. But sometime it’s
not a path, it’s a name corresponding to a path.

The size of a window can change during the execution of the program on
some desktop operating system.

Sometimes the window can be moved to another screen that has another
resolution. And there is this weird new screens where there are virtual
pixels that are bigger than true pixels ? Unless you are rendering an
images or something then you should make sure you uses all the power of
the small crispy pixes since the customer paid a premium for boasting
about how crispy his screen is.

That woman over there was jealous so she bought a TV with 40 bits per
pixels. You can’t really see the difference but are you going to tell
her she wasted 5000 bucks ?

And then there is a screen in your pocket and IT ROTATES in all the
directions and now the surface is all wonky. But it has no window so it
has no title or icon.

What time is it ? OH GOSH THAT THING HAS A SCREEN TOO BUT IT’S SO SMALL…
Better go and read a book WTF ELECTRONIC INK that you should refresh as
little as possible and that only is black ?

World is crazy, right ? Let’s stick to Linux, shall we? So on Linux
there is this thing called X11 which you request a surface to… oh sorry,
while you are writing the paper X11 is being deprecated and now you
should use Wayland… unless you rather have a frame buffer? It can be
accelerated using opengl. or embedded opengl. totally different thing.
But really, Vulkan is faster than both these things. Oh and on this
system we prefer that you draw the windows yourself, there is a war
raging on about CSD vs SSD it has been going on for years and you can’t
take side.

And if you have CSD please make sure that I can drag the windows
properly and I have set up sticky corners so that the windows can be
aligned nicely. Make sure to handle them. Properly. And when you drag
the window it should be a bit transparent, you know about windows
compositing right ?

Okay so, you start to tell yourself that maybe drawing stuff is
complicated. Let the implementors compiler writers and library vendors
deal with all that crap. So you provide an API that works everywhere, so
it handles absolutely nothing, which is to say it works nowhere.

Now the compiler writers are a bit pissed. All they wanted in life was
to write compilers and there they are, trying to understand how GDI
work. Plus Microsoft is maybe not really interested in providing a
drawing framework, they rather have their users use the WinRT xml-based
tools. Meanwhile the GCC guys are still trying to have
`std::thread`  work on windows.

Clang people get bug reports that “it doesn’t work”. People have
expectations that the STL will work perfectly, consistently, *anywhere*

No problem. We will make the graphics library optional. So now there are
bits of the Standard Library that are not standard. If and when they are
implemented, they don’t behave quite the same on every platform. So now
the code written using standard tools is not portable. So we need to
have a copy of the STL in the repository along with messy build scripts.
Back to square one.

Maybe we messed up somewhere? Let’s look at what exists on Internet.
People have displays so surely they do write libraries for them, right ?

Turns out Qt is pretty popular. It does a lot more than displaying
triangle though. It was released in 1995. It has strings, threads, tons
of stuffs. People really did not came up with anything better since ?

wxWidgets is even older. It too has strings and threads and a lot of
things that don’t have business being in a graphic library. GTK is the
exact same thing.

But C++ goals are more aligned with things like SDL. Released in 1995
with threads and strings and weird things. Allegro, released in 1990.
Same thing

You look at other languages. Surely the Rust community has an awesome
painting framework, right ? Or the Go people ? Turns out they write
wrappers around Qt, or SDL or something, like they deemed to complicated
to start from scratch.

So 20 years later you manage to draw a triangle on all platforms. For
some definition of all.

It’s quite the achievement, so you want to share your joy with the
world. People communicate mostly using languages \[citation needed\] so
you are going to display some words on the screen, how hard can it be to
go from a triangle to that ?

#### void draw\_text(std::point2d, std::string);

You learn that there is a standard called “Unicode” that describe all
the letters people around the world use. So many letters. The Unicode
standard is about 10 times the size of the proposal you have been
working on for 5 years. Fortunately most programming languages have
backed in support for at least parts of Unicode. Except C++. Well, okay
let’s put that aside for now.

So text is rendered using fonts. The fonts are often installed on the
system. There is that thing called a font database that tells what the
font are. Unless the systems has no font database. Or no fonts. Or no
system. People also like to use their own fonts.

A font is a file, whose format is standard. There are 5 or so competing
standards.

A font file can contain glyph tables, PNGs, SVGs, scripts executed in a
virtual machine, a mix of all that. Some fonts have color, but not all
people like colors. Your children like colors. They sent you a 🐈. You
will add support for cats, right ?

You learn about subpixel rendering. You spend a few months in jail for
patent infringement. You figure you can use that time to learn about
ligatures in the encyclopædia. You start regretting being a developer
and consider a new career as a monastic scribe.

There is a lot of math involved in font rendering so you pick up a
mathematics book written a dead guy called AL-Khwarizmi. You realize
everything is written from right to left. How does that even work ?

So maybe the optional 2D graphics library should have optional text
support ?

At the next committee meeting in Toronto ( Hawaii sank in the ocean long
ago), someone is trying to write a complex graphic application with
network and lots of input and to avoid spaghetti code they like to have
some kind of event loop with maybe some threading. It’s a theoretical
concern obviously as there is no input support. Consensus was never
reached on how to name the keyboard keys.

You think back on all the existing framework like Qt, now on version
8.0, that provide an event loop, a message passing system and a Unicode
string type. Maybe they were up to something.

During all this time, people continued using Qt. People were hired for
knowing Qt. They used it in their school projects. Of course Qt still
sucks because the C++ reflection features that got added in the standard
were never quite enough to replace their code generator. But people
don’t care that its sucks. People who do use QML. Or Electron.

Having not displayed a 🐅, let’s go back to 2018.

### Has the committee anything better to do anyway ? {#c4e3 .graf .graf--h3 .graf-after--p name="c4e3"}

To be considered, a proposal has to be written and put forward, and the
library proposal exists because someone put a lot of work in it.

However, currently, C++ has

 - Poor threading support ( no executors or facilities to use coroutines )
 - No support for launching processes
 - No support for Unicode
 - Poor I/O facilities
 - Poor locale facilities
 - No support for dynamic loaded libraries
 - No HTTP support
 - Nothing crypto related

The list, of course goes on. I don’t know what is a good candidate for a
C++ library, but, according to the committee itself, a library proposal
should

 - Be useful to most people
 - Have a stable API that isn’t too subject to frequent change
 - Have real world experience and feedback. That’s why most C++ library
started their lives as boost library.

Proposal are often dismissed from the get go for not being useful enough
or not being battle tested enough. Which is reasonable given the
expectation people have regarding the stability of the STL but then
those criteria should apply consistently.

And of course there are a lot of languages features that are still in
the pipelines after years and years of work, and they should take
priority over library features since pure library addition can be
polyfilled by boost or other.

### The teaching argument

One of the argument put forward for the inclusion of that library is
that it would make C++ more teachable and that people are more
interested in graphics-based projects.\
I sympathize and fully agree with the goal of making C++ more teachable.
However, there is a difference between making sure a given feature is
teachable and adding a major feature to the language with the primary
goal of being used in classrooms.

Teachability implies easy to use, hard to misuse, and a sane mapping
between a concept and its implementation, and generally behaving
accordingly to the expectation of the majority of users. Quality that
should be looked for in any new feature.

It is also to be expected that some features are targeted to advanced
users, library writers and experts.

However, the “teaching friendly part” of C++ should be a subset of the
features used in a professional settings rather than a different set.

I would prefer that people learn to use Qt (for example) as that is a
skill they can use in their professional careers, rather that something
dedicated to teaching purposes.

I also think that a library whose scope is too limited may give a bad
image to the language. If people are told that they can’t draw emojis or
gifs or use a gamepad, they may end up thinking that C++ is not powerful
enough and switch to another language like C\#, java, javascript, swift…
But if they get to use an existing, battle tested framework, that is
powerful enough to let them implement their design (Qt, SDL) even if the
code isn’t “modern”, they will get a better grasp of what c++ can and
does do.

In other words, I’m afraid that if people are introduced to a toy
library, they will think that C++ is a toy language.

Beside, “Teaching” needs to be better define.

Are we talking about middle schoolers? And if so, is teaching them C++ a
good idea ? In some cases Python, Javascript, Lua are more suitable,
easier to grasp choices. I think that’s okay.

Are we talking about college CS 101 ? In this case, it’s probably
desirable to introduce students to build systems, libraries and package
management. tools are important. And in my experience, a lot of juniors
developers don’t know how to use their tool and that’s as important as
knowing languages. It’s also important that people know and are taught
the ecosystem. Qt, boost, wxwidgets, SDL…

### The “We need a standard library because using third party libraries is hard” argument

I think most people agree on that. Including a library in a C++ project
is a bad, often painful experience. Investing a lot of resources on a 2d
graphic library don’t solve that problem. Unless every single library
that exist or will exist gets folded into the standard, so, where do we
stop ?

And I’m sorry to say, things won’t improve on their own, it’s just not
possible. The number one requirement for a package manager of any sort
is to be authoritative. It doesn’t even necessarily need to be good. But
until individual entities are left to deal with the issue, we will
continue to have myriad of incompatible, half backed tools. I understand
that the committee prerogatives don’t necessarily extend outside the
definition of the language and therefore the issue of package management
may not be solvable. But tools, not UI is the big challenge that C++
must tackle.

Note that there are ways the committee can help improving tooling
without extending its prerogatives, notably:

 - Finding ways to supersede all reasonable uses of the preprocessor (
the work on reflection / code injection is very important for that )
 - Defining a portable C++ ABI (N4028)
 - Defining a portable module representation

Sure, these works may not be as glamorous as a 2D API, but they are more
fundamental, and more importantly, can’t happen independently of the
committee.

### Things should move forward, somehow.

After looking at P0939 and P0267, I wanted to share my wishes for work
to be done in related areas. Of course, I’m not in a position of doing
more than wishing and I can only hope to inspire someone ! But I’m
interested in what you think is important in those areas !

#### Take the Unicode bull by the horns

I would not have suggested that, as I understand why C++ lacks Unicode
but if we are seriously considering 2D graphics, then we absolutely need
to have proper Unicode support.

 - A first step is the [`char8_t` paper](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0482r1.html) . Of course that is not sufficient, but it is necessary.
 -  We need a set of algorithm to normalize, compare, sanitize and
transform Unicode strings, count characters. Something range-based could work nicely
 -  class of characters, Regexps… We might not need as many features as
ICU but we do need some. That could be a `<unicode>` header.
I'm not certain that proper Unicode support is a goal aligned with the constraints outlined in P0939,
however it would be beneficial to any application dealing with user
input/output, including GUI, databases, (web) server, console application...

I don’t know if we can qualify Unicode strings of vocabulary type but
handling the world languages is certainly something everybody needs and
it would be easier if there was a universal, idiomatic tool to do that.

#### Add geometry primitives to the standard

It could be interesting to extract the vocabulary types introduced in
p0267 and standardize them independently of graphics. Types like
`point_2d` , `matrix_2d` ( and eventually `point_3d`, `matrix_3d`  ) are
useful for graphics but may have other use, for example scientific
computation, plot manipulation. They could be accompanied by a set of
methods to perform widely used analytic geometry computation. All of
that could live in a `<geometry>`  header.

There are multiple reasons why this would be beneficial

-   It’s something every library dealing with painting or surfaces need
    `SDL_Point`, `QPoint`, `wxPoint`.
    Converting from one type to the other is cumbersome, error prone.
    All these frameworks could benefit from speaking the same language
    in the same coordinate system. It's the definition of a vocabulary
    type.
-   It’s guarantee to stand the test of time. Math are not affected by
    new technology trends and as such the API would remain stable for
    decades.
-   For the same reason, it would be probably be easy to reach
    consensus, it’s hard to bike-shed basic maths.

#### Help improving existing graphics library

What can be done by the committee to improve Qt, wxWwidgets SDL and
other graphics frameworks ? It turns out a lot of frameworks rely on
boiler plate code that is either generated by extensive and invasive use
of macros or through a code generator. Reflection and code injection are
fundamental to the modernization and betterment of these frameworks, and
this is fundamentally a language feature that should take priority over
purely library work.

### Let grow the Graphics proposal on its own

Maybe we do need another graphics framework. Who am I to say otherwise ?
But existing framework have been battle tested for 20 years. I think the
2D graphics could thrive and grow as an independent or boost library
over the next few years. Most importantly, it could provide a single
implementation working on a wide variety of platforms rather than having
5 implementations or more of the same thing.

It would be free to experiment with text rendering, inputs, events,
back-end, threading models…

I feel that this proposal as well as the package management issue calls
for *something* that is authoritative without being ISO and I have no
idea what *that* could or should be.


{{< figure src="https://cdn-images-1.medium.com/max/800/1*qvzUNZUGrfG8pbgBLRlRnw.png">}}

In the mean time, Visual Studio and Xcode could ship with more third
party libraries and that would solve at least half of the issues that
this proposal is trying to solve.
