---
title: "Characters sets: A bad idea since the bronze age"
date: 2019-04-27T16:18:53+02:00
draft: false
---



{{< figure src="behistun.jpg" title="You who shall hereafter see this tablet, which I have written, or these sculptures, do not destroy them, but preserve them so long as you live!" >}}



In 522 BC, 𐎭𐎠𐎼𐎹𐎢𐏁 also known as Dārīus was king of the Persian Empire.
Kings crave fame as they do power and so Darius (who the greek later called Δαρεῖος)
had his henchmen carve his name in stone.
One such stone is the Behistun Inscription, which is really more a mountain than a stone.
And while having your biography carved on the face on the mountain is definitively a sign of success,
it doesn't mean much if people can't understand what is written.

From what I can gather the Behistun Inscription seats somewhere near the crossing of three empires: Babylonia, Persia, and Elam.
To ensure his greatness was known of all, the king has his biography translated into three languages:
Old Persian (known at the time as just "Persian"), Elamite and Babylonian.
Sure, it might not be as impressive as Harry Potter, but J.K. Rowling didn't carve her books on the face on
a mountain.

Papyrus with an Aramaic translation of that mountain were also found, which tend to indicate that, in the absence of netflix, a lot of people were keen to know about
Darius the great.

As all great kings, Dārīus died and like all empires, the Persian Empire fell.

A short while after that, people discovered they could turn sand into bugs, and so computer science was born.
At the time, sand was expensive and so IBM found in the late 50' a way to encode a character using 6 bits.

It was, however, one bit more than the International Telegraph Alphabet used a few decades before.
The International Telegraph Alphabet was itself derived from the Baudot code.
The Baudot code offered the upper case letters A to Z and the letter É.
That Letter É was very convenient if you happen to know someone called Émile.
For example Émile Zola, or Émile Baudot inventor of the Baudot code.

Like all good International Standard, the International Telegraph Alphabet was declined into a dozen slightly incompatible
versions.
We tend to see history negatively. At the time the American civil war was raging, Europe was going through a wave of revolutions
while the rest of the world was suffering the ravages of colonialism.
But in truth, it was worse than that: To send telegraphs, people had to queue at the post office only to endure mojibake more than a century before the term was coined.

But as I said, in spite of all of that, in the late 1950s, IBM elected to use a 6 bits encoding which they called binary-coded decimal (BCD).
The president of IBM at the time was called Thomas J. Watson. Maybe that's why, unlike the Bacon cipher, the IBM 704 BCD encoding
had a J. And maybe that's why it didn't have an É. Émile is an artist name, not an appropriate name for the CEO of International Business Machines.
The 704 BCD encoding could represent 64 characters so IBM carefully chose 51 characters. Including ⌑ and ‡ known to IBM as a Recordmark.
In the Unicode Standard, a good substitute is U+2021 DOUBLE DAGGER. Because punching cards might not appease you enough when dealing with all of this madness.
There was apparently so many variations of the BCD encoding that Wikipedia does not care to list them all.

By the time IBM realized 6 bits might not be enough for anyone and came up with EBCDIC, Bill Gates was born and people in Japan realized computers were kinda cool.
It has now been 245 years since one guy decided to send letters over an electric wire instead of having a nap.
Unicode was invented in 1991 and we are still debating whether we should use it consistently.

But here is the thing:

Text is for people.
People travel and they mingle.
They drink caffè with their fiancé. They have stupid names, like Ó Briain. Ó Briain is in the kitchen, cooking jalapeño and chouriço.
Of course, you might argue proper Shakespearian Ænglish of þe old never had to suffer this nonsense as people only used sensible letters.

Developers are not really human, they are happy with [a-Z] and that's perfectly fine.

Normal people, they use text.
And there is no way you can predict which character people will want to use in your system.
Some idiot may decide to put some some old Persian on their website (BTW Dārayauš was called <span class="Aramaic"> 𐡃𐡓𐡉𐡅𐡄𐡅𐡔 </span> in Aramaic)

The idea of alphabets started to get tenuous when people invented horses and boats.
The idea of characters sets was bogus from the start.
The idea that characters are limited to a place is just crazy talk.
And maybe it was okay for Gutenberg to drop letters he was too lazy to carve, but you don't get to make that choice.


So, your legacy code? It's broken. [The strongest man alive](https://en.wikipedia.org/wiki/Haf%C3%BE%C3%B3r_J%C3%BAl%C3%ADus_Bj%C3%B6rnsson) is so strong he can break your database with his name. That's right.

Would we know of Dārīus the Great if the Behistun Inscription had been contracted to IBM?

How much energy should we waste in supporting anything but Unicode?


<style>

@font-face {
  font-family: Aramaic;
  src: url("/NotoSansImperialAramaic-Regular.ttf");
};


@font-face {
  font-family: OldPersian;
  src: url("/NotoSansOldPersian-Regular.ttf");
  unicode-range: U+103A0-103DF;
}

.Aramaic {
	font-familly:Aramaic !important
}

</style>


