---
title: "What is C++ ?"
date: 2019-01-06T10:06:24+01:00
---

These past few weeks have been quite difficult for me.
I have therefore not followed closely the ongoing discussions about C++, ranges, game developers and `iota`.

I'm afraid my current outlook on things is rather cynical and I've been told I might be too assertive and opinionated.
So, rather than another exercise in quixotism, or a pointless opinion on how best name a function that creates a sequence, let me ask a few questions.
Open-ended questions that have no bad answers.

This format is inspired by a surprisingly enlightening brainstorming session the committee had in San Diego, trying to define "freestanding".

## So, what is C++?

C++ is a programming language.

Is C++ a general-purpose programming language?

Is C++ a programming language for system programming? What kind of systems?

Is C++ a programming language for embedded platforms? What kind of platforms?

Is C++ portable or suitable for writing portable applications? What does portable mean?

Is C++ suitable for concurrent programming? Asynchronous programming? Heterogenous programming?

Is C++ a superset of C ? Is C compatibility still important? What is the cost of that? Is C++ _oriented object_?
Is C++ modern? What does modern mean?

Is C++ an ecosystem? If so, what is the shape of that ecosystem?

Can C++ be successful without tooling? Should toolability be higher in the committee priorities?

Should the committee take a larger role in the ecosystem? Does the ecosystem need shepherds?

Is C++ is a community? Who is that community? Who are the 3 million or so developers who use C++?

Are the people using C++ and the people designing it on the same page? If not, does that mean C++ is overused or wrongly used?

Is C++ an _expert friendly_ language? How many people _know_ C++? Should C++ be taught in Programming 101? Is C++ taught correctly and can that be improved?

Is C++ usable by individual developers? Small team? Large teams?

Is C++ easy to use? Does it make simple things simple? Could it be easier? Are simplicity and performance antithetic?

Should C++ offer ways to make simple things simpler if that means more work for the committee and the implementers
(given that designing easy to use interfaces often require more effort)? Is complexity necessary?

Is C++ successful at being a zero-cost abstraction? What does that mean?
When people talk about performance, do they mean efficiency? predictability? determinism?

Is C++ consistent? What does consistency mean? Is consistency important?

Is there One True C++ or are there a multitude of dialects? What are the dialects? Are dialects an issue? Are they necessary?

Is there a disconnect between _The standard_ and the way C++ is used and implemented?

Does compiling with exceptions disabled make a program not C++? Is C++98 C++ ? Is Qt C++?
Are ever-changing best practices an issue in regard to maintainability?

Is the _Standard Library_ a shipping vehicle for various facilities or a first-class citizen? Should C++ be usable without the standard library?

What should the scope of the standard library be?

Is the _Standard Library_ illustrative of how libraries should be written? Should it be?
Should the committee standardize existing practices or lead the way?

Does the _Standard Library_ have the same performance concerns than the core language? Should it?

Does C++ evolve to fast? Too slow? What is the adoption rate of new standards?

Has most C++ code already be written?

Is the primary use case of C++ the maintenance of 30 years old codebases?
Is it important for old codebases to be compatible with newer standards? Is it in practice the case?

Is C++ suitable for new projects? What are the alternatives?
Does C++ benefit from cross-pollination with other languages? Should it?

Should C++ operate more breaking changes? Can these changes be toolable?
Are new languages easier to develop than tools?

Is ABI important? Do ABI concerns hinder the evolution of C++?
Do ABI concerns make the standard library suffer from design or performance issues? Is that acceptable? Can ABI be made a non-issue?

Should there be more API breaks or more aggressive deprecations? Should there be a STL2 or would that tear the ecosystem apart?
Would implementers go along with API or ABI breaks?

Is the compilation model still suitable to the way C++ is used today? Can it be improved upon?
Is it still important that C++ be designed in a way that it is compatible with "dumb linkers"?

Should compilers be build systems? Should there be a standardized way to build C++?

Is code distribution and reuse an important concern? Should code reuse be easier? Can it?
If making code distribution easier requires stricter rules pertaining to code organization, is that acceptable?

Is compilation speed important? Is debug speed important? Can they be improved?
Is having 4+ compiler architectures still useful?
Is implementing _The Standard_ still a reasonable endeavor? Are implementers spread too thin?

Is the standardization process effective? Is it open enough? Known enough?
Should more of it happen online?
Are papers the right model? Are there too many papers?  Should standardization be less accessible? more?
Are users interests represented enough in the committee? Or does the standardization process is biased toward a few use cases and users?

Is the papers model biased toward small changes and local fixes?
Should papers be more encompassing and offer consistent, unified solutions to common problems? How do we prevent such papers from being shut down?
Should the committee work toward more ambitious goals and do more design?

Is the scope of the standard sufficient to answers to all the challenges C++ faces? Should this scope be extended? Can it?
Is _The Standard_ the only tool we have to influence the C++ development?

* * *

There are no right answers to these questions.


You would find that the committee members would not agree on most of these.
C++ is used in a lot of industries for a variety of reasons by people with very different background.

And even if C++ has some core design philosophy, the answers change as the programming landscape evolve,
the community grows and new hardware and problems come about.

I think it's important to keep these questions in mind when writing or evaluating papers, or simply talking about C++.