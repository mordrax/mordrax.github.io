## Cmd
Effects in Elm ( Cmd msg )

## ADTs
Algebraic Data Type

## ADT
Algebraic Data Type

## TEA
The Elm Architecture. In 0.17, it was common and recommended to nest modules and wire up their init/update/view to each parent. I call these TEA modules.

## Alfred
The _helpful_ butler who knows a bunch of useful FP things, so named to avoid namespace conflicts with other popular modules that uses Helper as a top level module name.

## Lookup
A repository of static domain data such as enums, insurance products, users etc... Lookup is useful for displaying string representations of ids.

## FUN!
A reference to Dwarf Fortress where 'losing is fun'. Much in a similar vein, extensible records can get very confusing very quickly.

# Code listing

## Alfred.Logic
```haskell
module Alfred.Logic exposing (..)

{-| Alfred does logic. Specifically, these should not be dependant on any types.
-}


{-| when the predicate is true, transform the data
-}
when : Bool -> (a -> a) -> a -> a
when predicate f x =
    if predicate then
        f x
    else
        x


unless : Bool -> (a -> a) -> a -> a
unless predicate =
    when (not predicate)


{-| Adds element to list if it's not nothing
-}
maybeAddToList : Maybe a -> List a -> List a
maybeAddToList couldBe list =
    case couldBe of
        Just is ->
            is :: list

        _ ->
            list


appendWhen : Bool -> a -> List a -> List a
appendWhen pred newItem list =
    if pred then
        list ++ [ newItem ]
    else
        list


appendMap : (a -> b) -> Maybe a -> List b -> List b
appendMap f maybeA list =
    case maybeA of
        Just value ->
            list ++ [ f value ]

        Nothing ->
            list


prependWhen : Bool -> a -> List a -> List a
prependWhen pred newItem list =
    if pred then
        newItem :: list
    else
        list


combinePredicates : List (a -> Bool) -> (a -> Bool)
combinePredicates allPreds =
    \a -> List.all (\pred -> pred a) allPreds
```

## Components.Wizard

```haskell
module Components.Wizard
    exposing
        ( Wizard
        , back
        , hasBackStep
        , hasNextStep
        , init
        , makeStep
        , next
        , position
        , validate
        , view
        , withCondition
        , withInit
        )

import Alfred
import Alfred.ZipList as ZipList exposing (ZipList)
import Html exposing (Html)
import Job exposing (Job)
import Maybe.Extra


type alias Wizard oz msg =
    { steps : ZipList (Step oz msg)
    }


type alias Step state msg =
    { validStep : state -> Bool
    , init : state -> ( state, Job msg )
    , view : state -> Html msg
    , validate : state -> List String
    }


{-| Goto the next step in the wizard.
-}
next : state -> Wizard state msg -> ( Wizard state msg, state, Job msg )
next state model =
    case nextStep state model of
        Nothing ->
            Alfred.say "Trying to move next however, no more valid steps." "" ( model, state, Job.init )

        Just wizardNextStep ->
            currentStep wizardNextStep
                |> Maybe.Extra.unwrap ( state, Job.init ) (\step -> step.init state)
                |> (\( state, job ) -> ( wizardNextStep, state, job ))


{-| Goto back a step in the wizard.
-}
back : state -> Wizard state msg -> ( Wizard state msg, state, Job msg )
back state model =
    case backStep state model of
        Nothing ->
            Alfred.say "Trying to move back however, no more valid steps." "" ( model, state, Job.init )

        Just wizardBackStep ->
            currentStep wizardBackStep
                |> Maybe.Extra.unwrap ( state, Job.init ) (\step -> step.init state)
                |> (\( state, job ) -> ( wizardBackStep, state, job ))


currentStep : Wizard state msg -> Maybe (Step state msg)
currentStep =
    .steps >> ZipList.current


hasNextStep : state -> Wizard state msg -> Bool
hasNextStep state model =
    Maybe.Extra.isJust <| nextStep state model


hasBackStep : state -> Wizard state msg -> Bool
hasBackStep state model =
    Maybe.Extra.isJust <| backStep state model


nextStep : state -> Wizard state msg -> Maybe (Wizard state msg)
nextStep state ({ steps } as model) =
    let
        zipListNextStep =
            ZipList.forward steps

        nextStepIsValid =
            zipListNextStep
                |> ZipList.current
                |> Maybe.Extra.unwrap False (\f -> f.validStep state)
    in
    case ( ZipList.position steps, nextStepIsValid ) of
        ( ZipList.End, _ ) ->
            Nothing

        ( _, True ) ->
            Just { model | steps = zipListNextStep }

        ( _, False ) ->
            nextStep state { model | steps = zipListNextStep }


backStep : state -> Wizard state msg -> Maybe (Wizard state msg)
backStep state ({ steps } as model) =
    let
        zipListBackStep =
            ZipList.backward steps

        backStepIsValid =
            zipListBackStep
                |> ZipList.current
                |> Maybe.Extra.unwrap False (\f -> f.validStep state)
    in
    case ( ZipList.position steps, backStepIsValid ) of
        ( ZipList.Start, _ ) ->
            Nothing

        ( _, True ) ->
            Just { model | steps = zipListBackStep }

        ( _, False ) ->
            backStep state { model | steps = zipListBackStep }


position : state -> Wizard state msg -> ZipList.Position
position state wizard =
    if backStep state wizard == Nothing then
        ZipList.Start
    else if nextStep state wizard == Nothing then
        ZipList.End
    else
        ZipList.Between


makeStep : (state -> Html msg) -> (state -> List String) -> Step state msg
makeStep =
    makeStep_ (always True) (\s -> ( s, Job.init ))


withInit : (state -> ( state, Job msg )) -> Step state msg -> Step state msg
withInit init step =
    { step | init = init }


withCondition : (state -> Bool) -> Step state msg -> Step state msg
withCondition pred step =
    { step | validStep = pred }


makeStep_ : (state -> Bool) -> (state -> ( state, Job msg )) -> (state -> Html msg) -> (state -> List String) -> Step state msg
makeStep_ =
    Step


init : List (Step state msg) -> Wizard state msg
init listOfSteps =
    { steps = ZipList.fromList listOfSteps }


view : state -> Wizard state msg -> Html msg
view model wizard =
    Maybe.Extra.unwrap (Html.text "Error: The Wizard has disappeared!") (\{ view } -> view model) (ZipList.current wizard.steps)


validate : state -> Wizard state msg -> List String
validate model wizard =
    Maybe.Extra.unwrap [] (\{ validate } -> validate model) (ZipList.current wizard.steps)
```