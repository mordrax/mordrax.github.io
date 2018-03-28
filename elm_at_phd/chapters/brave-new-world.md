# Brave New W..holy cow!

_The first week on the job was challenging for me. It was the first time working with Elm professionally and the code I saw there was, to put it lightly, extremely horribly bad. I describe major flaws in the design and explain why they are bad._

> You can choose to ignore ADTs, couple modules by their Msg or Model types, write imperatively, carelessly create impossible state, type everything as strings, name functions completely different to what they do, introduce Maybes everywhere and the compiler will not save you.

The first thing I did was to add elm-format[^1]. Elmformatslikeaddingpunctuationspacesandcommastoasentence.


#### Global Msg coupling

The root level Msg.elm appeared as an import in 74 files. In many cases, this was `import Msg exposing (Msg(..))`

Since modules nested many levels deep returned the root level Msg, each of these modules mapped their msg to the global `Msg` all the way down and ended up constructing the global `Msg` like these:

```haskell
fieldTag : Tagger PersonPanelMsg -> String -> (String -> Msg)
fieldTag tagger fieldName =
    (\val -> tagger <| OverlayPanelMsg_ <| EditPersonMsg_ <| SetPersonField ( fieldName, val ))


clickTag : Tagger PersonPanelMsg -> String -> String -> Msg
clickTag tagger fieldName val =
    tagger <| OverlayPanelMsg_ <| EditPersonMsg_ <| SetPersonField ( fieldName, val )
```

In any module where this happens, it coupled that module to the global `Msg` type. Any change in `Msg` could potentially affect all modules that import it and thus caused all these modules to recompile.

#### Magic numbers everywhere

In a language with ADTs, where the compiler is able to prove the correct usage of these ADTs, it makes _absolutely no sense_ to represent a finite set of options with an infinite set of Int or even worse, Strings.

```haskell
if val == "0" then ...

case panel.tabIndex of
    7 ->
    6 ->
    _ ->

case formType of
    "L..." ->
    "M..." ->
    _ ->
```

This is something you might do in javascript. In fact, you are forced to because there is no other way, Javascript (and other languages like C#, Java, Ruby) only have primitive types. The compiler is unable to check whether all paths through the program is accounted for. ADTs in Elm are a core ingredient to reducing regression bugs because it allows the compiler to check the application for correctness.


#### Msg, Model, Parent coupling


So on first inspection, the following function updates the overlay pane. All good, carry on.

```haskell
-- src/Elixir/Person/Update.elm
updateOverlayPane :
    PersonTagger ->
    PersonOverlayMsg ->
    Model ->
    ( WBPanel, UIPersonPanelModel ) ->
    ( WBPanel, Cmd Msg )
```

err.. hang on, what are the two models `Model` and `UIPersonPanelModel` doing. Shouldn't we take in one model and return the updated version?

Well, it turns out, actually, that there are _three_ models in this module.

```haskell
updateOverlayPane :
    PersonTagger ->
    PersonOverlayMsg ->
    Model -> -- Global model
    ( WBPanel, UIPersonPanelModel ) -> -- (Parent model, Own model)
    ( WBPanel, Cmd Msg ) -- Global msg
```

It turns out this function updates this module's model `UIPersonPanelModel`, then takes the parent model (`WBPanel`), updates that too, then constructs a global level `Msg` because it has complete visibility of that type through the `exposing (Msg(..))`.

By coupling these three models in the one module, it made it much harder for me to understand what this function does because I have to understand what the parent does, and what the root does as they are all related. If only Elm would enforce purity of scope as well as purity of functions. ( hint: Opaque types )

_Aside: Much later on, I looked back on this and realised this was possibly their way of trying get child to parent communication working. At the time, it was really... really... scary._


#### Maybe ids and the ticking time bomb

```haskell
-- this sort of code...

type alias Person =
    { personId : Maybe Int
    }

-- lead to these hacks

pid = Maybe.withDefault personId 0
personId = Maybe.withDefault uiModel.number 0
id = Maybe.withDefault myId 0
```

So what's happening is that the backend is returning a `Maybe Int` for a person id. Then the frontend handles it by using 0 as a default whenever it encounters Nothing _reasoning that it'll never happen_.

What this does is basically:

```haskell
try {
    // get person id
    // do something with it
} catch () {
    // swallow error, suckers...
}
```

(_Aside: Backend recently stopped doing this. Now I'll sleep well again._)

The point is, never use a Maybe for something that should never be null. Coming from js, _everything_ could be undefined or null... you poor sods. In a typed functional language, Maybes are rare because for most functions, you actually don't want it's arguments to be null. e.g: `add: Maybe Int -> Maybe Int -> Maybe Int`

#### Impossible states

Things like the following example are common.

```haskell
type alias Form = { ...
    , verified : Bool -- has the form been verified
    , message : String -- verification error message
    }

-- used like
if fm.verified then
    div [] [ text (fm.message) ]
```

So what happens when verified is true and there is an error message? Or if it's false and there is no message? Or what do those two fields even have in common?
Here's how I would do it.

```haskell
type VerificationStatus
    = NotVerified
    | Error String
    | Success

type alias Form = {
    verificationStatus : VerificationStatus
    }
```

The type documents the intent + represents _exactly_ the states we want and now the compiler will help us if any of those cases are not handled either now or next year.

And so at the end of the first day, I was left wondering how to even start working on this project. To put it bluntly, this might as well have been written in javascript. At least you have a `try...catch` wrapper there for exceptions, here, most cases had a `_ -> ...` to swallow up all exceptions.

[^1]: https://github.com/avh4/elm-format