---


---

<p><em>This is a story of a guy who left it all to chase Elm full time, the technical trials and tribulations he faced, the euphoric and … not so euphoric bits of writing a SPA in Elm. My scope is ambitious, use these italic summary headers to decide on the sections you want to read.</em></p>
<p><em>Total word count: ~12k</em><br>
<em>Reading time: 60 mins. Bear in mind code snippets will take longer to comprehend.</em></p>
<p>I write Elm at Pacific Health Dynamics, a small health insurance software vendor with big dreams. Since July 2017, I’ve been leading the frontend rewrite of their flagship product.<br>
The codebase was at 16k LoC when I started. Since then, I’ve rewritten the various subsystems at least once (I’m looking at you <em>generic form component</em>). Now we hover around 45k LoC with most of the common SPA structures stabilizing. We are ~1/3 of the way to completion. This is a typical massive, legacy, regulated system.</p>
<p>The driving motivation for this essay is to share my experiences so far and discuss the various challenges we faced and provide practical solutions that we took to over come these. I feel that there is not enough long form analysis from teams using Elm and that is something I’ve craved when deciding whether to take on Elm in a professional setting rather than just as a hobby<sup class="footnote-ref"><a href="#fn1" id="fnref1">1</a></sup>. Therefore, this article is aimed at those coming from a js framework and thinking about using Elm professionally or not wanting to use js at all, good for you.</p>
<h1 id="content">Content</h1>
<p>So, at it’s heart, this a story. And a story, always has a beginning (<a href="#prologue">Prologue</a>).</p>
<p>The next chapter (<a href="#brave-new-w..holy-shit">Brave new w…holy shit</a>) details my analysis of the existing codebase when I started.</p>
<p>From there, we go straight into strategies for <a href="#refactoring">Refactoring</a> large codebases. That was when I had alot of fun digging into <a href="#compile-time">Compile times</a> which (prodded by Noah) produced a talk at a Elm remote meetup<sup class="footnote-ref"><a href="#fn2" id="fnref2">2</a></sup> with slides<sup class="footnote-ref"><a href="#fn3" id="fnref3">3</a></sup>.</p>
<p>With a much more structured foundation, I started developing patterns for <a href="#child-to-parentchildsiblingfirst-cousin-2nd-removed-communication">Child to parent</a> communication, our various types of <a href="#components">Components</a> matured, and I tackled generic <a href="#forms">Forms</a> over and over, which I’m still unhappy with.</p>
<p>A core piece of our frontend is how it interacts with the backend through our <a href="#endpoints">Endpoints</a> which consists solely of auto-generated decoders. Playing with extensible types can lead to <a href="#extensible-type-hell"><abbr title="A reference to Dwarf Fortress where 'losing is fun'. Much in a similar vein, extensible records can get very confusing very quickly.">FUN</abbr></a>!</p>
<p>I also go over our <a href="#file-structure">File Structure</a>, which mostly grew organically, with the main restriction given to how it affects module coupling and in turn, compile time.</p>
<p>Finally, I <a href="#Final-words">rant and rave</a> about how much I love Elm and try my best to word it like it’s a objective observation of the strengths and weaknesses of the language.</p>
<h1 id="prologue">Prologue</h1>
<p><em>How far will you go to follow your intuition? Leaving a place of engineering excellence, friends, culture for something… else.</em></p>
<p>So I worked at a great place. The organisation had a great mission (we built educational games for kids), their engineering team was focused on quality, the management was actually supportive and aligned business and tech needs well. In my 13 years coding professionally, I’ve seen my fair share of cheap, soulless ‘IT’ sweatshops, the corporate monoliths buried in red tape and politics, the struggling startups where you cut so much you’re cutting bone and still feeling burned out but where I was, was good. People cared about their work and were more or less competent. I was comfortable, doing ruby, coffeescript, ember during the day and hacking on Elm in the evenings.<br>
Up to this point, my only experience in Elm had been remaking a old game<sup class="footnote-ref"><a href="#fn1" id="fnref1:1">1</a></sup> for the better part of 18 months, learning Elm and game making.</p>
<p>The phone call was out of the blue, from a manager I’d known in a previous role. He was honest about the situation and what he described failed every single workplace indicator of mine. Every indicator bar one: <em>Autonomy with language of choice</em>. Opportunities like this are very rare. I had just been thinking about what Richard recently said, something along the lines of “I work with Elm 8 hours a day” and that’s what I wanted to do (because at my age, one realises how little time we actually have to make a difference). So I did it.</p>
<p>The application is a rewrite of a decade old legacy desktop app. Tech stack consisted of Elm and polymer in the frontend and Elixir, .Net WebApi with MSSQL in the back, as is common with corporate entities (wait what Elixir? More on that later). The job vacancy was made possible by their lead (and sole) developer abandoning the project halfway, leaving the company scrambling for resources (one of my workplace indicators). The application was already at 16k LoC and I was coming in as the ‘Elm expert’, meaning they couldn’t find anyone else with any experience in this strange and exotic language.</p>
<h1 id="brave-new-w..holy-shit">Brave New W…holy shit!</h1>
<p><em>You can choose to ignore ADTs, couple modules by their Msg or Model types, write imperatively, carelessly create impossible state, type everything as strings, call functions that do something completely different, introduce Maybes everywhere and the compiler will not save you.</em></p>
<p>The first thing I did was enforce elm-format<sup class="footnote-ref"><a href="#fn4" id="fnref4">4</a></sup>. Elmformatslikeaddingpunctuationspacesandcommastoasentence.</p>
<p>Then I started looking into the code. The following sections are the major issues that I remember from that time.</p>
<h4 id="global-msg-coupling">Global Msg coupling</h4>
<p>The root level Msg.elm appeared as a import in 74 files. In many cases, this was <code>import Msg exposing (Msg(..))</code></p>
<p>And since modules nested many levels deep returned the root level Msg, each of the updates took ‘taggers’ all the way down and ended up constructing Msg like these:</p>
<pre><code>fieldTag : Tagger PersonPanelMsg -&gt; String -&gt; (String -&gt; Msg)
fieldTag tagger fieldName =
    (\val -&gt; tagger &lt;| OverlayPanelMsg_ &lt;| EditPersonMsg_ &lt;| SetPersonField ( fieldName, val ))


clickTag : Tagger PersonPanelMsg -&gt; String -&gt; String -&gt; Msg
clickTag tagger fieldName val =
    tagger &lt;| OverlayPanelMsg_ &lt;| EditPersonMsg_ &lt;| SetPersonField ( fieldName, val )
</code></pre>
<p>These msgs came from many different levels in multiple files and constructed the root level Msg. This essentially couples all the modules to the Msg type. Any change in Msg could potentially affect all modules that import it.</p>
<h4 id="magic-numbers-everywhere">Magic numbers everywhere</h4>
<p>In a language with ADTs, it makes <em>absolutely no sense</em> to represent a finite set of options with a infinite set of Int or even worse, strings as representations.</p>
<pre><code>if val == "0" then ...

case panel.tabIndex of
    7 -&gt;
    6 -&gt;
    _ -&gt;

case formType of
    "L..." -&gt;
    "M..." -&gt;
    _ -&gt;
</code></pre>
<p>Let me repeat this. <em>When you have a finite set of options, use a union type.</em></p>
<p>This is something you might do in javascript. In fact, you are forced to because there is no other way, javascript only has primitive types and C# and Java etc… so we go through life thinking of enums as Ints. They are not and it makes a huge difference.</p>
<h4 id="msg-model-parent-coupling">Msg, Model, Parent coupling</h4>
<p><em>Aside: Much later on, I looked back on this and realised this was possibly their way of trying get child -&gt; parent communication working. At the time, it was really… really… scary.</em></p>
<p>So on first inspection, the function updates the overlay pane. All good, carry on.</p>
<pre><code>-- src/Elixir/Person/Update.elm
updateOverlayPane : 
	PersonTagger -&gt; 
	PersonOverlayMsg -&gt; 
	Model -&gt; 
	( WBPanel, UIPersonPanelModel ) -&gt; 
	( WBPanel, Cmd Msg )
</code></pre>
<p>err… hang on, what’s the two models <code>Model</code> and <code>UIPersonPanelModel</code> doing. Shouldn’t we take in one model and return the updated version?</p>
<p>Well, it turns out, actually, that there are <em>three</em> models.</p>
<pre><code>updateOverlayPane : 
	PersonTagger -&gt;              
	PersonOverlayMsg -&gt; 
	Model -&gt;                            -- Global model
	( WBPanel, UIPersonPanelModel ) -&gt;  -- (Parent model, Own model)
	( WBPanel, Cmd Msg )                -- Global msg
</code></pre>
<p>It turns out this function updates it’s own model <code>UIPersonPanelModel</code>, then take’s the parent model (<code>WBPanel</code>), updates that too, then constructs a global level <code>Msg</code> because it has complete visibility of that type through the <code>exposing (Msg(..))</code>.</p>
<p>Coupling is a topic that I’ll touch on through much of this article. This is probably the worst example that I’ve seen in any Elm codebase however it’s scarily easy to get yourself into this position.</p>
<h4 id="maybe-ids-and-the-ticking-time-bomb">Maybe ids and the ticking time bomb</h4>
<pre><code>-- this sort of code...

type alias Person =
    { personId : Maybe Int
    }
    

-- lead to these hacks

