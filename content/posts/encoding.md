---
title: "C++ compilation: Fifty shades of Mojibake"
date: 2019-08-17T11:46:12+02:00
---

Interestingly, writing was initially invented as a way to keep track of numbers.
Words came much later.

Computers are good at numbers. It's the only thing they understand really.
So text has to be represented as a sequence of numbers which are interpreted
and ascribed meaning.


Code, in the presence of arbitrary identifiers and string literals as to be considered as text.
In the context of C++, how is the text of our program interepreted and transcoded during compilation?

Let say we want to execute this program:

```cpp
#include <iostream>
int main() {
    std::cout << "Γειά σου Κόσμε";
}
```

Possibly, what the compiler sees looks like this:

```
23696e636c756465203c696f73747265616d3e0a696e74206d
61696e2829207b0a202020207374643a3a636f7574203c3c20
22ce93ceb5ceb9ceac20cf83cebfcf8520ce9acf8ccf83cebc
ceb5223b0a7d0a
```

These numbers represent characters, but which numbers represent which characters?
How many bytes are used to represent individual characters?

## That's where encodings come in.

An encoding is a method by which a sequence of 1 or more bytes is mapped to something we
understand as being a character.
There are some nuances there: there are a variety of encodings.

* Some encodings will map 1 byte (or less) to a unique character which means they can represent a ridiculously low number of characters - that is for example ascii or ebcdic.

* Some encodings will map a fixed number of bytes (often 2) to unique characters. 
Still widely insufficient to represent all characters used by people. That's for example UCS2.

* Some encodings will have a variadic number of bytes per characters, which make them memory efficient at the cost of 0(n) indexing - this is for example UTF-8. 

Ok, I lied.
Encodings don't map to characters. _Character_ is a really fuzzy, hard-to-define term.
Some encodings map to glyphs - basically an index into the font file -  while more modern encodings map to a code point
which is a number assigned to a character or part of a "character".

In any case, each encoding maps to a character set which is, to simplify the set of characters and an encoding can represent.

An encoding maps to one specific character set, while the same character set can be represented with different encodings.
For example ASCII is both an encoding and a character set, while UTF-8 and UTF-16 are two encoding that map to the **Unicode** character set.

