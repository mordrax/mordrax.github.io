# Tools

## Alfred

## Oracle

We have a model helper (named Oracle.elm) that works off common fields and types. Since the backend is mostly constructed via interfaces, our endpoints contains many overlapping fields. Oracle helps to consolidate getter logic in one place:

```haskell
-- Oracle.elm
type alias HasSurnameFirstName a =
    { a | surname : String, firstName : String }

type alias HasId a =
    { a | id : Maybe Int }

{-| Returns: Jones, Bob ( 123 )
-}
surnameFirstnameId : HasSurnameFirstName (HasId a) -> String
surnameFirstnameId { surname, firstName, id } =
    surname ++ ", " ++ firstName ++ " ( " ++ (id |> Maybe.map toString |> Maybe.withDefault "") ++ " ) "
```

Oracle is great, it allows us to fetch complex relationships in our data models by specifying the minimal fields required and also allows us to mix and match these type aliases. It also means that the functions are not tied to any particular record and we have _a lot_ of records that repeat fields.

Practically though, we don't actually use this as much as the creator would have wished because the more complex getters typically come with domain logic which looks into our Model folder for logic.


## Component Wizardry

I just wanted to show off our 'wizard' component and demonstrate another kind of component.
```
type alias Wizard oz msg =
    { steps : ZipList (Step oz msg)
    }

type alias Step state msg =
    { validStep : state -> Bool
    , init : state -> ( state, Job msg )
    , view : state -> Html msg
    , validate : state -> List String
    }

next : state -> Wizard state msg -> ( Wizard state msg, state, Job msg )
back : state -> Wizard state msg -> ( Wizard state msg, state, Job msg )

init : List (Step state msg) -> Wizard state msg
view : state -> Wizard state msg -> Html msg
makeStep : (state -> Html msg) -> (state -> List String) -> Step state msg
```

Used like this:

```haskell
Wizard.init
    [ Wizard.makeStep (viewBase lookup) (always [])
    , Wizard.makeStep (viewNewDependant lookup) validateNewDependant
        |> Wizard.withCondition (...)
    , Wizard.makeStep (viewMemberTransfer lookup) validateMemberTransfer
        |> Wizard.withInit ...a Job
        |> Wizard.withCondition (\state -> ... True/False)
    ]

-- on next
case Wizard.validate state wizard of
    [] ->
        let
            ( nextWizard, nextState, job ) =
                Wizard.next state wizard
        in

-- on view
Wizard.view state wizard
```

So our wizard component holds a bunch of functions. It doesn't hold the steps, this is passed in. Next happens once a step is validated. Steps are conditional, they can contain initial jobs (Cmds + extras). Having a component like this is a massive time saver for multi-stepped forms which we seem to have more of now that there's a wizard.

## Grid