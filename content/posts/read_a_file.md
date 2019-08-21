---
title: "A Json Parser Part 1 : Reading a file"
date: 2018-05-23T20:03:48+02:00
draft: true
---

The year is 2021. A rainy Sunday afternoon.
It's getting increasingly more difficult to ignore the looming shadow of *The Great Rewrite*.

But you ran out of cacao and there is nothing on HBO.
So you guzzle a fourth cup of coffee and decide to start a new project. In C++ of course.
You sacrificed too much to reevaluate your life choices now.

But you need to do something new. Something to impress, Something to shine, like a diamond.

Something bold. Pionerring:

**A command line parser !**  <small>Unfortunately, someone already did that, apparently.</small>

**A console logger !** <small>That too was already done.</small>

**A console emulator**  <small>.... too complicated....</small>


That's it! Let's build a **JSON parser**. Truly novel, and simple.


First, you setup some cmake scripts.

The year is 2022.

# Reading a file

You figure that to make any kind of parser, you will probably need to read a file, so you can lex it.
Let's open a file:

```cpp
#include <fstream>
```

You reevaluate your life choices.

`<fstream>` and all of `<iostream>` is _terrible_.
It hard to say exactly why.
In some case, [a benchmark reveals](https://cristianadam.eu/20160410/c-plus-plus-i-slash-o-benchmark/) it may have worse performance than manipuling a file handle directly,
but results aren't exactly clear cut.
