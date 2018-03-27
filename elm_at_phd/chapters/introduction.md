WIP: Draft 22/3/2018

# Introduction


_This is a story of a guy who left it all to chase Elm full time, the technical trials and tribulations he faced, the euphoric and … not so euphoric bits of writing a SPA in Elm. My scope is ambitious, use these italic summary headers to decide on the sections you want to read._

Total word count: around 12,000

Reading time: 60 mins. + Code snippets

I write Elm at Pacific Health Dynamics, a small health insurance software vendor with big dreams. Since July 2017, I’ve been leading the frontend rewrite of their flagship product.
The codebase was at 16k LoC when I started. Since then, I’ve rewritten the various subsystems at least once (I'm looking at you _generic form component_). Now we hover around 45k LoC with most of the common SPA structures stabilizing. We are about a third of the way to completion. This is a typical massive, legacy, regulated system.

The driving motivation for this article is to share my experiences, discuss the various challenges we faced and provide practical solutions that we took to overcome them. I feel that there is not enough long form analysis from teams using Elm and that is something I'd craved when deciding whether to take on Elm in a professional setting rather than just as a hobby.[^cotwelm] Therefore, this article is aimed at those similar to where I was, not sure whether to take the step or for teams who are using Elm and struggle with the various topics I cover here. To that effect, the code samples here are not for beginners of the language and my focus is not in explaining basic concepts.



# Overview

So, at its heart, this a story. And a story, always has a beginning ([Prologue](#prologue)).

The next chapter ([Brave new w..holy cow](#brave-new-w..holy-cow)) details my analysis of the existing codebase when I started.

From there, we go straight into strategies for [Refactoring](#refactoring) large codebases. That was when I had alot of fun digging into [Compile times](#compile-time) which (prodded by Noah) produced a talk at a Elm remote meetup[^elmremotemeetup] with slides[^compiletime].

With a much more structured foundation, I started developing patterns for [Child to parent](#child-to-parentchildsiblingfirst-cousin-2nd-removed-communication) communication, our various types of [Components](#components) matured, and I tackled generic [Forms](#forms) over and over, which I'm still unhappy with.

A core piece of our frontend is how it interacts with the backend through our [Endpoints](#endpoints) which consist solely of auto-generated decoders. [Extensible types](extensible-type-hell.md) are powerful but also fun!

I also go over our [File Structure](#file-structure), which mostly grew organically, with the main restriction given to how it affects module coupling and in turn, compile time.

Finally, I [rant and rave](#Final-words) about how much I love Elm and try my best to word it like it's an objective observation of the strengths and weaknesses of the language.

[^cotwelm]: www.google.com