You can find the definition of all these things on the [Unicode glossary](https://unicode.org/glossary/)

We have been encoding text for machines for over 150 years, and because of reasons that made sense at the time we have a lot of encodings.

Over 250 officially registered.


# Physical source file characters

You are caught up on the basis, so what's the encoding of the above snippet?
And therein lies the rub: We don't know, the compiler doesn't know.

Encodings are not stored along the rest of the sequence of bytes that constitute
our piece of text. Encodings are not something we can observe. 

But we can't possibly interpret that sequence of numbers without
knowing which encoding was used to create it.
Just like you can't interpret a language without knowing what language is spoken.
(You can't of course have text without encodings, like you can't have words without language.)

Of course, we can ask the user, maybe the user knows (haha).

Both GCC and MSVC have an option for that (`-finput-charset` and `/source-charset` respectively).

That works as long as all your headers included in a given file share the same encoding.
Do you know how the files that make up you third party libraries were encoded?
Probably not.
Might as well guess.
Which is what compilers do by default. They guess.

Clang and GCC guess everything is encoded in UTF-8, while MSVC derives the encoding from the locale of the computer you are compiling your program on.

MSVC assumptions work great as long as people don't try to ever share their code,
especially with people living in a different country, or using a different operating system.
But why would anybody ever do that?

You may have noticed that as long you stick to ASCII encoding,
your program will compile just fine. This is because most 1 byte encodings, including UTF-8 are ASCII supersets - so they have the same mapping as ASCII for all
codepoints in the ASCII range.
The biggest exception to that is EBCDIC which is only used on IBM systems.
Shift-JIS, - an encoding suitable to encode Japanese [^1] - is mostly ASCII compatible with a couple of exceptions.

This is the first reason why people tend to avoid non-ASCII characters in source code.

But what if you really want you have Greek in your source file?
Well, GCC and clang will already support that as they assume UTF-8, MSVC has an option to interpret files as UTF-8, so everything is great, right?

Well, not so fast.
First, that puts the responsibility on downstream code they compile your code with the right flags.
So some information **necessary** to build your code is offloaded to the build system, which is brittle and a maintenance burden.
And as I said, compiler flags operate on translation units whereas you want to set the encoding on individual files.
Modules will solve everything as in a fully modular world 1 file = 1 translation unit.

In the meantime, maybe we can put the encoding in the source file, like [python does](https://www.python.org/dev/peps/pep-0263/)?


```cpp
#pragma encoding "UTF-8"
#include <iostream>
int main() {
    std::cout << "Γειά σου Κόσμε";
}
```

There is a couple of issues with is.
First, it doesn't work for EBCDIC encodings at all.
If interpreted as EBCDIC, the above UTF-8 file might look something like
that 

```
?/_/?>?>??????>%??/_??>?_/>???#???????????l?ce?c???'?\
```


_Doesn't look like C++ to me._




Ok, so let's not care about EBCDIC[^5], as people working on these systems already have to transcode everything.
We can use that directive at the beginning of all and single files that is UTF-8?

Except [UTF-8](http://utf8everywhere.org/) is the right default, all open source code is UTF-8, and compiling in UTF-8 is at this point standard practice.

So forcing people to write `#pragma encoding "UTF-8"` for the compiler to assume
UTF-8 would be the bad default.

Maybe we could force compiler to assume UTF-8 unless otherwise specified by a pragma (or some other mechanism)?
That would break some code. How much is anyone's guess.
Re-encoding an entire codebase from any encoding to UTF-8 should be a straight forward,
not breaking operation in most cases, but, ironically, it is likely that some encoding test code would break.

Nevertheless, very few languages don't assume UTF-8 by default, except of course C++.
And it's becoming necessary, as every compiler speaking the same language as immediate benefits.

First of, the UTF-8 string `const char8_t * = u8"こんにちは世界";` might be interpreted by MSVC 
as `const char8_t * = u8"ã“ã‚“ã«ã¡ã¯ä¸–ç•Œ";` [on many windows machines in the US an western europe](https://en.wikipedia.org/wiki/Windows-1252).

Not what we want.

Because of course `u8` string literals are not strings in UTF-8, but strings that will be converted from the source encoding to UTF-8.
This is [confusing and not portable](https://stackoverflow.com/questions/23471935/how-are-u8-literals-supposed-to-work?rq=1).


But of course it gets worse.
Some compilers accept identifiers composed of codepoints outside 
of the basic source character set supported by the standard[^2].

This poses interesting questions:

* Can we mangle portably these symbols?
* Can we reflect portably on these symbols?

If all parts of the systems do not expect and produced UTF-8,
the results are inconsistent and therefore, not portable.

I have no idea what the committee will elect to do,
but I hope we will at least find a way to push implementers and user gently towards more UTF-8 sources files.


Which is not even one half of the problem.
Because so far, we only converted the source to the internal encoding - which is not specified but can be thought of as being Unicode.
So internally, the compiler can represent any codepoint. Great.

`u8`, `u` and `U` character and string literals get then converted to UTF-8, utf-16 and utf-32 respectively, 
which is a lossless operation.

So if you have a u8 literal inside a UTF-8 source file, it will be stored in your program memory unmodified - 
although this is not really guaranteed by the standard, an implementation could for example normalize unicode strings.
Great!

But then, there are `char` and `wchar_t` literals. This is where things start to really fall apart.

So, remember that all strings need to be encoded to _something_. But what?
C++ will encode all literals with the encoding it think will be used by the operating system of the computer the program will run on.

Most compilers have an option for that but by default, implementations will assume that this is the same encoding as the one 
derived from the locale of the environment the compiler is running on.

This is the **execution encoding**.

# Presumed execution encoding

The deeper assumption of course is that the Internet does not exist or all people have all the same locale[^3]
or there is a binary per encoding.

This of course works wonderfully well on most linux/OSX/Android systems because all components talk UTF-8,
so the compiler will convert literals to UTF-8, which will then be interpreted as UTF-8 at runtime.

Using MSVC on the other end, the execution encoding, by default, will depend on how your Windows is configured, 
which basically depends on where you live.

All of that raise interesting challenges...

* Conversion from Unicode to non Unicode might be lossy. So they are lossy. 
Implementations are not required to emit a diagnostic and MSVC will happily drop characters 
on the floor[^4] while GCC will make that ill-formed.
* Of course, the assumption that the machine where the code is compiled matches the one of the machine is run is not illustrative
of the reality.
* The presumed execution encoding is not exposed so the only conversion functions you can use are 
the [delightful ones](https://en.cppreference.com/w/cpp/string/multibyte) provided by the C and C++ standards.

# Oh, so you want to execute your program?

At runtime, your program will be confronted by standard facilities such as `iostream` that might (loosely) transcode
your text to what they think the environment expects or produces (using wonderful interfaces such as [codecvt](https://en.cppreference.com/w/cpp/locale/codecvt) and [locale](https://en.cppreference.com/w/cpp/locale).

Or worse, strings that you want to display but you don't know their encodings (because they come from a part of the system you don't have control over),
or strings that are simply not text - for example, paths are considering non-displayable bag of bytes on some platforms.

And of course, many systems will produce UTF-8 which simply cannot be converted in the narrow encoding if it is not UTF-8, leading to loss of data - and therefore meaning. 

Unfortunately, the standard is somewhat limited there as there is nothing it can do to control its environment.

Windows users can rejoice that it is becoming easier to have well behaving UTF-8 strings in your program thanks to
the combination of:

* The [`/utf8` option](https://docs.microsoft.com/en-us/cpp/build/reference/UTF-8-set-source-and-executable-character-sets-to-UTF-8?view=vs-2019) of MSVC
* The new [windows terminal](https://github.com/microsoft/terminal) that should be able to support the full range of unicode codepoints depending on font availability.
* An [ongoing work](https://docs.microsoft.com/en-us/windows/uwp/design/globalizing/use-utf8-code-page) to support UTF-8 in the system API - alleviating the need for `wchar_t`.

I've started to work on a [project to illustrates how this works](https://github.com/cor3ntin/utf8-windows-demo).

That doesn't solve the problem for EBCDIC platforms and legacy codebases.

Alas, it doesn't appear that the standard will be realistically able to move away from non-unicode encodings any time soon, and the
narrow and wide literals are here to stay.

Therefore, to properly support text, the standard might have to add `char8_t` overloads to any standard facilities dealing with text,
from I/O to reflection, DNS, etc.

I do not think it is worth it trying to patch `<locale>` or `<iostream>`, as the assumptions they were designed on are simply no longer valid,
nor do I think it is worth trying to deprecate them as so much code depend on them.

It will be interesting to see how that pans out from an education perspective. Nevertheless, that duplication is probably a necessary evil;
Improved Unicode is what ultimately lead to [Python 3](https://snarky.ca/why-python-3-exists/) and we might want to avoid that in C++.

[^1]: For a very loose definition of "suitable". Shift-JIS can only encode a bit over 10% of Japanese characters.
[^2]:
    ``` a b c d e f g h i j k l m n o p q r s t u v w x y z
    A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
    0 1 2 3 4 5 6 7 8 9
    _ { } [ ] # ( ) < > % : ; . ? * + - / ^ & | ~ ! = , \ " '
    ```
[^3]: It hurts writing that because the idea that locale and encoding are tied to begin with [is bonkers](https://github.com/mpv-player/mpv/commit/1e70e82baa9193f6f027338b0fab0f5078971fbe) to begin with.
    But remember these assumptions were made 70 years ago.
[^4]: [I am hoping to make that ill-formed](D1854.pdf).
[^5]: [C++ is mostly an ASCII-centric language now](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4210.pdf)
[^6]: _ã«ã¡ã¯ä¸–ç•Œ_ in Japanese.

























