pid = Maybe.withDefault personId 0
personId = Maybe.withDefault uiModel.number 0
id = Maybe.withDefault myId 0
</code></pre>
<p>So what’s happening is that the backend is returning a <code>Maybe Int</code> for a person id. Then the frontend handles it by using 0 as a default whenever it encounters Nothing <em>reasoning that it’ll never happen</em>.</p>
<p>What this does is basically:</p>
<pre><code>try {
    // get person id
    // do something with it
} catch () {
    // swallow error, suckers...
}
</code></pre>
<p>(<em>Aside: Backend recently stopped doing this. Now I’ll sleep well again.</em>)</p>
<p>The point is, never use a Maybe for something that should never be a Maybe type. Coming from js, <em>everything</em> is a maybe type… you poor sods. In a typed functional language, we use Maybes sparingly and have certainties about the rest of the codebase.</p>
<h4 id="impossible-states">Impossible states</h4>
<p>Things like the following example are common.</p>
<pre><code>type alias Form = { ...
, verified : Bool    -- has the form been verified
, message : String   -- verification error message
}

-- used like
if fm.verified then
    div [] [ text (fm.message) ]
</code></pre>
<p>So what happens when verified is true and there is a error message? Or if it’s false and there is no message? Or what does those two fields even have in common?<br>
Here’s how I would do it.</p>
<pre><code>type VerificationStatus
	= NotVerified 
	| Error String 
	| Success
	
type alias Form = { 
  verificationStatus : VerificationStatus
}
</code></pre>
<p>The type documents the intent + represents <em>exactly</em> the states we want and now the compiler will help us if any of those cases are not handled either now or next year.</p>
<p>And so at the end of the first day, I was left wondering how to even start working on this project. To put it bluntly, this might as well have been written in javascript. At least you have a <code>try...catch</code> wrapper there for exceptions, here, most cases had a <code>_ -&gt; ...</code> to swallow up all exceptions.</p>
<h1 id="refactoring">Refactoring</h1>
<p><em>Refactoring typically leads to better code. The effect is cumulative. The opposite is also true, a fear of introducing regressions leads to a cumulative growth of technical debt. To refactor a large codebase: start by tightening primitive types into ADTs, then make Msgs opaque (models too if you’re not confident enough to keep it single responsibility), take a minimalist approach to function args then what modules exposes and FINALLY remove impossible state because now the compiler will have your back on these large state changes.</em></p>
<p>In my last two years of using Elm, the strongest attraction has been the ability to change logic and core components with confidence at any stage of a product’s lifecycle. Time and again we accrue technical debt from fear of breaking things, “It’s not worth the risk”. Eventually, these areas of the code becomes a liability, but it’s also too risky to modify. Elm’s compiler gave me confidence that no amount of unit or integration test coverage<sup class="footnote-ref"><a href="#fn5" id="fnref5">5</a></sup> ever did. So improve the architecture often and fearlessly. This momentum to create and confidence in my code is the drug that keeps me hooked.</p>
<p>For larger codebases, the order of refactoring is important because type changes cascade and primitive types make regressions much more likely. The aim of these refactors is to reduce the scope of changes and tighten the types so the compiler is able to help us.<br>
Hence we start with converting primitives to ADTs.</p>
<h4 id="primitive-types-to-adts">Primitive types to ADTs</h4>
<p>Replace all Int or String types which represent a finite group with an <abbr title="Algebraic Data Type">ADT</abbr>.<br>
For us, this was all enum types and adhoc state transistions ( success, failed, haven’t tried ).</p>
<pre><code>case formType of
    "Limits" -&gt;
    "Benefits" -&gt;
    _ -&gt; empty    -- (NEVER do this. What happens when you add a new value to the type?)
</code></pre>
<p>Replace things like the above with the following:</p>
<pre><code>type FormType = 
    Limits | Benefits
    
case formType of
    Limits -&gt;
    Benefits -&gt;
