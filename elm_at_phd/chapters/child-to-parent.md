# Child to Parent/Child/Sibling/First Cousin 2nd removed Communication

_Child to parent communication is a misnomer because how you communicate state is different to msgs and how you communicate to your immediate parent will be different to talking to an arbitrary child or root node. This section talks about some common forms we use and one custom solution we made. This has currently catered for all our needs well enough. I stress, we do not have a generic solution to this non-generic problem._

#### Tuples

So we saw in [Hybrid Components](#hybrid-components) that tuples are passed back.

update : Msg -> ReceiptMethod a -> ReceiptMethodEditor -> ( ReceiptMethod a, ReceiptMethodEditor, Job Msg )
This is the simplest form of child to parent. The parent gives the child (component in this case) it's model (`ReceiptMethod`) which the child takes as `ReceiptMethod a`. The child then updates the parts of the parent's model that it knows and returns it back to the parent. Since the child here is a stateful component, it also has it's own state `ReceiptMethodEditor` that must be updated.

We use this pattern in most forms.

update : Msg -> Contact a -> Contact a
update : Msg -> Address a -> ( Address a , Job Msg)
update : Msg -> Medicare a -> MedicareEditor -> ( Medicare a, MedicareEditor, Job Msg )


Since our pages are one level below main and the forms another level below pages, we only thread the tuple up a single level. It minimises the amount of msg/state passing.

#### (Msg -> parentMsg)

A module's `view : Model -> Html Msg` will only emit the module's msg and then you can `Html.map` it to the parent. But this means that the child view cannot communicate any other messages. The only case we have needed it for is routing.


view : (Route -> msg) -> (Msg -> msg) -> Model -> Html msg
view toRoute toParent model = ...


The `Route` type is therefore a global type which all modules know about and so the child can pass that up to the main level to action.

#### PubSub ports (aka wormholes)

I need to preface this section with context: Very early on, the application was in a real mess. Modules overstepped their scope by importing multiple parent level models and returning them to the parent using a global `Msg` type. There was some *really* convoluted child to parent communication going on. A one size fits all idea our team came up with was to use ports for all inter-module dependencies. However, through experience with Elm previously, I knew this was a overkill solution that would be more trouble than it was worth.


{-| Not safe, use publish
-}
pub : String -> Value -> Bool
pub topic value =
Native.PubSub.pub topic value
{-| Not safe, use subscribe
-}
port sub : (( String, Value ) -> msg) -> Sub msg

This version of ours uses Native code, which should be going away in 0.19 but you can do this using ports as well.

I was against this approach for two reasons:

1. The event becomes impure and untrackable. What happens when more than one arrive? What if none arrive? How can you be sure? How do you trace through a system once there is more than one emitter for the same Msg type? One of the great things about Elm is that it is explicit and deterministic. By introducing a global _external_ message bus, I felt that we would lose all of that certainty. You also lose the time travelling debugger as these Msgs are outside of the system.

2. The compiler loses track of whether these `Msgs` are handled. Once it goes into the port, there is no guarantee it is handled anywhere. If someone forgets to subscribe where it's required, bugs will invariably occur. This is a huge price to pay for a generic solution that is overkill for the problem at hand.

In the end, we didn't need it. I decoupled and rewrote the hairier bits, flattened out the architecture and most of the child -> parent went away.

#### Jobs

_Cmd is the most common effect mechanism in Elm, however, you cannot use it to pass Tasks, especially curried ones. So we created our own Cmd wrapper, called a Job and packed it full of useful things._

Here is its interface:

-- it acts like Cmd
init: Job msg
map : (a -> b) -> Job a -> Job b
batch : List (Job msg) -> Job msg
-- but allow curried tasks
addHttpTask : (payload -> msg) -> HostToTask payload -> Job msg -> Job msg
fromHttpTask : (payload -> msg) -> HostToTask payload -> Job msg
-- route switching
addRoute : Route -> Job msg -> Job msg
-- adhoc child -> root actions
addAction : Action -> Job msg -> Job msg
-- wraps other non-Job
addCmd : Cmd msg -> Job msg -> Job msg

Here are some example usages:

-- tell the main level to change route from a child view
GotoDependant dep ->
( model, Job.addRoute (Types.Route.RoutePerson personId Types.PersonSubPage.PersonDetails) Job.init )
-- tell the main level to queue up a task
SearchMembers policyId ->
( model, Job.fromHttpTask ReceiveResults (Api.findPersons request) )
-- get main to perform a custom action
( person
, personId
|> Types.TemplateRequestEncoder.certificate
|> flip Job.addAction Job.init
)
-- map all jobs to the same Msg
, Job.fromHttpTask ReceivePolicy (Api.getPolicy membershipId)
|> Job.add (Job.map ActivityMsg activityJob)
|> Job.add (Job.map QuotesMsg quotesJob)



So the main reason why we need to do this is a page ( child module ) will send off a HTTP request. However, the request requires information about the endpoint like JWT for auth, domain to hit etc so previously this information was passed down through every level from Main.elm to Child.elm as an extra state.

type alias HostToTask payload =
Host -> Task Http.Error payload
fromHttpTask : (payload -> msg) -> HostToTask payload -> Job msg
Job.fromHttpTask ReceiveResults (Api.findPersons request)
findPersons : PersonSearchRequest -> Host -> Task Http.Error (List PersonSearchResult)


Notice here that Api.findPersons is only given one argument, and thus returns `Host -> Task ...`. The job therefore holds a list of these curried functions waiting for the `Host` which is on the `Main.elm` level. `Job.map` takes care of mapping these msgs up the chain much like `Cmd.map`. `ReceiveResults` is what the `payload` is mapped to when it's successful, otherwise it's a hardcoded error which is handled on the top level.

Without this requirement, we'd not need Job, the other things can be turned into Cmds because they are resolved at the child level.
