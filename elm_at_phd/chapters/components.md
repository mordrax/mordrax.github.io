# Components
_I present the various types of components we use, discuss their purpose and provide examples. UI Components are arguably the simplest form and everyone should have them. Stateful/Stateless and Hybrid components are patterns we defined along the way as we refactored our codebase to keep domain logic together._

I use the term _components_ fairly loosely here. By a component, I mean a function or a module which does no more or less to achieve one behaviour and acts _independently_ of the calling code. Some will have `msg` others will have `Msg`, others may have state, some will just use extensible records `{ a | ... }`.

#### UI components

We start with the simplest form of components and it looks like this:
```haskell
input : String -> (String -> msg) -> Html msg
readonlyTextInput : String -> Html msg

dateInput : DateTime -> (DateTime -> msg) -> Html msg
maybeDateInput : Maybe DateTime -> (Maybe DateTime -> msg) -> Html msg

ageInput : Int -> (Int -> msg) -> Html msg
emailInput : String -> (String -> msg) -> Html msg
```

These are great, you give it the current state, a msg to call and it will give you a UI control.
We also do dropdowns, which are all typed. They are either an enum or a dynamic list of types. For the latter they are a record, not strictly typed.

```haskell
genderDropdown : Lookup -> Int -> (Int -> msg) -> Html msg
genderDropdown lookup =
    Dropdown.enumDropdown (Lookup.genders lookup)   -- gender types are fixed (Enum)

bsbDropdown : Lookup -> String -> (String -> msg) -> Html msg
bsbDropdown lookup =
    Dropdown.dropdown (Lookup.bsb lookup)           -- bsb is variable (Lookup table)
```

This means our forms can use fields like this:
```
View.Layout.paneField "Gender"
    (View.Components.genderDropdown lookup model.gender Gender)
```

It took a bit of time to evolve our UI components to the function signatures above. I kept removing things until they had the minimal arguments required to make a function and when I couldn't take any more out, then I knew I was done.

Due to how our data types are in the backend, there are `Maybe` variants of these components to make handling maybe types easier.

#### Stateless components

A stateless component does not hold state. If you think that statement was obvious, then we have successfully named them. They look like this:

```haskell
type alias Address a =
    { a
        | addressLine1 : String
        , countryCode : String
        , postcode : Int
        , state : String
    }

update : Lookup -> Msg -> Address a -> Address a
view : Lookup -> Bool -> Address a -> Html Msg
validate : Address a -> List String
```

Couple of things to note here. To make it possible to interact with the rest of the world with no state, the component defines an extensible record and interacts with that. The component is like a service, it contains _domain logic_ to handle edits for these fields. It knows how to display them, update the fields and we also include all validations in the component.

This way our various pages or even other components could pick this up and say, hey! please handle _my_ address state for me. And do the validation for it.

Sure enough our AddressesEditor is one such happy customer:

```haskell
type alias AddressesEditor a =
    { defaultAddress : Address a
    , addresses : List (Address a)
    , addressType : Enums.AddressType
    }
```

It's concerned with different types of addresses like a home, work, delivery address and happily uses the AddressEditor.elm component to handle updating single addresses.

AddressesEditor is a [Stateful Component](#stateful-components), read on!

#### Stateful components

Stateful components hold state. If that was obvious... see [Stateless components](#stateless-components).

An Address**es**Editor (not to be confused with an AddressEditor) is one such component, it looks like this (Note, this type of component has fallen out of favor for the reasons outlined at the end of this code block):

```haskell
type alias AddressesEditor a =
    { defaultAddress : Address a
    , addresses : List (Address a)
    , addressType : Int
    , viewMode : ViewMode
    }

init : Address a -> AddressesEditor a
update : Lookup -> Msg -> AddressesEditor a -> ( AddressesEditor a, Job Msg )
view : Lookup -> msg -> (Msg -> msg) -> AddressesEditor (AddressWithBarcode a) -> Html msg
validate : AddressesEditor a -> List String

getAddresses : AddressesEditor a -> List (Address a)
getAddressType : AddressesEditor a -> Int
getAddress : Int -> AddressesEditor a -> Maybe (Address a)
```

So here the component holds addresses as well as the type of address that's currently being edited and various ways to display the component. Its view function is already looking a bit cluttered and there are a bunch of getters. These actually expose functions to return state to the parent _but you have to remember to do this_. And if the rest of the world has moved on in the meantime (ie the address in the parent changed), the editor would be holding stale data and worse, saving stale data.

The core issue here is that the component is _holding model state that belongs to the parent_. So we no longer use this style of component.

Our current way of doing stateful components is for the component to hold only UI state and pass parent state in as the stateless components do, ie, our Grid.

```haskell
type alias StatefulGrid subject =
    Grid subject (Msg subject)

type alias Grid subject msg =
    { columns : List (Column subject msg)
    , selected : subject -> Bool
    , rowClicked : Maybe (subject -> msg)
    , pageSizeChanged : Maybe (Int -> msg)
    , pageClicked : Maybe (Int -> msg)
    , pageSize : Int
    , pagePosition : Int
    , comparer : subject -> subject -> Bool
    , selection : List subject
    , multiSelect : Bool
    , sorter : subject -> subject -> Order
    , sortGrid : Bool
    , stateful : Bool
    , rowAttributes : subject -> List (Html.Attribute msg)
    }

initPaging : (subject -> subject -> Bool) -> StatefulGrid subject
update : Msg subject -> StatefulGrid subject -> StatefulGrid subject
addCol : String -> (subject -> comparable) -> Grid subject msg -> Grid subject msg
render : List subject -> Grid subject msg -> Layout.Block msg
```

Let's not look too deep into the `type alias Grid`, it's a great example of how **not** to write bug free, minimal state type aliases. The point here is that `Grid` only holds _UI state_. It sorts, it selects, it pages and these are reflected in its UI state. When it comes time to call the view function, we pass the data into it.

Usage looks like this:
```haskell
Grid.empty
    |> Grid.addCol "Component" .description
    |> Grid.addCol "From Date" (.fromDate >> Alfred.Dates.toDate)
    |> Grid.render waitingPeriods
```

#### Hybrid components

So our stateless components define the data that they interact with in terms of a extensible record. Our stateful components only hold UI state. Naturally then, if there is a component that both requires UI state and operates on the domain model we combine the two types of components above:

```haskell
type alias ReceiptMethod a =
    { a
        | receiptMethodTypeId : String
        , methodReference : String
        , methodReferenceName : String
        , methodDate : DateTime
    }

type alias ReceiptMethodEditor =
    { newRecipientToggle : NewRecipientToggle
    }

update : Lookup -> Msg -> ReceiptMethod a -> ReceiptMethodEditor -> ( ReceiptMethod a, ReceiptMethodEditor, Job Msg )
view : Lookup -> ReceiptMethod a -> ReceiptMethodEditor -> Html Msg
```

`ReceiptMethod a` is a extensible record and operates on the `domain model`. `ReceiptMethodEditor` holds _UI state_ that no parents care about.
The `update` function explicitly takes in both separately and returns both. This _forces_ the developer to use its parent's domain model thus not duplicating the state and removes the risk of forgetting to update it in the parent as was the problem with `AddressesEditor`.

