# Forms

_You have a component that shares behaviour ( open, close, save ). You try to make it generic. It does not work. You try another way. It does not work. You try again. It does not work. You think about it for weeks, months. It does not help. You fail and write a 10,000 word essay about how you failed to make generic forms._

Firstly, let's define what I mean by a form. It's a TEA module with some common characteristics. All forms have validation, the ability to save, only one can be open at any given point in time and you can open and close them.

```haskell
-- src/Membership/Forms.elm
type FormState
= NoForm
| AddressState Address
| ... x30 forms
-- src/Membership/Forms/Address.elm

type alias Address = ...
type Msg = ...
init : Address
update: Msg -> Address -> (Address, Job Msg)
view: Address -> Html msg
save: Address -> Result (List String) (Job Msg)
validate: Address -> List String
```

Also their layout is very similar, e.g the same header bar with save/cancel buttons. This is driven by the fact we want the user to recognise a form when they see one.

This is a prime case for making a generic form module to handle this. _Surely_.

I've rewritten the mechanism around opening, closing, saving and viewing of a form three times and present them below.

#### Approach 0: Forms as pages, share view

The simplest solution is to wire up each form like a TEA module. The parent will `init` to open and the form will respond with a tuple to close itself. Since their layout is similar, we used a common view function.
```
-- src/Membership/Forms/Address.elm
update: Msg -> Address -> ( Address, Bool, Job Msg)
update msg address =
case msg of
Close -> ( address, True, Job.init )

view : Address -> Html Msg
view model =
let
buttons =
[ View.Component.button "Save" Save ]
in
View.Form buttons (render model)
```

But this means that each form will have to implement their own validate, save and response which are also common.

```haskell
update: Msg -> Address -> ( Address, Bool, Job Msg)
update msg address =
case msg of
Save ->
case validate address of
[] ->
(address, False, addressApi address |> Job.fromTask SaveResponse)
errs ->
({ address | validationErrors = errs }, False, Job.init)

SaveResponse response ->
-- common code for all forms
```

So what are the good and bads of this approach.

Pros:
* The view uses a helper and provides a consistent layout
* Each form is independant and arguments, return value do not have to be homogenous

Cons:
* There appears to be alot of repeat code in the update and the view to handle saves, validation, closing etc...

So by this time I was itching for some component.

#### Approach 1: Forms manager module

The common theme around forms is that we were wiring the save, response and view up for every form, repetitive. So by having a top level manager that held the form state and the msgs, forms could just expose `save`, `validate` and `update` for the manager to call. It would also handle validation errors in the one place!

```haskell
-- Forms.elm
type alias Forms =
{ formState : FormState
, validationErrors : List String
}
type FormState
= NoForm
| AddressState Address
| ... x30 forms

view: Forms -> Html Msg
view { formState, validationErrors } =
let
headers =
[ View.Components.button "Save" Save
, View.Components.button "Close" Close
, viewErrors validationErrors
]
render body =
div [] [ headers, body ]
in
case formState of
AddressState address ->
render (Address.view address)
```

However, each msg that goes to the form now will have to have a corresponding `case ... of` to delegate it to the right form.
```
update : Msg -> Forms -> (Forms, Job Msg)
update msg {formState, =
case msg of
FormMsg formMsg->
updateForm formMsg formState
Save ->
saveForm formState

updateForm formMsg formState =
case (formMsg, formState) of
(AddressMsg msg, AddressState state) ->
Address.update msg address
... x30 forms

saveForm formState =
case formState of
AddressState state ->
Address.save state

... x30 forms
```

So let's analyse this approach.

Pros:
* in view, `render` provides a consistent layout
* handling of form close, validate and save have been moved to a central place so the form itself doesn't have to repeat that code
* since all validation is of the same type, we only need to refer to a single validationErrors state in the view (actually, this is a opportunity for bugs... so this is not a pro)

