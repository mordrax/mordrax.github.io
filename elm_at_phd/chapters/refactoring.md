# Refactoring
_Refactoring typically leads to better code. The effect is cumulative. The opposite is also true, a fear of introducing regressions leads to a cumulative growth of technical debt. To refactor a large codebase: start by tightening primitive types into ADTs, then make Msgs opaque (models too if you're not confident enough to keep it single responsibility), take a minimalist approach to function args then what modules expose and FINALLY remove impossible state because now the compiler will have your back on these large state changes._

In my last two years of using Elm, the strongest attraction has been the ability to change logic and core components with confidence at any stage of a product's lifecycle. Time and again we accrue technical debt from fear of breaking things, "It's not worth the risk". Eventually, these areas of the code become a liability, but it's also too risky to modify. Elm's compiler gave me confidence that no amount of unit or integration test coverage[^cotw-testing] ever did. So improve the architecture often and fearlessly. This momentum to create and confidence in my code is the drug that keeps me hooked.

For larger codebases, the order of refactoring is important because type changes cascade and primitive types make regressions much more likely. The aim of these refactors is to reduce the scope of changes and tighten the types so the compiler is able to help us.
Hence we start with converting primitives to ADTs.

#### Primitive types to ADTs

Replace all Int or String types which represent a finite group with an ADT.
For us, this was all enum types and adhoc state transistions ( success, failed, haven't tried ).

```haskell
case formType of
"Limits" ->
"Benefits" ->
_ -> empty -- (NEVER do this. What happens when you add a new value to the type?)
```
Replace things like the above with the following:
```haskell

type FormType =
Limits | Benefits

case formType of
Limits ->
Benefits ->
```

Consider using tagged types instead of primitives to represent IDs. In our app, we have about 50 types of ids, most of them Ints, some Strings, others... `Maybe Float`, we won't talk about those.

```haskell

type alias Person = {
personId : Int
}

-- replaced with
type alias Person = {
personId : PersonId
}

-- You'd have to be on a special spectrum of 'special'
-- to mistakenly mix up ids ever again.
type PersonId =
PersonId Int
```

This is the best 'bang for buck' refactor _and_ the first in order of what to refactor. When you spot it, change it! It affects only the specific argument or field that uses it but by using ADTs, the compiler starts to understand your domain. Without this, refactoring in Elm will feel as unsafe as any other language. **Use ADTs**

#### Opaque Msg and Models ( to a lesser extent )

For modules with a `type Msg` definition (your typical `init, update, view?` module), make Msg opaque.

_No module should be able to see another module's internal Msgs._

This is a sign that the two modules have overlapping responsibility and the modelling isn't quite right. The exception we use is for common Msg types which is used in places such as [Routing](#routing) and [Child to Parent Communication](#child-to-parentchildsiblingfirst-cousin-2nd-removed-communication)

_Modules should not return a parent model_

I feel like I shouldn't have to say this, but I've seen it at my workplace and the consequences of this are catastrophic in the coupling that it creates. Use a parent model's data but only ever update your own model.

If you're seeing code like:
`model.login.name.first ++ model.login.name.last`
or you're trying to perform nested model updates:
`{ model | login = { login | name = newName } }`
then, making models opaque may help here.

The idea is that each module should have responsibility of its own logic. ie, login should be responsible for giving back the first and last name in a `fullName: Login -> String` function rather that the parent reaching in to manage this. Using opaque models _forces_ the coder to go down the correct path in this case at the cost of extra destructuring.

If one module is importing another module's Msg, then **always decouple from the child node**. This is because the child has to make the more generic Msg to pass up the chain. If you start refactoring from the parent, you'll end up having a domino effect that forces updating all Msg types in one go. Theoretically it should work out, it just takes a long time and sometimes you'll encounter some known compiler bugs so best to keep the scope small between successful compiles.

#### Be a minimalist when exposing functions and the arguments of those functions

So, only type defining modules or helper modules should use `module xyz exposing (..)`, all others should expose the absolute minimum. Do you know how hard it is for a new developer to join a company with a sizeable existing codebase and see **Every. Single. Module. Expose. Everything**? Why even have modules?

```haskell
-- compare
module Login exposing (..)
module RouteTypes exposing (..)
module Logic exposing (..)

-- to
module Login exposing ( init, update, view, Msg, Login )
module RouteTypes exposing (Routes(..))
module Logic exposing (when, filterBy, ifThen)
```
These modules tell a story just by looking at their exposing. It doesn't matter that Login has a function called `authenticateUser`, or that RouteTypes defines more type aliases, when it's not part of exposing, it's _clearly_ not designed to be used externally.

Make functions single responsibility and simple. It is not cool to have all views take in the top level Model. Simplicity in this context does not mean basic or small, it means it takes exactly what it needs to perform exactly what the name implies.

```haskell
-- simple validator in Login.elm
validate: Login -> List String

-- still a simple validator in AddressEditor.elm because it takes in exactly what it needs to perform the validation
validate: Lookup -> Address a -> AddressEditor -> List String

-- not simple
validate: ParentModel -> List String

-- even simpler validator in Login.elm which is now usable in other forms, more on this in Components
validate: UsernamePassword a -> List String
```

The reasons why you'd want to keep your function arguments as simple as possible is because:
1. You can then pull these functions out to be composed with other data types.
2. Bugs will be much easier to narrow down since all functions only take in what they need and so typically only a handful of functions deal with any given area of your app.
3. These types then act as great code documentation, compare

```haskell
view : Model -> Html msg -- are we rendering the whole mode?
view : PersonDetail -> Html msg -- actually, just the person
```

#### Model the minimal set of state needed (remove impossible state)

The first time I heard the term 'impossible state' was from Richard Feldman's talk[^impossible-state]. This is about the ability to model your business/app state exactly as it is, no more, no less. I do this last because state changes tend to have a domino effect and logic changes are the only way regressions can be introduced into the codebase. With the aforementioned refactoring steps (ADTs, opaque Msg, modules decoupling), the compiler is able to help minimise errors alot better.

Here's an example of finding and removing impossible states.
We have a Grid component, with the ability to show more columns via a 'Show More' toggle button. This feature can also be disabled.

```haskell
type alias Grid msg =
{ hasShowMore : Bool -- whether the feature is enabled
, showMore : Bool -- on/off state of the toggle button
, toggleMore : Maybe msg -- the event fired when toggled
}

-- how many possible states are here?
-- 2 x 2 x 2 = 8 states in total

-- examples: --

-- 1. the feature is disabled but there is a msg? What is the correct intention here?

hasShowMore = False -- feature is disabled
showMore = True -- toggle is on
toggleMore : Just ... -- a message exists


-- 2. feature is enabled but goes nowhere, wasted a couple of hours here alone ( early days )

hasShowMore = True -- feature is enabled
showMore = True
toggleMore : Nothing -- but clicking does nothing (constant source of bugs)
```

How many states should it have, or rather, what should the logic for this feature be constrained by?

```haskell
type alias ShowMoreToggle = Bool -- on/off state of the toggle button

type ShowMoreState msg
= Hidden -- feature disabled, no state
| Visible ShowMoreToggle msg -- feature enabled, toggle state + msg
type alias Grid =
{ showMoreState : ShowMoreState
}
-- How many states now?
-- ShowMoreToggle = 2
-- ShowMoreState = 1 + ShowMoreToggle
-- Total of 3 states where the type declarations are self documenting
```

So these sorts of refactoring are more time consuming because they will force you to handle all the possible states. On the positive side, you'll _never_ have to consider the invalid cases, not in tests, not in debugging, they are impossible.

A popular excuse ( the one used for the above code ) is, it works for _me_, _I_ never make these mistakes, why bother spending the effort. Removing impossible state means no-one else will ever make these mistakes either. This is why we have the confidence to let our newer devs go crazy on the codebase because _our state and types are locked down._ And once your code hits a level where one brain is no longer enough to store it, then this strategy becomes crucial in reducing regressions.
(Aside: Flattening an architecture is not something that I consider to be a core refactor. The reason being that it happens as a _consequence_ of decoupling your msg, model, making things opaque and removing impossible state. Blindly flattening everything to the same level under Main.elm does not result in a decouple architecture so those issues are still there and lead to responsibility breakdowns in overexposing types, sky rocketing compile times etc... but more on this in the [Compile Time](/compile-time.md).
