# Compile Time

_Coupling, in the form of importing modules, is the key to causing large jumps in compiled files and eventually a blowout in compile time. Anyone who's experienced their workflow grind to a halt knows what this feels like. elm-module-graph[^1] is an excellent tool in diagnosing where your dependencies come from. Making modules Simple[^2] is the key to having a constant compile time as your application grows._

There are two kinds of issues when trying to improve compile time. Compiler/OS optimisation issues[^3] and your application architecture. The former issues are well known and likely to be addressed in the 0.19 release. The latter is the focus of this section as it will affect most codebases beyond ~10k lines. If your application is less than this, or your compile time is under 10 seconds, focus on features and do not waste time fiddling with compile time. Read something useful like [Refactoring](/chapters/refactoring.md) to reduce time spent on bugs.

#### Do you have a coupling problem?
Here's the test:
1. Compile your app
2. Touch a commonly edited module ( a page, a service, model, controller etc... )
3. How many files recompile?
4. Repeat this for a few files and get an average.

Number of Files | Health of your app
----------------|-------------------
3-5 | Healthy! Read this section to laugh at my struggles but otherwise you're wasting time here.
5-10 | You've likely already taken some evasive action, 5 is borderline, with 10, there's a margin to improve but hopefully the compiler will get smarter before your app needs to. There's not much to gain here.
10-30 | If you've just been hacking along without a care, have ~100 files, this is likely where you're at. If this is a small hobby project, it's not a big issue. If you're at work and this is sizeable production code, shit will eventually hit the fan at the velocity you're going. Stop. Read this section. Improve.
30+ | There is some serious coupling here. The xkcd on compiling[^7] is no longer funny because you're about to lose your job from lack of productivity.
100+ | Do I even need to welcome you to your own personal hell? Read on. (Seriously though, if this is your average, and this section didn't help you, I'm happy to help you personally. Hit me up on elm-slack, #compile-time, @mordrax)

So, the above was a bit of fun (need some fun after mashing out 4000 words). The serious point though is that a healthy app should really only be compiling 3-5 files _for most changes regardless of the size of the app_. This means when you hit 1000 files and 100k LoC, you're still only compiling 3-5 files. Of course this does not hold for your common files or library files because by definition, they will be imported by alot of modules but more on strategies to mitigate that in [File Structure](/chapters/file-structure.md).

To give you an idea, our application of 450+ files with 45k LoC typically recompiles 3-6 modules with page changes. Components cause a jump to about 100+ taking over 3 minutes to complete. Touching Alfred.elm or our Types.elm definition recompiles over 200 files and we take a short coffee break. Unavoidable.

#### Our story...

We had ~350 files in ~30k LoC. **A typical change affected a 65+ files and compiled in ~2 minutes.** This meant adding a Html.div or a Debug.log took 2 minutes each time. This had a _HUGE_ effect on workflow and team morale. We literally stopped working and gave birth to elm-hack (ported a version[^6] to my game), our infamous compile aide. I won't go into the details there, my talk[^4] with slides[^5] goes into a bit more detail how we were able to get from 2 minutes down to 2 seconds. This bought us enough time to get to a point where management trusted us enough to let me fix the core issue.
Fast forward 3 months, and a complete re-architect of the root modules, we're now at ~45k LoC in 426 files. A typical change will affect 3-4 files and take 15-30 secs.

_(Aside: "Complete re-architect" sounds scarrrry, ooohhhhh, a COMPLETE re-architect, aaarrrhhhh. But we're in Elm. And I had done all the refactorings I mentioned in the refactoring section. I spent 2-3 days to do some major 1-2k lines of reshuffling of modules. Then I spent the rest of the week and another 3-4k lines to rewrite routing, page loading, port existing pages over. One week, handful of bugs, not the end of the world. There is a very good reason experienced coders fear rewrites or changing core architecture, we've all experienced the pain of going through a long regression trail after such changes. Elm rewrites the rules. Refactor fearlessly.)_

#### Case study ( Quotes )

To explain why coupling is bad for compile time, I'm going to use a work related example.
In health insurance, there is a concept of _Person_, these people take up a either a single, couple or family _Membership_.

```haskell
-- Person.elm
type alias Person =
    { id : Int
    , -- more fields
    }

-- Membership.elm
type alias Membership =
    { id : Int
    , -- more fields
    }
```

There is a large list of pages which handle various aspects of both:

```haskell
-- Medicare.elm
import Person exposing (Person)

-- ClearanceCertificate.elm
import Person exposing (Person)

-- Rebates.elm
import Membership exposing (Membership)

-- etc...
```

A feature of insurance is we get a quote for a single person or a membership.

```haskell
-- Quote.elm
type QuoteID
    = PersonID Int
    | MembershipId Int

type alias Quote =
    { id : QuoteId
    , -- more fields...
    }
```

Now, since one can make a quote in Person or Membership, it makes sense to add it to the respective models.

```haskell
-- Person.elm
type alias Person =
    { id : Int
    , quote: Quote
    , ...
    }

-- Membership.elm
type alias MemberPolicy =
    { id : Int
    , quote: Quote
    , ...
    }
```

Visually this looks like the following:

![abc](https://preview.ibb.co/mWjMfx/Screen_Shot_2018_03_02_at_9_10_56_pm.png)

Great, we've got some code sharing here. Both Person and Membership now share the Quote component and we're happy! ( Of course, the avid reader would know that's not how the story goes.)

```haskell
touch Quotes.elm
elm-make ...
Compiling 33 modules
```

According to my highly accurate 'Do you have a Coupling problem?' chart, we ranked (30+):
> There is some serious coupling here. The xkcd on compiling is no longer funny because you’re about to lose your job from lack of productivity.

So we had a common component, it was added to the model where it was needed most, then the view and update logic are shared. _What's the problem?_

Here's some food for thought:
```
touch Person/Page1.elm
elm-make ...
Compiling 1 module

touch Membership/Page15.elm
elm-make ...
Compiling 1 module

touch Membership/Model.elm
elm-make ...
Compiling 15 modules
```

**When you touch a module (Quote.elm), it will re-compile *ALL* modules (all pages, person/membership models) that directly or indirectly import the changed module.**

When does it not do this?

Myth #1: If I change a non-exposed function in Quotes.elm it will not re-compile all modules that import it.

Myth #2: If I make my Quote type opaque and only export the opaque type, it will not re-compile all modules that import it.

Myth #3: If I add a comment to Quotes.elm, it will not re-compile all modules that import it.

Fact: Doing anything to Quotes.elm will re-compile all modules that import it.

This is the reason for all non-compiler/OS related compile time blowouts and it can happen very quickly, just with the introduction of one feature, badly coupled.

There is an excellent tool for looking at this specific scenario of coupling in your app, elm-module-graph[^1]. And for the case study, it produces the following graph:

![elm module graph demonstrating coupling](https://preview.ibb.co/jNG3Sc/Screen_Shot_2018_03_02_at_9_30_51_pm.png)

So Quotes is the highlighted module. It has 11 blue modules lit up, these are the modules that either import it directly or indirectly. There are 3 black lines that comes out of quotes, these are the modules that import Quotes directly. The rest of them come out of Model.Person because Model.Person imports Quotes.

This is how it looked after I _removed Quotes from the Person/Membership_ models and passed it into functions that required it.

![Refactored version of quotes](https://image.ibb.co/djq2Lx/Screen_Shot_2018_03_02_at_9_36_22_pm.png)

So now Quotes is directly imported by Membership and Person, two TEA modules and this is imported by Main.

```haskell
touch Quotes.elm
elm-make ...
Compiling 4 modules
```

Let's count that, Quotes, Membership, Person, Main. _Four_!

> Healthy! Read this section to laugh at my struggles but otherwise you're wasting time here.

So with this pattern, even if your app reaches 426 files, your compile time will not grow with it.

[^1]: elm-module-graph: https://github.com/justinmimbs/elm-module-graph
[^2]: Rich Hicky's Simple Made Easy: https://www.infoq.com/presentations/Simple-Made-Easy
[^3]: Compile time issues and workarounds: https://gist.github.com/zwilias/7ed394ec0e9c6035e1874d19b721e294
[^4]: Elm Remote Meetup talk - https://youtu.be/ulrukPRYsws?t=50m4s
[^5]: Elm Remote Meetup slides - https://docs.google.com/presentation/d/10vN7eLr3qsd4nK2zcxUHgbu68fO_yzAA-b7Hf7kEcik/edit?usp=sharing
[^6]: elm-hack.sh, a script to compile single modules VERY fast - https://github.com/mordrax/cotwelm/blob/master/hack.sh
[^7]: What we do when Elm is compiling - https://xkcd.com/303/