Cons:
* handling of validation in a common place can ( and has ) caused a bug where we forgot to clear it when closing a form. This would be trivially mitigated by composing the form state with the validation errors though ie `type alias Forms = { formState : ( FormState, List String ) }` or creating a `empty : Forms` function.
* it really hasn't generalised very much. It's moved the 'forms' handling out of each of the form which is great w.r.t decoupling the form mechanics to what it does but having each msg type being accompanied by a `case ... of` that spans all the forms is undesirable. Luckily, our forms component only has a handful of common msg types.

_Aside: This is where we are at now, at the time of writing this article. Do not do this... it is no better than whatever else you're doing, fairly certain._

#### Approach 2: Forms manager module with an interface

I don't know what these are, but coming from the OO world, I shall call them... interfaces
```
type alias IForm formMsg formState =
{ update : Repository -> formMsg -> formState -> ( formState, Job formMsg )
, save : Repository -> formState -> Result (List String) (Task Http.Error MemberPolicyDelta)
, view : Repository -> formState -> Html formMsg
, state : formState
}
```

So this is great, it allows me to do the following:

```haskell
-- define a IForm

type FormState
= NoForm
| AddressState (IForm Address.Msg Address)
| ... x30 IForms


-- initialise a form

MemberAddressForm addressType ->
AddressState
{ update = Address.update
, save = Address.save
, view = Address.view
, state = Address.init addressType membership.policy.addresses
}
|> returnFormState

-- then in the update
updateIForm : IFormMsg -> Forms -> (Forms, Job Msg)
updateIForm formMsg { formState } =
case formState of
in
case forms.formState of
AddressState form ->
applyMsg AddressState AddressMsg form
... x 30 forms

-- applyMsg gets a little bit hairy, let's walk through it
-- the key here is that we get the generic save, update, state from the
-- IForm
applyMsg toFormState toFormMsg { save, update, state } =
case msg of

-- then we can apply the form's msg and state to the form's update
-- that the form provides, meaning we don't need the massive case ... of
IFormFormMsg formMsg ->
update repository formMsg state

-- this is just my shorthand for mapping state and msg
-- back to the Forms.elm level of types
|> mapStateAndJob toFormState toFormMsg

-- similarly, we apply the form's state to the form's save
-- which means we skip on the big case ... of here as well
IFormSave ->
case save repository state of
Result.Ok httpRequest ->
( { forms | saveState = Saving }, httpRequest )

Result.Err validationErrors ->
{ forms | validationErrors = validationErrors }
|> (\forms_ -> (forms_, Job.init ))
```

It's ok if you didn't quite follow the whole (poorly presented) example above, the main idea here is that by making forms homogeneous and applying it to a single type, we no longer need a case ... of to handle each form behaviour.

_But_.

In our case, it actually wasn't worth it. Because we still need one to route the correct msg/state to the right form and we still needed one for the view... _because is it a union type_.
And we still needed one for the `init` of a form.

Pros:
* The single IForm collects all update msgs into one `case ... of`.
* Impossible to make a logic error in form handling for a new form.
* Forms manager does not grow with each new feature of a form
* View and layout is still separated from the form
* New form creation are very streamlined

Cons:
* More complex than the 'slap it on as you go' solutions
* Feels like we're reimplementing TEA ( which is what stopped me the first two times )
* _Makes all forms homogeneous_. So the `Repository` that is passed into each form now holds quite a bit of unnecessary state because the interface has to take in the same datatypes and thus the end result is the lowest common denominator of state. **This feels very bad** as it goes against the whole minimalist approach we've used throughout the rest of the project to reduce impossible state and decouple modules.


#### Approach 3: Forms as a component

So much in the ECS vein, instead of saying a address form _is_ a form ( approach 0 ) or that there is a form manager that _has_ all forms ( approaches 1 and 2 ), we say that a page is _composed of_ an address editor and a form.

Um actually, I haven't made this change yet ( as there isn't enough of a business case to do this atm ) so I'll let the avid reader try to link this section up with [Components](#components) and see what results.

#### Conclusions

So my feeling about how we've done this is as unsatisfactory as how I've left the reader on Approach 3. I hope the pros and cons have been useful in analysing the various approaches of making a re-usable component or at least that I've steered you away from the above component approaches because the more I discuss them, the more I dislike them.
