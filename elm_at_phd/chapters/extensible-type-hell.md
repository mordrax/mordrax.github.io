# Extensible type hell

_Extensible types are a great way to make reusable components. However, step outside of their basic usage and the type errors start to look very strange. We use extensible types in [Components](/chapters/components), as [domain model getters](/chapters/tools.md#oracle) and in our [Endpoints](/chapters/endpoints.md) for encoding/decoding. However, we have encountered, thrice now, a perplexing error when converting from concrete to extensible types._

This section exists because I spent two days scratching my head and later on observed both my work colleagues (concurrent financial systems expert and haskell guy) doing the same thing. The problem arises from a compiler error such as this:

```haskell
TYPE MISMATCH
Line 11, Column 13
The definition of container does not match its type annotation.

The type annotation for container says it is a:

Main.PointContainer Main.ThreeDPoint
But the definition (shown above) is a:

Main.PointContainer { z : Int }
Hint: Looks like a record is missing these fields: x and y. Potential typos include:

x -> z
y -> z
```

So the compiler says: "The type annotation say it's the following"

```haskell
Main.PointContainer Main.ThreeDPoint
```

But you are giving it this instead:
```haskell
Main.PointContainer { z : Int }
```

Let's look at the relevant code (full demo[^1] hosted by ellie-app[^2]):

```haskell
module Main exposing (main)

import Html exposing (Html, text)


main : Html msg
main =
    let
        container : PointContainer ThreeDPoint
        container =
            make3D 1 2 3
                |> makeContainer
    in
    container
        |> (\{ point } -> printAPoint point)

type alias PrintablePoint a =
    { a | x : Int, y : Int }


type alias ThreeDPoint =
    { x : Int, z : Int, y : Int }


type alias PointContainer xyPoint =
    { point : PrintablePoint xyPoint
    }


makeContainer : PrintablePoint a -> PointContainer a
makeContainer printablePoint =
    { point = printablePoint }


make3D : Int -> Int -> Int -> ThreeDPoint
make3D =
    ThreeDPoint
```

So going back to our error, let's work out what is going on:

```haskell
-- the error is in the 'container' function.
container : PointContainer ThreeDPoint
container =
    make3D 1 2 3          -- makes a ThreeDPoint = { x : Int, z : Int, y : Int }
        |> makeContainer  -- takes in PrintablePoint a = { a | x : Int, y : Int }
                          -- returns PointContainer = { point : PrintablePoint a }
```

What's the problem???

Turns out, it isn't _really_ a problem:

```haskell
container : PointContainer ThreeDPoint
container =
    make3D 1 2 3
        |> toPrintablePoint
        |> makeContainer

toPrintablePoint : ThreeDPoint -> PrintablePoint ThreeDPoint
toPrintablePoint =
    identity
```

Here, all we're doing is matching the types, make3D returns a `ThreeDPoint` but makeContainer accepts a `PrintablePoint ThreeDPoint` so we have to give it that type. However, since they are the same concrete record, we use use the `identity` function here... confused? So am I.

I played around with a few other things:
```haskell

-- just add identity, Fail.
container : PointContainer ThreeDPoint
container =
    make3D 1 2 3
        |> identity
        |> makeContainer

-- add toPrintablePoint without a type annotation, Fail.
container : PointContainer ThreeDPoint
container =
    make3D 1 2 3
        |> toPrintablePoint
        |> makeContainer

toPrintablePoint =
    identity
```

The issue is definitely that the compiler is not able to resolve the two types. The full working demo[^3] demonstrates how adding a identity function solves the error.

In time, others smarter than I will be able to explain to me why this behaviour occurs as such, however, for the moment, if you are hitting type issues which you are _sure_ should work, try explicitly converting the value to the type that's required even if the underlying record is the same.


[^1]: Full broken version: https://ellie-app.com/f726SMFcpa1/0
[^2]: By Luke Westby, a full featured elm compiler in your browser - https://ellie-app.com
[^3]: Full working example : https://ellie-app.com/m9f8zgZZSa1/1