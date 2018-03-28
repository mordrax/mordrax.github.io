# File Structure

_Brief look into how an app file structure with 45k LoC and 436 files evolved. Without a pre-meditated plan, our file structure grew as needed, the only exception was to make sure global, popular modules ( lots of imports ) got broken up to avoid compile time hell._

#### How we structure
```
|-- Main.elm -- obligatory. Fun Fact: Ours is 802 lines and counting (0).

|-- Login.elm -- simple subsystems are root level modules (1)
|-- Notification.elm -- another simple subsystems

|-- Alfred.elm -- OBSOLETE: all helper functions (3)
|-- Alfred -- helper folder, so named to prevent namespace clashing

|-- Components -- stateful components (2)

|-- Membership -- large subsystems group into a folder
| |-- Api.elm -- OBSOLETE: all http helper functions (3)
| |-- Forms.elm -- Explained in the Forms section (4)
| |-- Forms
|-- Membership.elm -- large subsystems generally have a top level TEA module to distribute msg

|-- Model -- aka Repository, business logic for get/set of data (5)

|-- Person -- another large subsystem
| |-- Forms
| |-- Forms.elm
|-- Person.elm

|-- Search -- YALS (YASD?)
|-- Search.elm

|-- Ports.elm -- all our ports in one place

|-- Types -- non domain specific, non module specific, custom types
| |-- Api -- all endpoint types
|-- Types.elm -- global Type, wait... I can explain (6)

|-- Validators -- common validation helpers
|-- View --
| |-- Components -- UI components
| |-- Fields -- domain specific UI components to make building forms easier
```

#### Why we structure it so

Some commentary on our file structuring.

0. (_Fun Fact: Ours is 802 lines and counting_) We do not care about file size. The last time I looked at a file size was... to quote the line above but no, the time before then was... never. When considering what goes into a file, the _only_ consideration is: Does this module fulfil one responsibility. When a module does more (_or less_) than this, coupling occurs. Every. Single. Time.

1. (_Simple subsystems are root level modules_) There is a bunch of files at root level, they are typically directly imported by Main or they are global like Types.Elm or Routes.elm

2. (_Stateful Components_) The elm-guide says don't use _reusable components_ but doesn't actually define what it is. In our experience, certain patterns of components have emerged. We discuss these in depth in [Components](/chapters/components.md). Hint: UI state and Domain state should not be grouped under the same terminology of 'state'.

3. (_OBSOLETE: all helper functions_) So we used to group helper functions in the one module as you do in _most_ other programming languages (disclaimer: I only know C#, Ruby, Java, Javascript does this). However, as the avid reader would know from having already read [Compile Time](/chapters/compile-time.md), this is actually a really bad idea in Elm because a helper module by definition is a toolbox for the rest of your app.

```haskell
touch Alfred.elm
elm-make ...
Compiling 105 modules (Have a nice day!)
```
Ah compiler, always so friendly.

Instead, we now have a folder called Alfred and try to break our helpers up into modules that are commonly used together.

4. (_Explained in the Forms section_) [Forms](/chapters/forms.md), what a mess.

5. (_aka Repository, business logic for get/set of data_) We don't use opaque types and it's already bitten me twice. As a team grows and introduces more junior/non-domain experts, I think it's definitely worth making domain models opaque. This folder contains all domain specific helpers (e.g Membership.getDependants will sort by age). Unfortunately, domain logic is also tied in page updates, in the page views and validators.

6. (_global Type, wait... I can explain_) So Type.elm contains Id tags. The kind that looks like this:
```
type alias PersonId
    = Int

type QuoteId
    = Person PersonId
    | Member MemberId
```
The compile time on this is tolerable because these IDs are defined by the backend and change rarely.

