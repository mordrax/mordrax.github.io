# Extensible type hell

_Extensible types are great! (as long as you read the fine print). Step outside of their basic usage and the type errors start to look very strange. We use extensible types in Components, as domain model getters and in our Endpoints for encoding/decoding. I also discuss a darker side to their versatility when trying to convert between concrete and extensible types._

Our [Components](/chapters/components.md) make heavy use of extensible types, because by definition, they are only a component if they are shared between at least one other page/component/thing. So the types they work with has to be parameterised. eg:

```haskell
type alias Address a =
    { a
        | addressLine1 : String
        , countryCode : String
        , postcode : Int
        , state : String
    }
```

We have a certain model helper (Oracle.elm) that works off common fields and types. Since the backend is heavily constructed via interfaces, our endpoints typically contains many overlapping fields. Oracle helps reduce getters:

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

Oracle is great, it allows us to fetch complex relationships in our data models by specifying the minimal fields required and also allows us to mix and match these type aliases. It also means that the functions are not tied to any particular record and we have _alot_ of records that repeat fields.

Practically though, we don't actually use this as much as the creator would have wished because the more complex getters typically come with domain logic which looks into our Model folder for logic.

Lastly, I touch on a type inference issue.