</code></pre>
<p>Consider using tagged types instead of primitives to represent IDs. In our app, we have about 50 types of ids, most of them Ints, some Strings, others… <code>Maybe Float</code>, we won’t talk about those.</p>
<pre><code>type alias Person = {
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
</code></pre>
<p>This is the best ‘bang for buck’ refactor <em>and</em> the first in order of what to refactor. When you spot it, change it! It affects only the specific argument or field that uses it but by using ADTs, the compiler starts to understand your domain. Without this, refactoring in Elm will feel as unsafe as any other language. <strong>Use ADTs</strong></p>
<h4 id="opaque-msg-and-models--to-a-lesser-extent-">Opaque Msg and Models ( to a lesser extent )</h4>
<p>For modules with a <code>type Msg</code> definition (your typical <code>init, update, view?</code> module), make Msg opaque.</p>
<p><em>No module should be able to see another module’s internal Msgs.</em></p>
<p>This is a sign that the two modules have overlapping responsibility and the modelling isn’t quite right. The exception we use is for common Msg types which is used in places such as <a href="#routing">Routing</a> and <a href="#child-to-parentchildsiblingfirst-cousin-2nd-removed-communication">Child to Parent Communication</a></p>
<p><em>Modules should not return a parent model</em></p>
<p>I feel like I shouldn’t have to say this, but I’ve seen it at my workplace and the consequences of this is catastrophic in the coupling that it creates. Use a parent model’s data but only ever update your own model.</p>
<p>If you’re seeing code like:<br>
<code>model.login.name.first ++ model.login.name.last</code><br>
or you’re trying to perform nested model updates:<br>
<code>{ model | login = { login | name = newName } }</code><br>
then, making models opaque may help here.</p>
<p>The idea is that each module should have responsibility of it’s own logic. ie, login should be responsible for giving back the first and last name in a <code>fullName: Login -&gt; String</code> function rather that the parent reaching in to manage this. Using opaque models <em>forces</em> the coder to go down the correct path in this case at the cost of extra destructuring.</p>
<p>If one module is importing another module’s Msg, then <strong>always decouple from the child node</strong>. This is because the child has to make the more generic Msg to pass up the chain. If you start refactoring from the parent, you’ll end up having a domino effect that forces updating all Msg types in one go. Theoretically it should work out, it just takes a long time and sometimes you’ll encounter some known compiler bugs so best to keep the scope small between successful compiles.</p>
<h4 id="be-a-minimalist-when-exposing-functions-and-the-arguments-of-those-functions">Be a minimalist when exposing functions and the arguments of those functions</h4>
<p>So, only type defining modules or helper modules should use <code>module xyz exposing (..)</code>, all others should expose the absolute minimal. Do you know how hard it is for a new guy to join a company with a sizeable existing codebase and see <strong>Every. Single. Module. Expose. Everything</strong>? Why even have modules?</p>
<pre><code>-- compare
module Login exposing (..)
module RouteTypes exposing (..)
module Logic exposing (..)

-- to
module Login exposing ( init, update, view, Msg, Login )
module RouteTypes exposing (Routes(..))
module Logic exposing (when, filterBy, ifThen)
</code></pre>
<p>These modules tell a story just by looking at their exposing. It doesn’t matter that Login has a function called <code>authenticateUser</code>, or that RouteTypes define more type aliases, when it’s not part of exposing, it’s <em>clearly</em> not designed to be used externally.</p>
<p>Make functions single responsibility and simple. It is not cool to have all views take in the top level Model. Simplicity in this context does not mean basic or small, it means it takes exactly what it needs to perform exactly what the name implies.</p>
<pre><code>-- simple validator in Login.elm
validate: Login -&gt; List String

-- still a simple validator in AddressEditor.elm because it takes in exactly what it needs to perform the validation
validate: Lookup -&gt; Address a -&gt; AddressEditor -&gt; List String

-- not simple
validate: ParentModel -&gt; List String

-- even simpler validator in Login.elm which is now usable in other forms, more on this in Components
validate: UsernamePassword a -&gt; List String
</code></pre>
<p>The reasons why you’d want to keep your function arguments as simple as possible is because:</p>
<ol>
<li>You can then pull these functions out to be composed with other data types.</li>
<li>Bugs will be much easier to narrow down since all functions only take in what they need and so typically only a handful of functions deal with any given area of your app.</li>
</ol>
<h4 id="model-the-minimal-set-of-state-needed-remove-impossible-state">Model the minimal set of state needed (remove impossible state)</h4>
<p>The first time I heard the term ‘impossible state’ was from Richard Feldman’s talk<sup class="footnote-ref"><a href="#fn6" id="fnref6">6</a></sup>. This is about the ability to model your business/app state exactly as it is, no more, no less. I do this last because state changes tend to have a domino effect and logic changes are the only way regressions can be introduced into the codebase. With the aforementioned refactoring steps (ADTs, opaque Msg, modules decoupling), the compiler is able to help minimise errors alot better.</p>
<p>Here’s an example of finding and removing impossible states.<br>
We have a Grid component, with the ability to show more columns via .a ‘Show More’ toggle button. This feature can also be disabled.</p>
<pre><code>-- how many possible states are here? 
type alias Gid =
    { hasShowMore : Bool     -- whether the feature is enabled
    , showMore : Bool        -- on/off state of the toggle button
    , toggleMore : Maybe msg -- the event fired when toggled
	}
	
-- 2 x 2 x 2 = 8 states in total

eg:

-- the feature is disabled but there is a msg? What is the correct intention here?

hasShowMore = False    -- feature is disabled
showMore = True        -- toggle is on
toggleMore : Just ...  -- a message exists


-- feature is enabled but goes nowhere, wasted a couple of hours here alone ( early days )

hasShowMore = True    -- feature is enabled
showMore = True        
toggleMore : Nothing  -- but clicking does nothing (constant source of bugs)
</code></pre>
<p>How many states should it have, or rather, what should the logic for this feature be constrained by?</p>
<pre><code>type alias ShowMoreToggle = Bool  -- on/off state of the toggle button

type ShowMoreState msg
	= Hidden                      -- feature disabled, no state
	| Visible ShowMoreToggle msg  -- feature enabled, toggle state + msg
	
type alias Grid =
	{ showMoreState : ShowMoreState
	}

-- How many states now?
-- ShowMoreToggle = 2
-- ShowMoreState = 1 + ShowMoreToggle
-- Total of 3 states where the type declarations are self documenting
</code></pre>
<p>So these sort of refactoring are more time consuming because it will force you to handle all the possible states. On the positive side, you’ll <em>never</em> have to consider the invalid cases, not in tests, not in debugging, it is impossible.</p>
<p>A popular excuse ( the one used for the above code ) is, it works for <em>me</em>, <em>I</em> never make these mistakes, why bother spending the effort. Removing impossible state means no-one else will ever make these mistakes either. This is why we have the confidence to let our newer devs go crazy on the codebase because <em>our state and types are locked down.</em> And once your code hits a level where one brain is no longer enough to store it, then this strategy becomes crucial in reducing regressions.</p>
<p>(Aside: Flattening an architecture is not something that I consider to be a core refactor. The reason being that it happens as a <em>consequence</em> of decoupling your msg, model, making things opaque and removing impossible state. Blindly flattening everything to the same level under Main.elm does not result in a decouple architecture so those issues are still there and lead to responsibility breakdowns in overexposing types, sky rocketing compile times etc… but more on this in the <a href="#compile-time">Compile Time</a> .)</p>
<h1 id="compile-time">Compile Time</h1>
<p><em>Coupling, in the form of importing modules, is the key to causing large jumps in compiled files and eventually a blowout in compile time. Anyone who’ve experienced their workflow grind to a halt knows what this feels like. elm-module-graph<sup class="footnote-ref"><a href="#fn7" id="fnref7">7</a></sup> is an excellent tool in diagnosing where your dependencies come from. Making modules Simple<sup class="footnote-ref"><a href="#fn8" id="fnref8">8</a></sup> is the key to having a constant compile time as your application grows.</em></p>
<p>There are two kinds of issues when trying to improve compile time. Compiler/OS optimisation issues<sup class="footnote-ref"><a href="#fn9" id="fnref9">9</a></sup> and your application architecture. The former issues are well known and likely to be addressed in the 0.19 release. The latter is the focus of this section as it will affect most codebases beyond ~10k lines. If your POC is less than this, or your compile time is under 10 seconds, focus on your POC and do not waste time fiddling with compile time. Read something useful like <a href="#refactoring">Refactoring</a> to reduce time spent on bugs.</p>
<h4 id="do-you-have-a-coupling-problem">Do you have a coupling problem?</h4>
<p>Here’s the test:</p>
<ol>
<li>Compile your app</li>
<li>Touch a commonly edited module ( a page, a service, model, controller etc… )</li>
<li>How many files recompile?</li>
<li>Repeat this for a few files and get an average.</li>
</ol>

<table>
<thead>
<tr>
<th>Number of Files</th>
<th>Health of your app</th>
</tr>
</thead>
<tbody>
<tr>
<td>3-5</td>
<td>Healthy! Read this section to laugh at my struggles but otherwise you’re wasting time here.</td>
</tr>
<tr>
<td>5-10</td>
<td>You’ve likely already taken some evasive action, 5 is borderline, with 10, there’s a margin to improve but hopefully the compiler will get smarter before your app needs to. There’s not much to gain here.</td>
</tr>
<tr>
<td>10-30</td>
<td>If you’ve just been hacking along without a care, have ~100 files, this is likely where you’re at. If this is a small hobby project, it’s not a big issue. If you’re at work and this is sizeable production code, shit will eventually hit the fan at the velocity you’re going. Stop. Read this section. Improve.</td>
</tr>
<tr>
<td>30+</td>
<td>There is some serious coupling here. The xkcd on compiling<sup class="footnote-ref"><a href="#fn10" id="fnref10">10</a></sup> is no longer funny because you’re about to lose your job from lack of productivity.</td>
</tr>
<tr>
<td>100+</td>
<td>Do I even need to welcome you to your own personal hell? Read on. (Seriously though, if this is your average, and this section didn’t help you, I’m happy to help you personally. Hit me up on elm-slack, #compile-time, @mordrax)</td>
</tr>
</tbody>
</table><p>So, the above was a bit of fun (need some fun after mashing out 4000 words). The serious point though is that a healthy app should really only be compiling 3-5 files <em>for most changes regardless of the size of the app</em>. This means when you hit 1000 files and 100k LoC, you’re still only compiling 3-5 files. Of course this does not hold for your common files or library files because by definition, they will be imported by alot of modules but more on strategies to mitigate that in <a href="#file-structure">File Structure</a>.</p>
<h4 id="our-story...">Our story…</h4>
<p>We had ~350 files in ~30k LoC. <strong>A typical change affected a 65+ files and compiled in ~2 minutes.</strong> This meant adding a Html.div or a Debug.log took 2 minutes each time. This had a <em>HUGE</em> effect on workflow and team morale. We literally stopped working and gave birth to elm-hack (ported a version<sup class="footnote-ref"><a href="#fn11" id="fnref11">11</a></sup> to my game), our infamous compile aide. I won’t go into the details there, my talk<sup class="footnote-ref"><a href="#fn12" id="fnref12">12</a></sup> with slides<sup class="footnote-ref"><a href="#fn3" id="fnref3:1">3</a></sup> goes into a bit more detail how we were able to get from 2 minutes down to 2 seconds. This bought us enough time to get to a point where management trusted us enough to let me fix the core issue.<br>
Fast forward 3 months, and a complete re-architect of the root modules, we’re now at ~45k LoC in 426 files. A typical change will affect 3-4 files and take 15-30 secs.</p>
<p><em>(Aside: “Complete re-architect” sounds scarrrry, ooohhhhh, a COMPLETE re-architect, aaarrrhhhh. But we’re in Elm. And I had done all the refactorings I mentioned in the refactoring section. I spent 2-3 days to do some major 1-2k lines of reshuffling of modules. Then I spent the rest of the week and another 3-4k lines to rewrite routing, page loading, port existing pages over. One week. Not the end of the world. Had about ~15 bugs, I only remember 3 but I don’t think anyone would believe me if I said that after rewriting 5k LoC, I had three bugs. I remember these bugs because they still had impossible state. That’s the only thing that breaks. reliably. every. time.)</em></p>
<h4 id="case-study--quotes-">Case study ( Quotes )</h4>
<p>To explain why coupling is bad for compile time, I’m going to use a work related example.<br>
In health insurance, there is a concept of <em>Person</em>, these people take up a  either a single, couple or family <em>Membership</em>.</p>
<pre><code>-- Person.elm
type alias Person =  
	{ id : Int  
	, -- more fields
	}

-- Membership.elm
type alias Membership =  
	{ id : Int  
	, -- more fields
	}
</code></pre>
<p>There is a large list of pages which handle various aspects of both:</p>
<pre><code>-- Medicare.elm
import Person exposing (Person)

-- ClearanceCertificate.elm
import Person exposing (Person)

-- Rebates.elm
import Membership exposing (Membership)

-- etc...
</code></pre>
<p>A feature of insurance is we get a quote for a single person or a membership.</p>
<pre><code>-- Quote.elm
type QuoteID 
    = PersonID Int 
    | MembershipId Int
    
type alias Quote =
	{ id : QuoteId
	, -- more fields...
	}
</code></pre>
<p>Now, since one can make a quote in Person or Membership, it makes sense to add it to the respective models.</p>
<pre><code>-- Person.elm
type alias Person =  
  { id : Int
  , quote: Quote
  , ...
  }

-- Membership.elm
type alias MemberPolicy =  
  { id : Int
  , quote: Quote
  , ...
  }
</code></pre>
<p>Visually this looks like the following:</p>
<p><img src="https://preview.ibb.co/mWjMfx/Screen_Shot_2018_03_02_at_9_10_56_pm.png" alt="abc"></p>
<p>Great, we’ve got some code sharing here. Both Person and Membership now shares this quotes component and we’re happy! ( Of course, the avid reader would know that’s not how the story goes.)</p>
<pre><code>touch Quotes.elm
elm-make ...
Compiling 33 modules
</code></pre>
<p>According to my highly accurate ‘Do you have a Coupling problem?’ chart, we ranked (30+):</p>
<blockquote>
<p>There is some serious coupling here. The xkcd on compiling<a href="https://stackedit.io/app#fn7">7</a> is no longer funny because you’re about to lose your job from lack of productivity.</p>
</blockquote>
<p>So we had a common component, it was added to the model where it was needed most, then the view and update logic are shared. <em>What’s the problem?</em></p>
<p>Here’s some food for thought:</p>
<pre><code>touch Person/Page1.elm
elm-make ...
Compiling 1 module

touch Membership/Page15.elm
elm-make ...
Compiling 1 module

touch Membership/Model.elm
elm-make ...
Compiling 15 modules
</code></pre>
<p><strong>When you touch a module (Quote.elm), it will re-compile <em>ALL</em> modules (all pages, person/membership models) that directly or indirectly imports the changed module.</strong></p>
<p>When does it not do this?</p>
<p>Myth #1: If I change a non-exposed function in Quotes.elm it will not re-compile all modules that import it.</p>
<p>Myth #2: If I make my Quote type opaque and only export the opaque type, it will not re-compile all modules that import it.</p>
<p>Myth #3: If I add a comment to Quotes.elm, it will not re-compile all modules that import it.</p>
<p>Fact: Doing anything to Quotes.elm will re-compile all modules that import it.</p>
<p>This is the reason for all non-compiler/OS related compile time blowouts and it can happen very quickly, just with the introduction of one feature, badly coupled.</p>
<p>There is an excellent tool for looking at this specific scenario of coupling in your app, elm-module-graph<sup class="footnote-ref"><a href="#fn7" id="fnref7:1">7</a></sup>. And for the case study, it produces the following graph:</p>
<p><img src="https://preview.ibb.co/jNG3Sc/Screen_Shot_2018_03_02_at_9_30_51_pm.png" alt="elm module graph demonstrating coupling"></p>
<p>So Quotes is the highlighted module. It has 11 blue modules lit up, these are the modules that either import it directly or indirectly. There are 3 black lines that comes out of quotes, these are the modules that imports Quotes directly. The rest of them come out of Model.Person because Model.Person imports Quotes.</p>
<p>This is how it looked after I <em>removed Quotes from the Person/Membership</em> models and passed it into functions that required it.</p>
<p><img src="https://image.ibb.co/djq2Lx/Screen_Shot_2018_03_02_at_9_36_22_pm.png" alt="Refactored version of quotes"></p>
<p>So now Quotes is directly imported by Membership and Person, two <abbr title="A module that has init, update and view, wired up to Main in a nested fashion">TEA</abbr> modules and this is imported by Main.</p>
<pre><code>touch Quotes.elm
elm-make ...
Compiling 4 modules
</code></pre>
<p>Let’s count that, Quotes, Membership, Person, Main. <em>Four</em>!</p>
<blockquote>
<p>Healthy! Read this section to laugh at my struggles but otherwise you’re wasting time here.</p>
</blockquote>
<p>So with this pattern, even if your app reaches 426 files, your compile time will not grow with it.</p>
<h1 id="file-structure">File Structure</h1>
<p><em>Brief look into how an app file structure with 45k LoC and 436 files evolved. Without a pre-meditated plan, our file structure grew as needed, the only exception was to make sure global, popular modules ( lots of imports ) got broken up to avoid compile time hell.</em></p>
<h4 id="how-we-structure">How we structure</h4>
<pre><code>|-- Main.elm         -- obligatory. Fun Fact: Ours is 802 lines and counting (0).

|-- Login.elm        -- simple subsystems are root level modules (1)
|-- Notification.elm -- another simple subsystems

|-- Alfred.elm       -- OBSOLETE: all helper functions (3)
|-- Alfred           -- helper folder, so named to prevent namespace clashing

|-- Components       -- stateful components (2)

|-- Membership       -- large subsystems group into a folder
|  |-- Api.elm       -- OBSOLETE: all http helper functions (3)
|  |-- Forms.elm     -- Explained in the Forms section (4)
|  |-- Forms
|-- Membership.elm   -- large subsystems generally have a top level TEA module to distribute msg

|-- Model            -- aka Repository, business logic for get/set of data (5)

|-- Person           -- another large subsystem
|  |-- Forms
|  |-- Forms.elm
|-- Person.elm

|-- Search           -- YALS (YASD?)
|-- Search.elm

|-- Ports.elm        -- all our ports in one place

|-- Types            -- non domain specific, non module specific, custom types
|  |-- Api           -- all endpoint types
|-- Types.elm        -- global Type, wait... I can explain (6)

|-- Validators       -- common validation helpers
|-- View             -- 
|  |-- Components    -- UI components
|  |-- Fields        -- domain specific UI components to make building forms easier
</code></pre>
<h4 id="why-we-structure-it-so">Why we structure it so</h4>
<p>Some commentary on our file structuring.</p>
<ol start="0">
<li>
<p>(<em>Fun Fact: Ours is 802 lines and counting</em>) We do not care about file size. The last time I looked at a file size was… to quote the line above but no, the time before then was… never. When considering what goes into a file, the <em>only</em> consideration is: Does this module fulfil one responsibility. When a module does more (<em>or less</em>) than  this, coupling occurs. Every. Single. Time.</p>
</li>
<li>
<p>(<em>Simple subsystems are root level modules</em>) There is a bunch of files at root level, they are typically directly imported by Main or they are global like Types.Elm or Routes.elm</p>
</li>
<li>
<p>(<em>Stateful Components</em>) The elm-guide says don’t use <em>reusable components</em> but doesn’t actually define what it is. In our experience, certain patterns of components have emerged. We discuss these in depth in <a href="#components">Components</a>. Hint: UI state and Domain state should not be grouped under the same terminology of ‘state’.</p>
</li>
<li>
<p>(<em>OBSOLETE: all helper functions</em>) So we used to group helper functions in the one module as you do in <em>most</em> other programming languages (disclaimer: I only know C#, Ruby, Java, Javascript does this). However, as the avid reader would know from having already read <a href="#compile-time">Compile Time</a>, this is actually a really bad idea in Elm because a helper module by definition is a toolbox for the rest of your app.</p>
<pre><code>touch Alfred.elm
elm-make ...
Compiling 105 modules (Have a nice day!)
</code></pre>
<p>Ah compiler, always so friendly.</p>
<p>Instead, we now have a folder called <abbr title="The helpful butler whom learned a bunch of useful FP things and helps us avoid namespace conflicts.">Alfred</abbr> and try to break our helpers up into modules that are commonly used together.</p>
</li>
<li>
<p>(<em>Explained in the Forms section</em>) <a href="#forms">Forms</a>, what a mess.</p>
</li>
<li>
<p>(<em>aka Repository, business logic for get/set of data</em>) We don’t use opaque types and it’s already bitten me twice. As a team grows and introduces more junior/non-domain experts, I think it’s definitely worth making domain models opaque. This folder contains all domain specific helpers (e.g Membership.getDependants will sort by age). Unfortunately, domain logic is also tied in page updates, in the page views and validators.</p>
</li>
<li>
<p>(<em>global Type, wait… I can explain</em>) So Type.elm contains Id tags. The kind that looks like this:</p>
<pre><code>type alias PersonId 
    = Int
    
type QuoteId 
    = Person PersonId 
    | Member MemberId
</code></pre>
<p>The compile time on this is tolerable because these IDs are defined by the backend and change rarely.</p>
</li>
</ol>
<h1 id="components">Components</h1>
<p><em>I present the various types of components we use, discuss their purpose and provide examples. UI Components are arguably the simplest form and everyone should have them. Stateful/Stateless and Hybrid components are patterns we defined along the way as we refactored our codebase to keep domain logic together.</em></p>
<p>I use the term <em>components</em> fairly loosely here. By a component, I mean a function or a module which does no more or less to achieve one behaviour and acts <em>independently</em> of the calling code. Some will have <code>msg</code> others will have <code>Msg</code>, others may have state, some will just use extensible records <code>{ a | ... }</code>.</p>
<h4 id="ui-components">UI components</h4>
<p>We start with the simplest form of components and it looks like this:</p>
<pre><code>input : String -&gt; (String -&gt; msg) -&gt; Html msg  
readonlyTextInput : String -&gt; Html msg

dateInput : DateTime -&gt; (DateTime -&gt; msg) -&gt; Html msg
maybeDateInput : Maybe DateTime -&gt; (Maybe DateTime -&gt; msg) -&gt; Html msg

ageInput : Int -&gt; (Int -&gt; msg) -&gt; Html msg
emailInput : String -&gt; (String -&gt; msg) -&gt; Html msg
</code></pre>
<p>These are great, you give it the current state, a msg to call and it will give you a UI control.<br>
We also do dropdowns, which are all typed. They are either a enum or a dynamic list of types. For the latter they are a record, not strictly typed.</p>
<pre><code>genderDropdown : Lookup -&gt; Int -&gt; (Int -&gt; msg) -&gt; Html msg  
genderDropdown lookup =  
  Dropdown.enumDropdown (Lookup.genders lookup)  -- gender types are fixed (Enum)
  
bsbDropdown : Lookup -&gt; String -&gt; (String -&gt; msg) -&gt; Html msg  
bsbDropdown lookup =
  Dropdown.dropdown (Lookup.bsb lookup)  -- bsb is variable (Lookup table)
</code></pre>
<p>This means our forms can use fields like this:</p>
<pre><code>View.Layout.paneField "Gender" (View.Components.genderDropdown lookup model.gender Gender)
</code></pre>
<p>It took a bit of time to evolve our UI components to the function signatures above. I kept removing things until they had the minimal arguments required to make a function and when I couldn’t take any more out, then I knew I was done.</p>
<p>Due to how our data types are in the backend, there are <code>Maybe</code> variants of these components to make handling maybe types easier.</p>
<h4 id="stateless-components">Stateless components</h4>
<p>A stateless component does not hold state. If you think that statement was obvious, then we have successfully named them. They look like this:</p>
<pre><code>type alias Address a =  
  { a  
	  | addressLine1 : String  
	  , countryCode : String  
	  , postcode : Int  
	  , state : String  
  }

update : Lookup -&gt; Msg -&gt; Address a -&gt; Address a
view : Lookup -&gt; Bool -&gt; Address a -&gt; Html Msg
validate : Address a -&gt; List String
</code></pre>
<p>Couple of things to note here. To make it possible to interact with the rest of the world with no state, the component defines a extensible record and interacts with that. The component is like a service, it contains <em>domain logic</em> to handle edits for these fields. It knows how to display them, update the fields and we also include all validations in the component.</p>
<p>This way our various pages or even other components could pick this up and say, hey! please handle <em>my</em> address state for me. And do the validation for it.</p>
<p>Sure enough our AddressesEditor is one such happy customer:</p>
<pre><code>type alias AddressesEditor a =  
  { defaultAddress : Address a  
  , addresses : List (Address a)  
  , addressType : Enums.AddressType
  }
</code></pre>
<p>It’s concerned with different types of addresses like a home, work, delivery address and happily uses the AddressEditor.elm component to handle updating single addresses.</p>
<p>AddressesEditor is a <a href="#stateful-components">Stateful Component</a>, read on!</p>
<h4 id="stateful-components">Stateful components</h4>
<p>Stateful components hold state. If that was obvious… see <a href="#stateless-components">Stateless components</a>.</p>
<p>An Address<strong>es</strong>Editor (not to be confused with an AddressEditor) is one such component, it looks like this (Note, this type of component has fallen out of favor for the reasons outlined at the end of this code block):</p>
<pre><code>type alias AddressesEditor a =  
  { defaultAddress : Address a  
  , addresses : List (Address a)  
  , addressType : Int  
  , viewMode : ViewMode  
  }
  
init : Address a -&gt; AddressesEditor a  
update : Lookup -&gt; Msg -&gt; AddressesEditor a -&gt; ( AddressesEditor a, Job Msg )
view : Lookup -&gt; msg -&gt; (Msg -&gt; msg) -&gt; AddressesEditor (AddressWithBarcode a) -&gt; Html msg
validate : AddressesEditor a -&gt; List String

getAddresses : AddressesEditor a -&gt; List (Address a)
getAddressType : AddressesEditor a -&gt; Int
getAddress : Int -&gt; AddressesEditor a -&gt; Maybe (Address a)
</code></pre>
<p>So here the component holds addresses as well as the type of address that’s currently being edited and various ways to display the component. It’s view function is already looking a bit cluttered and there is a bunch of getters. These actually expose functions to return state to the parent <em>but you have to remember to do this</em>. And if the rest of the world has moved on in the meantime (ie the address in the parent changed), the editor would be holding stale data and worse, saving stale data.</p>
<p>The core issue here is that the component is <em>holding model state that belongs to the parent</em>. So we no longer use this form of components.</p>
<p>Our current way of doing stateful components is for the component to hold only UI state and pass parent state in as the stateless components do, ie, our Grid.</p>
<pre><code>type alias StatefulGrid subject =  
  Grid subject (Msg subject)
  
type alias Grid subject msg =  
  { columns : List (Column subject msg)  
  , selected : subject -&gt; Bool  
  , rowClicked : Maybe (subject -&gt; msg)  
  , pageSizeChanged : Maybe (Int -&gt; msg)  
  , pageClicked : Maybe (Int -&gt; msg)  
  , pageSize : Int  
  , pagePosition : Int  
  , comparer : subject -&gt; subject -&gt; Bool  
  , selection : List subject  
  , multiSelect : Bool  
  , sorter : subject -&gt; subject -&gt; Order  
  , sortGrid : Bool  
  , stateful : Bool  
  , rowAttributes : subject -&gt; List (Html.Attribute msg)  
 }

initPaging : (subject -&gt; subject -&gt; Bool) -&gt; StatefulGrid subject
update : Msg subject -&gt; StatefulGrid subject -&gt; StatefulGrid subject
addCol : String -&gt; (subject -&gt; comparable) -&gt; Grid subject msg -&gt; Grid subject msg
render : List subject -&gt; Grid subject msg -&gt; Layout.Block msg 
</code></pre>
<p>Let’s not look too deep into the <code>type alias Grid</code>, it’s a great example of how <strong>not</strong> to write bug free, minimal state type aliases. The point here is that <code>Grid</code> only holds <em>UI state</em>. It sorts, it selects, it pages and these are reflected in it’s UI state. When it comes time to call the view function, we pass the data into it.</p>
<p>Usage looks like this:</p>
<pre><code>Grid.empty  
    |&gt; Grid.addCol "Component" .description  
    |&gt; Grid.addCol "From Date" (.fromDate &gt;&gt; Alfred.Dates.toDate)  
    |&gt; Grid.render waitingPeriods
</code></pre>
<h4 id="hybrid-components">Hybrid components</h4>
<p>So our stateless components define the data that they interact with in terms of a extensible record. Our stateful components only hold UI state. Naturally then, if there is a component that both requires UI state and operates on the domain model we combine the two types of components above:</p>
<pre><code>type alias ReceiptMethod a =  
 { a  
  | receiptMethodTypeId : String  
  , methodReference : String  
  , methodReferenceName : String  
  , methodDate : DateTime  
  }  
  
  
type alias ReceiptMethodEditor =  
 { newRecipientToggle : NewRecipientToggle  
 }

update : Lookup -&gt; Msg -&gt; ReceiptMethod a -&gt; ReceiptMethodEditor -&gt; ( ReceiptMethod a, ReceiptMethodEditor, Job Msg )
view : Lookup -&gt; ReceiptMethod a -&gt; ReceiptMethodEditor -&gt; Html Msg
</code></pre>
<p><code>ReceiptMethod a</code> is a extensible record and operates on the <code>domain model</code>. <code>ReceiptMethodEditor</code> holds <em>UI state</em> that no parents care about.<br>
The <code>update</code> function explicitly takes in both separately and returning both. This <em>forces</em> the developer to use it’s parent’s domain model thus not duplicating the state and removes the risk of forgetting to update it in the parent as was the problem with <code>AddressesEditor</code>.</p>
<h4 id="component-wizardry">Component Wizardry</h4>
<p>I just wanted to show off our ‘wizard’ component and demonstrate another kind of component.</p>
<pre><code>type alias Wizard oz msg =  
 { steps : ZipList (Step oz msg)  
 }  
  
type alias Step state msg =
  { validStep : state -&gt; Bool
  , init : state -&gt; ( state, Job msg )
  , view : state -&gt; Html msg
  , validate : state -&gt; List String
  }
  
next : state -&gt; Wizard state msg -&gt; ( Wizard state msg, state, Job msg )
back : state -&gt; Wizard state msg -&gt; ( Wizard state msg, state, Job msg )

init : List (Step state msg) -&gt; Wizard state msg
view : state -&gt; Wizard state msg -&gt; Html msg
makeStep : (state -&gt; Html msg) -&gt; (state -&gt; List String) -&gt; Step state msg
</code></pre>
<p>Used like this:</p>
<pre><code>Wizard.init
    [ Wizard.makeStep (viewBase lookup) (always [])
    , Wizard.makeStep (viewNewDependant lookup) validateNewDependant
        |&gt; Wizard.withCondition (...)
    , Wizard.makeStep (viewMemberTransfer lookup) validateMemberTransfer
        |&gt; Wizard.withInit ...a Job
        |&gt; Wizard.withCondition (\state -&gt; ... True/False)
    ]


-- on next
case Wizard.validate state wizard of
    [] -&gt;
        let
            ( nextWizard, nextState, job ) =
                Wizard.next state wizard
        in
              
-- on view          
Wizard.view state wizard
</code></pre>
<p>So our wizard component holds a bunch of functions. It doesn’t hold the steps, this is passed in. Next happens once a step is validated. Steps are conditional, they can contain initial jobs (Cmds + extras). Having a component like this is a massive time saver for multi-stepped forms which we seem to have more of now that there’s a wizard.</p>
<h1 id="forms">Forms</h1>
<p><em>You have a component that shares behaviour ( open, close, save ). You try to make it generic. It does not work. You try another way. It does not work. You try again. It does not work. You think about it for weeks, months. It does not help. You fail and write a 10,000 word essay about how you failed to make generic forms.</em></p>
<p>Firstly, let’s define what I mean by a form. It’s a <abbr title="A module that has init, update and view, wired up to Main in a nested fashion">TEA</abbr> module with some common characteristics. All forms have validation, the ability to save, only one can be open at any given point in time and you can open and close them.</p>
<pre><code>-- src/Membership/Forms.elm
type FormState
    = NoForm
    | AddressState Address
    | ... x30 forms
    
-- src/Membership/Forms/Address.elm

type alias Address = ...
type Msg = ...
init : Address
update: Msg -&gt; Address -&gt; (Address, Job Msg)  
view: Address -&gt; Html msg
save: Address -&gt; Result (List String) (Job Msg)
validate: Address -&gt; List String
</code></pre>
<p>Also their layout is very similar, e.g the same header bar with save/cancel buttons.  This is driven by the fact we want the user to recognise a form when they see one.</p>
<p>This is a prime case for making a generic form module to handle this. <em>Surely</em>.</p>
<p>I’ve rewritten the mechanism around opening, closing, saving and viewing of a form three times and present them below.</p>
<h4 id="approach-0-forms-as-pages-share-view">Approach 0: Forms as pages, share view</h4>
<p>The simplest solution is to wire up each form like a <abbr title="A module that has init, update and view, wired up to Main in a nested fashion">TEA</abbr> module. The parent will <code>init</code> to open and the form will respond with a tuple to close itself. Since their layout is similar, we used a common view function.</p>
<pre><code>-- src/Membership/Forms/Address.elm
update: Msg -&gt; Address -&gt; ( Address, Bool, Job Msg)
update msg address =
	case msg of
	    Close -&gt; ( address, True, Job.init )		

view : Address -&gt; Html Msg
view model =
    let
        buttons = 
            [ View.Component.button "Save" Save ]
    in
    View.Form buttons (render model)
</code></pre>
<p>But this means that each form will have to implement their own validate, save and response which are also common.</p>
<pre><code>update: Msg -&gt; Address -&gt; ( Address, Bool, Job Msg)
update msg address =
	case msg of
	    Save -&gt; 
	        case validate address of
	            [] -&gt; 
	                (address, False, addressApi address |&gt; Job.fromTask SaveResponse)
	            errs -&gt; 
	                ({ address | validationErrors = errs }, False, Job.init)

        SaveResponse response -&gt;
			-- common code for all forms
</code></pre>
<p>So what are the good and bads of this approach.</p>
<p>Pros:</p>
<ul>
<li>The view uses a helper and provides a consistent layout</li>
<li>Each form is independant and arguments, return value do not have to be homogenous</li>
</ul>
<p>Cons:</p>
<ul>
<li>There appears to be alot of repeat code in the update and the view to handle saves, validation, closing etc…</li>
</ul>
<p>So by this time I was itching for some component.</p>
<h4 id="approach-1-forms-manager-module">Approach 1: Forms manager module</h4>
<p>The common theme around forms is that we were wiring the save, response and view up for every form, repetitive. So by having a top level manager that held the form state and the msgs, forms could just expose <code>save</code>, <code>validate</code> and <code>update</code> for the manager to call. It would also handle validation errors in the one place!</p>
<pre><code>-- Forms.elm
type alias Forms =
    { formState : FormState
    , validationErrors : List String
    }
    
type FormState
    = NoForm
    | AddressState Address
    | ... x30 forms

view: Forms -&gt; Html Msg
view { formState, validationErrors } =
	let
		headers = 			
			[ View.Components.button "Save" Save
			, View.Components.button "Close" Close
			, viewErrors validationErrors
			]
			
		render body = 
			div []  [ headers, body ]
	in
	case formState of
		AddressState address -&gt;
			render (Address.view address)			
</code></pre>
<p>However, each msg that goes to the form now will have to have a corresponding <code>case ... of</code> to delegate it to the right form.</p>
<pre><code>update : Msg -&gt; Forms -&gt; (Forms, Job Msg)
update msg {formState,  =
	case msg of
		FormMsg formMsg-&gt;
			updateForm formMsg formState
		Save -&gt;
			saveForm formState

updateForm formMsg formState =
	case (formMsg, formState) of
		(AddressMsg msg, AddressState state) -&gt; 
			Address.update msg address
			
		... x30 forms

saveForm formState =
	case formState of
		AddressState state -&gt;
			Address.save state

		... x30 forms
</code></pre>
<p>So let’s analyse this approach.</p>
<p>Pros:</p>
<ul>
<li>in view, <code>render</code> provides a consistent layout</li>
<li>handling of form close, validate and save have been moved to a central place so the form itself doesn’t have to repeat that code</li>
<li>since all validation is of the same type, we only need to refer to a single validationErrors state in the view (actually, this is a opportunity for bugs… so this is not a pro)</li>
</ul>
<p>Cons:</p>
<ul>
<li>handling of validation in a common place can ( and has ) caused a bug where we forgot to clear it when closing a form. This would be trivially mitigated by composing the form state with the validation errors though ie <code>type alias Forms = { formState : ( FormState, List String ) }</code> or creating a <code>empty : Forms</code> function.</li>
<li>it really hasn’t generalised very much. It’s moved the ‘forms’ handling out of each of the form which is great w.r.t decoupling the form mechanics to what it does but having each msg type being accompanied by a <code>case ... of</code> that spans all the forms is undesirable. Luckily, our forms component only has a handful of common msg types.</li>
</ul>
<p><em>Aside: This is where we are at now, at the time of writing this article. Do not do this… it is no better than whatever else you’re doing, fairly certain.</em></p>
<h4 id="approach-2-forms-manager-module-with-an-interface">Approach 2: Forms manager module with an interface</h4>
<p>I don’t know what these are, but coming from the OO world, I shall call them… interfaces</p>
<pre><code>type alias IForm formMsg formState =
    { update : Repository -&gt; formMsg -&gt; formState -&gt; ( formState, Job formMsg )
    , save : Repository -&gt; formState -&gt; Result (List String) (Task Http.Error MemberPolicyDelta)
    , view : Repository -&gt; formState -&gt; Html formMsg
    , state : formState
    }
</code></pre>
<p>So this is great, it allows me to do the following:</p>
<pre><code>-- define a IForm

type FormState  
  = NoForm  
  | AddressState (IForm Address.Msg Address)
  | ... x30 IForms


-- initialise a form

MemberAddressForm addressType -&gt;
    AddressState
        { update = Address.update
        , save = Address.save
        , view = Address.view
        , state = Address.init addressType membership.policy.addresses
        }
        |&gt; returnFormState

-- then in the update
updateIForm : IFormMsg -&gt; Forms -&gt; (Forms, Job Msg)
updateIForm formMsg { formState } =
	case formState of
	    in
    case forms.formState of
        AddressState form -&gt;
            applyMsg AddressState AddressMsg form
		... x 30 forms

-- applyMsg gets a little bit hairy, let's walk through it
-- the key here is that we get the generic save, update, state from the
-- IForm
applyMsg toFormState toFormMsg { save, update, state } =
    case msg of

		-- then we can apply the form's msg and state to the form's update
		-- that the form provides, meaning we don't need the massive case ... of
        IFormFormMsg formMsg -&gt;
            update repository formMsg state

	            -- this is just my shorthand for mapping state and msg
	            -- back to the Forms.elm level of types
                |&gt; mapStateAndJob toFormState toFormMsg

		-- similarly, we apply the form's state to the form's save
		-- which means we skip on the big case ... of here as well
        IFormSave -&gt;
            case save repository state of
                Result.Ok httpRequest -&gt;
                    ( { forms | saveState = Saving }, httpRequest )

                Result.Err validationErrors -&gt;
                    { forms | validationErrors = validationErrors }
                        |&gt; (\forms_ -&gt; (forms_, Job.init ))
</code></pre>
<p>It’s ok if you didn’t quite follow the whole (poorly presented) example above, the main idea here is that by making forms homogeneous and applying it to a single type, we no longer need a case … of to handle each form behaviour.</p>
<p><em>But</em>.</p>
<p>In our case, it actually wasn’t worth it. Because we still need one to route the correct msg/state to the right form and we still needed one for the view… <em>because is it a union type</em>.<br>
And we still needed one for the <code>init</code> of a form.</p>
<p>Pros:</p>
<ul>
<li>The single IForm collects all update msgs into one <code>case ... of</code>.
<ul>
<li>Impossible to make a logic error in form handling for a new form.</li>
<li>Forms manager does not grow with each new feature of a form</li>
</ul>
</li>
<li>View and layout is still separated from the form</li>
<li>New form creation are very streamlined</li>
</ul>
<p>Cons:</p>
<ul>
<li>More complex than the ‘slap it on as you go’ solutions</li>
<li>Feels like we’re reimplementing <abbr title="A module that has init, update and view, wired up to Main in a nested fashion">TEA</abbr> ( which is what stopped me the first two times )</li>
<li><em>Makes all forms homogeneous</em>. So the <code>Repository</code> that is passed into each form now holds quite a bit of unnecessary state because the interface has to take in the same datatypes and thus the end result is the lowest common denominator of state. <strong>This feels very bad</strong> as it goes against the whole minimalist approach we’ve used throughout the rest of the project to reduce impossible state and decouple modules.</li>
</ul>
<h4 id="approach-3-forms-as-a-component">Approach 3: Forms as a component</h4>
<p>So much in the ECS vein, instead of saying a address form <em>is</em> a form ( approach 0 ) or that there is a form manager that <em>has</em> all forms ( approaches 1 and 2 ), we say that a page is <em>composed of</em> an address editor and a form.</p>
<p>Um actually, I haven’t made this change yet ( as there isn’t enough of a business case to do this atm ) so I’ll let the avid reader try to link this section up with <a href="#components">Components</a> and see what results.</p>
<h4 id="conclusions">Conclusions</h4>
<p>So my feeling about how we’ve done this is as unsatisfactory as how I’ve left the reader on Approach 3. I hope the pros and cons have been useful in analysing the various approaches of making a re-usable component or at least that I’ve steered you away from the above component approaches because the more I discuss them, the more I dislike them.</p>
<h1 id="endpoints">Endpoints</h1>
<p><em>I discuss our auto-generated datatypes from the backend and how it has benefited and hindered us when using these. Ideally, you want your en/decoders to be generated for you as it leaves no room for misinterpretation. Having generated type aliases means you can also generate setters. Json.Decode.Pipeline is great, until you need to fold over extensible records which all our endpoints are becoming.</em></p>
<p>So this is very exciting because recently, we’ve taken our domain specific datatypes from good to <em>great</em>! But to explain how that works, I need to start at the beginning. Our backend is sql + stored procs + C#. We’re very lucky in that our backend guy loves to structure his code, create interfaces, write meta-code everywhere and hack Elm on the side.</p>
<p>We have over 100 datatypes right now, and they are all generated by the backend. So our <code>src/Types/Api</code> folder contains over 100 files and 16k Loc (1/3 of our app). These datatypes are used for two things.</p>
<ol>
<li>Every endpoint we hit will communicate via one of these, so we <em>never</em> get errors about using the wrong field or having the wrong type. The decoder is generated using the backend and resolves to type aliases or primitive types. Soon, union types too.</li>
<li>These types are used in the clientside as domain models and since their types are known, they have setters, encoders, init all auto-generated too. It saves alot of time and manual wiring up.</li>
</ol>
<h4 id="domain-specific-datatypes">Domain specific datatypes</h4>
<p>Here’s what one of them looks like:</p>
<pre><code>module Types.Api.Person exposing (..)  
  
{-| Auto generated from ... C# library...
-}  
  
import Json.Decode as JD  
import Json.Decode.Pipeline as JDP  
import Json.Encode as JE

  
type Field
    = Id Int
    | PersonDetail PersonDetail
    | ...


update : PersonField -&gt; Person -&gt; Person
update f o =
    case f of
        Id v -&gt;
            { o | id = v }

        PersonDetail v -&gt;
            { o | personDetail = v }

type alias Person =
    { id : Maybe Int
    , personDetail : PersonDetail
	, ... }

init : Person
init =
    { id = Nothing
    , personDetail = PersonDetail.init
    ,... }

decoder : JD.Decoder Person
decoder =
    JDP.decode Person
        |&gt; JDP.optional "id" JD.int 0
        |&gt; JDP.optional "personDetail" PersonDetail.decoder PersonDetail.init

encoder : Person -&gt; JE.Value
encoder o =
    JE.object
        [ ( "id", HP.encodeMaybe JE.int o.id )
        , ( "personDetail", PersonDetail.encoder o.personDetail )
</code></pre>
<p>So as you can imagine, the Person object has alot of data, it can have nested types but these are also auto-generated and the type resolved by the backend through reflection. It means that the frontend doesn’t have the opportunity to screw up encoding/decoding as it’s strongly typed for us already.</p>
<p>The <code>Field</code> type is exposed, so any modules using this type will just use it’s <code>update</code> function to update the record.</p>
<p>This essentially becomes a <abbr title="A module that has init, update and view, wired up to Main in a nested fashion">TEA</abbr> module. All our pages are built on one to many of these.<br>
eg, when creating a person, we use the <code>CreatePersonRequest</code> type, so the page looks like this:</p>
<pre><code>-- NewPerson.elm

type alias NewPerson =
    { request : CreatePersonRequest
    , step : Step
    , errors : List String
    }


init : Lookup -&gt; ( NewPerson, Job Msg )
init lookup =
    let
        initRequest =
            CreatePersonRequest.init

update : Msg -&gt; NewPerson -&gt; Lookup -&gt; ( NewPerson, Job Msg )
update msg panel lookup =
    case msg of
        CreatePersonRequestMsg msg -&gt;
            let
                request =
                    CreatePersonRequest.update msg panel.request
            in
            ( { panel | request = request }, Job.init )

view : Lookup -&gt; NewPerson -&gt; Html Msg
view lookup panel =
	...
    [ ( "Title", Components.titleDropdown lookup request.title (CreatePersonRequest.Title &gt;&gt; CreatePersonRequestMsg) )
    , ( "Surname", Components.input request.surname (CreatePersonRequest.Surname &gt;&gt; CreatePersonRequestMsg) )
</code></pre>
<p>So the page owns the data in <code>NewPerson.request</code>, it uses the auto-generated <code>CreatePersonRequest.init</code> and routes updates to <code>CreatePersonRequest.update</code> and finally maps the msgs in the view to it’s exposed <code>Field</code> via <code>CreatePersonRequest.Surname &gt;&gt; CreatePersonRequestMsg</code>.</p>
<p>Prior to this, the code was copying the domain model into a client view model and duplicated each field and rewrote the setter for each field in each form that needed it. I deleted about 5k LoC when we moved to this approach.</p>
<h4 id="generic-domain-specific-datatypes">Generic domain specific datatypes</h4>
<p>The above pattern works very well for us, but as our components became more generic (<a href="#hybrid-components">Hybrid components</a>) we encountered a problem. If a component which does not use a record but a extensible record wants to send a request out, it cannot because if you remember, the encoder is fixed to a particular type <code>encoder : Person -&gt; JE.Value</code>, well, not to worry, we can make this generic. <code>encoder : Person a -&gt; JE.Value</code> But then, we had to change the alias <code>type alias Person a = { a | ... }</code> and that means changing every single reference to <code>Person</code> and all the other 100+ datatypes spread throughout the code. It would be a <em>massive</em> refactor.</p>
<p>The solution was in defining both a generic and a concrete record:</p>
<pre><code>-- we introduce a interface that covers all of Person
type alias IPerson a = 
    { a
        | id : Int
        , personDetail : PersonDetail
        , ...
    }

-- we make Person the same for all intents and purposes
type alias Person =
    IPerson {}

-- no one's the wiser
init : Person
decoder : JD.Decoder Person

-- but now we can auto-generate a generic encoder
encoder : IPerson a -&gt; JE.Value
</code></pre>
<p>This means the caller, our hybrid component, doesn’t need to know about the record it’s handling either.</p>
<pre><code>-- some new age hybrid.elm

guessAtGender: Person a -&gt; ...
guessAtGender impersonator =
	Person.encode impersonator	
</code></pre>
<p>One stumbling block was that I realised that Json.Decode.Pipeline<sup class="footnote-ref"><a href="#fn13" id="fnref13">13</a></sup> couldn’t decode into a extensible record because the way it works is expose a curried record constructor and fills in each field on each pipe which gives it quite a nice usage.</p>
<pre><code>decoder : JD.Decoder Person
decoder =
	-- JDP.decode just returns a JD.succeed, which looks like 
	-- ( a -&gt; b -&gt; c -&gt; d -&gt; Person) assuming Person had 4 fields
    JDP.decode Person
	       -- gives the curried function one field
        |&gt; JDP.optional "id" JD.int Nothing
	       -- gives the curried function a second field
        |&gt; JDP.optional "personDetail" PersonDetail.decoder PersonDetail.init
	       -- etc...
	       -- last field, now I have a JD.Decoder Person, yaya!!
	    |&gt; JDP.optional ....  
</code></pre>
<p>So rather than use the constructor, and since the <code>update</code> function for each <code>Field</code> is conveniently given to us, we called upon <abbr title="The helpful butler whom learned a bunch of useful FP things and helps us avoid namespace conflicts.">Alfred</abbr>.</p>
<pre><code>optional : String -&gt; JD.Decoder a -&gt; a -&gt; (a -&gt; field) -&gt; (field -&gt; model -&gt; model) -&gt; model -&gt; JD.Decoder model
optional jsonField decoder default toField update model =
    JD.field jsonField decoder
        |&gt; JD.maybe
        |&gt; JD.map (Maybe.Extra.unwrap (toField default) toField)
        |&gt; JD.map (\field -&gt; update field model)
        
required : String -&gt; JD.Decoder a -&gt; (a -&gt; field) -&gt; (field -&gt; model -&gt; model) -&gt; model -&gt; JD.Decoder model
required jsonField decoder toField update model =
    JD.field jsonField decoder
        |&gt; JD.map toField
        |&gt; JD.map (\field -&gt; update field model)
</code></pre>
<p>If the above doesn’t make any sense, it’s ok, you only need to understand the usage:</p>
<pre><code>decoder : JD.Decoder Person
decoder =
    [ optional "id" JD.int 0 Id update
    , optional "personDetail" PersonDetail.decoder PersonDetail update
    , ...
    ]
        |&gt; List.foldl JD.andThen (JD.succeed init)    
</code></pre>
<p>If this still doesn’t make sense, that’s ok. Maybe come back in a while :) The gist of it is that we use <code>JD.andThen</code> to fold over the initial state <code>JD.succeed init</code> much like Json.Decode.Pipeline but rather than adding to a curried function, we fold over an actual record and update each field as we decode them.</p>
<p>So there you have it, auto-generated domain datatypes which decodes from all endpoints and encodes to them as well, being flexible enough to handle extensible records and handling setters. Thanks backend C# guy (who actually really hates M$, EF, .Net cause he’s a Java guy)!</p>
<h1 id="extensible-type-hell">Extensible type hell</h1>
<p><em>Extensible types are great! (as long as you read the fine print). Step outside of it’s basic usage and the type errors you get exponentially increase in confusion. I present various ways in which we use extensible types and one issue to look out for.</em></p>
<p>Our <a href="#components">Components</a> make heavy use of extensible types, because by definition, they are only a component if they are shared between at least one other page/component/thing. So the type they work with has to be parameterised. eg:</p>
<pre><code>type alias Address a =  
  { a  
	  | addressLine1 : String  
	  , countryCode : String  
	  , postcode : Int  
	  , state : String  
  }
</code></pre>
<p>We have a certain model helper that works off common fields and types. Since the backend is heavily constructed via interfaces, the endpoints we have can contain alot of common fields at times. Oracle helps reduce getters:</p>
<pre><code>-- Oracle.elm
type alias HasSurnameFirstName a =
    { a | surname : String, firstName : String }

type alias HasId a =
    { a | id : Maybe Int }

{-| Returns: Jones, Bob ( 123 )
-}
surnameFirstnameId : HasSurnameFirstName (HasId a) -&gt; String
surnameFirstnameId { surname, firstName, id } =
    surname ++ ", " ++ firstName ++ " ( " ++ (id |&gt; Maybe.map toString |&gt; Maybe.withDefault "") ++ " ) "
</code></pre>
<p>Oracle is great, it allows us to fetch complex relationships in our data models by specifying the minimal fields required and also allows us to mix and match these type aliases. It also means that the functions are not tied to any particular record and we have <em>alot</em> of records that repeat fields.</p>
<p>Practically though, we don’t actually use this as much as the creator would have wished because the more complex getters typically come with domain logic which looks into our Model folder for logic.</p>
<p>Lastly, I touch on a type inference issue.</p>
<h1 id="child-to-parentchildsiblingfirst-cousin-2nd-removed-communication">Child to Parent/Child/Sibling/First Cousin 2nd removed Communication</h1>
<p><em>Child to parent communication is a misnomer because how you communicate state is different to msgs and how you communicate to your immediate parent will be different to talking to an arbitrary child or root node. This section talks about some common forms we use and one custom solution we made. This has currently catered for all our needs well enough. I stress, we do not have a generic solution to this non-generic problem.</em></p>
<h4 id="tuples">Tuples</h4>
<p>So we saw in <a href="#hybrid-components">Hybrid Components</a> that tuples are passed back.<br>
<code>update : Msg -&gt; ReceiptMethod a -&gt; ReceiptMethodEditor -&gt; ( ReceiptMethod a, ReceiptMethodEditor, Job Msg )</code>. This is the simplest form of child to parent. The parent says, here’s my model, you know what to do with <code>ReceiptMethod a</code>, do it and give it back to me. Imagine how else we could have a child work on the parent’s model. The parent would either have to pass down a bunch of messages for the child to use ( Translator ) or the child would make a copy of the parent’s model ( nested <abbr title="A module that has init, update and view, wired up to Main in a nested fashion">TEA</abbr> ).</p>
<p>We use this pattern in most forms.</p>
<pre><code>update : Msg -&gt; Contact a -&gt; Contact a
update : Msg -&gt; Address a -&gt; ( Address a , Job Msg)
update : Msg -&gt; Medicare a -&gt; MedicareEditor -&gt; ( Medicare a, MedicareEditor, Job Msg )
</code></pre>
<p>Also, designing pages and forms such that they only have one level of nesting under them means we don’t thread tuples of state up and up, just a single level.</p>
<h4 id="msg---parentmsg">(Msg -&gt; parentMsg)</h4>
<p>A module’s <code>view : Model -&gt; Html Msg</code>will only emit the module’s msg and then you can <code>Html.map</code> it to the parent. But this means that the child view cannot communicate any other messages. The only case we have needed it for is routing.</p>
<pre><code>view : (Route -&gt; msg) -&gt; (Msg -&gt; msg) -&gt; Model -&gt; Html msg
view toRoute toParent model = ...
</code></pre>
<p>The <code>Route</code> type is therefore a global type which all modules know about and so the child can pass that up to the main level to action.</p>
<h4 id="pubsub-ports-aka-wormholes">PubSub ports (aka wormholes)</h4>
<p>I need to preface this section with context: Very early on, the app was in a real mess. Modules would import 3 levels of it’s parent’s models just to construct it’s parent’s model and pass that up in a message. There was some <em>really</em> convoluted child to parent communication going on and one suggestion was to just go into ports.</p>
<p>The suggestion was to do something like this:</p>
<pre><code>{-| Not safe, use publish
-}
pub : String -&gt; Value -&gt; Bool
pub topic value =
    Native.PubSub.pub topic value


{-| Not safe, use subscribe
-}
port sub : (( String, Value ) -&gt; msg) -&gt; Sub msg
</code></pre>
<p>This version of ours uses Native code, which should be going away in 0.19 but you can do this using ports as well.</p>
<p>I was against this approach for two reasons:</p>
<ol>
<li>
<p>The event becomes impure and untrackable. What happens when more than one arrive? What if none arrive? How can you be sure? How do you trace through a system once there is more than one emitter for the same Msg type? One of the great things about Elm is that it is explicit and deterministic. By introducing a global <em>external</em> message bus, I felt that we would lose all of that certainty. (metaphore:  You also cannot use a time travelling debugger anymore as these Msgs are outside of the system.</p>
</li>
<li>
<p>You lose all guarantees about ‘correctness’ of a system because the compiler no longer helps you during refactoring to work out if all Msgs are handled (are all paths are accounted for?). By having Msgs that goes into a port and forgetting to subscribe in a scenario where it’s required, bugs will invariably occur. This is a huge price to pay, why even use Elm at this point?</p>
</li>
</ol>
<p>In the end, we didn’t need it. I decoupled and rewrote the hairier bits, flattened out the architecture and most of the child -&gt; parent went away.</p>
<h4 id="jobs">Jobs</h4>
<p><em><abbr title="Commands in elm">Cmd</abbr> is the most common effect mechanism in Elm, however, you cannot use it to pass Tasks, especially curried ones. So we created our own <abbr title="Commands in elm">Cmd</abbr> wrapper, called a Job and packed it full of useful things.</em></p>
<p>Here is it’s interface:</p>
<pre><code>-- creating a job
fromHttpTask : (payload -&gt; msg) -&gt; HostToTask payload -&gt; Job msg
fromCmd : Cmd a -&gt; Job a
gotoRoute : Route -&gt; Job msg

-- combine jobs like Cmd.batch
add : Job msg -&gt; Job msg -&gt; Job msg

-- helpers to append jobs
addHttpTask : (payload -&gt; msg) -&gt; HostToTask payload -&gt; Job msg -&gt; Job msg
addRoute : Route -&gt; Job msg -&gt; Job msg
addAction : Action -&gt; Job msg -&gt; Job msg
addMsg : msg -&gt; Job msg -&gt; Job msg
addCmd : Cmd msg -&gt; Job msg -&gt; Job msg
addCmds : List (Cmd msg) -&gt; Job msg -&gt; Job msg

-- custom actions
addTitle : String -&gt; Route -&gt; Job msg -&gt; Job msg
templateRequest : HostToTask TemplateRequest -&gt; Job msg

-- like Cmd.map
map : (a -&gt; b) -&gt; Job a -&gt; Job b
noJob : a -&gt; ( a, Job msg )
</code></pre>
<p>Here are some example usages:</p>
<pre><code>-- tell the main level to change route from a child view
GotoDependant dep -&gt;
    ( model, Job.gotoRoute (Types.Route.RoutePerson personId Types.PersonSubPage.PersonDetails) )

-- tell the main level to queue up a task
SearchMembers policyId -&gt;
    ( model, Job.fromHttpTask ReceiveResults (Api.findPersons request) )    

-- get main to perform a custom action
( person
, personId
    |&gt; Types.TemplateRequestEncoder.certificate
    |&gt; Job.templateRequest
)

-- map all jobs to the same Msg
, Job.fromHttpTask ReceivePolicy (Api.getPolicy membershipId)
    |&gt; Job.add (Job.map ActivityMsg activityJob)
    |&gt; Job.add (Job.map QuotesMsg quotesJob)
</code></pre>
<p>So the main reason why we need to do this is a page ( child module ) will send off a Http request. However, the request requires information about the endpoint like JWT for auth, domain to hit etc so previously this information was passed down through every level from Main.elm to Child.elm as an extra state.</p>
<pre><code>type alias HostToTask payload =
    Host -&gt; Task Http.Error payload

fromHttpTask : (payload -&gt; msg) -&gt; HostToTask payload -&gt; Job msg

Job.fromHttpTask ReceiveResults (Api.findPersons request)

findPersons : PersonSearchRequest -&gt; Host -&gt; Task Http.Error (List PersonSearchResult)
</code></pre>
<p>Notice here that Api.findPersons is only given one argument, and thus returns <code>Host -&gt; Task ...</code>. The job therefore holds a list of these curried functions waiting for the <code>Host</code> which is on the <code>Main.elm</code> level. <code>Job.map</code> takes care of mapping these msgs up the chain much like <code>Cmd.map</code>. <code>ReceiveResults</code> is what the <code>payload</code> is mapped to when it’s successful, otherwise it’s a hardcoded error which is handled on the top level.</p>
<p>Without this requirement, we’d not need Job, the other things can be turned into Cmds because they are resolved at the child level.</p>
<h1 id="final-words">Final words</h1>
<p>Abc</p>
<h1 id="appendix-and-references">Appendix and References</h1>
<hr class="footnotes-sep">
<section class="footnotes">
<ol class="footnotes-list">
<li id="fn1" class="footnote-item"><p><a href="https://github.com/mordrax/cotwelm">https://github.com/mordrax/cotwelm</a> <a href="#fnref1" class="footnote-backref">↩︎</a> <a href="#fnref1:1" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn2" class="footnote-item"><p><a href="https://youtu.be/ulrukPRYsws?t=49m54s">https://youtu.be/ulrukPRYsws?t=49m54s</a> <a href="#fnref2" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn3" class="footnote-item"><p><a href="https://docs.google.com/presentation/d/10vN7eLr3qsd4nK2zcxUHgbu68fO_yzAA-b7Hf7kEcik/edit?usp=sharing">https://docs.google.com/presentation/d/10vN7eLr3qsd4nK2zcxUHgbu68fO_yzAA-b7Hf7kEcik/edit?usp=sharing</a> <a href="#fnref3" class="footnote-backref">↩︎</a> <a href="#fnref3:1" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn4" class="footnote-item"><p><a href="https://github.com/avh4/elm-format">https://github.com/avh4/elm-format</a> <a href="#fnref4" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn5" class="footnote-item"><p><a href="https://mordrax.github.io/cotwmtor/2016/04/09/DaySix.html">https://mordrax.github.io/cotwmtor/2016/04/09/DaySix.html</a> <a href="#fnref5" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn6" class="footnote-item"><p><a href="https://www.youtube.com/watch?v=IcgmSRJHu_8">https://www.youtube.com/watch?v=IcgmSRJHu_8</a> <a href="#fnref6" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn7" class="footnote-item"><p><a href="https://github.com/justinmimbs/elm-module-graph">https://github.com/justinmimbs/elm-module-graph</a> <a href="#fnref7" class="footnote-backref">↩︎</a> <a href="#fnref7:1" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn8" class="footnote-item"><p><a href="https://www.infoq.com/presentations/Simple-Made-Easy">https://www.infoq.com/presentations/Simple-Made-Easy</a> <a href="#fnref8" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn9" class="footnote-item"><p><a href="https://gist.github.com/zwilias/7ed394ec0e9c6035e1874d19b721e294">https://gist.github.com/zwilias/7ed394ec0e9c6035e1874d19b721e294</a> <a href="#fnref9" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn10" class="footnote-item"><p><a href="https://xkcd.com/303/">https://xkcd.com/303/</a> <a href="#fnref10" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn11" class="footnote-item"><p><a href="https://github.com/mordrax/cotwelm/blob/master/hack.sh">https://github.com/mordrax/cotwelm/blob/master/hack.sh</a> <a href="#fnref11" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn12" class="footnote-item"><p><a href="https://youtu.be/ulrukPRYsws?t=50m4s">https://youtu.be/ulrukPRYsws?t=50m4s</a> <a href="#fnref12" class="footnote-backref">↩︎</a></p>
</li>
<li id="fn13" class="footnote-item"><p><a href="https://github.com/NoRedInk/elm-decode-pipeline">https://github.com/NoRedInk/elm-decode-pipeline</a> <a href="#fnref13" class="footnote-backref">↩︎</a></p>
</li>
</ol>
</section>

