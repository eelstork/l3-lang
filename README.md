# L3 Language specification [DRAFT]

L3 is a transpective, dynamic language for multi-agent systems; a general programming language (GPL), L3 emulates key cognitive functions (narrative memory, planning, decision making, creativity) (mostly) through established, deterministic models.

L3 supports procedural, functional and objective programming.

L3 unifies behavior trees, STRIPS style planning (such as goal oriented planning), problem solving (BFS/DFS/BeFS) and evolutionary programming (via STGP).

There is a draft implementation. The implementation may be made publicized in March 2024 or later.

Transpective: remembering the past, while peering into the future.

*If you implement L3, or derive a language from this specification, or implement L3 inspired features, include a fair, explicit, prominent attribution notice in your documentation, such as "this language is inspired from L3, a programming originally designed by T.E.A de Souza".*

Sufficient information about the author(s) is available [here](AUTHORS.md)

## Who should read this?

This document is a *draft* specification; for a more engaging overview, [read here].

- If your main focus is behavior trees, [start here].
- If you are interested in GOAP, [read here].
- If you think behavior trees have nothing to do with your daily programming drudge, yet would be proven wrong, [read here].

## Execution model

In L3, a session is assumed. Within the session several L3 programs are scheduled for execution. Absent resource shortages, programs run at every tick, unless a program has requested to yield for a number of frames or a specific duration, or until an external agent (often an underlying effector) pings the L3 program.

Often, L3 programs are used for planning purposes. They drive low level effectors and run at or much below 10 Hz.

By default, recording captures every operation at every frame, along with evaluation results. Records may be RLE compressed; costly re-evaluations may be avoided through memoization.

## Units and modules

Units are associated with source files; a unit declares (membership to) a namespace and dependencies through the "using" and "not using" clauses.

A unit may pose as an external object or type bridged through the runtime, either through referencing or instantiation.

Units contain procedures, functions, and type definitions.

A module specifies namespace configuration and cross-module dependencies; modules enforce architectural integrity. Configuration may involve disabling language features, such as recording or dynamic typing.

## Literals and primitives

Conventionally, literals include integers, decimals, characters, bools (*true/false*), *null* and *void*.

*done*, *fail* and *cont* represent task statuses, whereas *maybe* and *unknown* signify uncertainty.

Primitive types (extended):

```
agent, bool, char, double, float, int, long, pro, quat, trilean, status, v2, v3, v4, v2i
```

Restricted statuses (certainties):

```
action, failure, loop, pending, impending
```

Primitive types (core):

```
bool, float, int, pro, tri, status.
```

## Statuses and tasks

Statuses may be returned either in lieu of returning nothing (void), or attached as (perhaps implied) meta-properties (when a function does return a value).

Statuses express the outcomes of a function, when thinking of the function as a task.

Restricted statuses (otherwise known as certainties) limit uncertainty regarding the status of a given task. As an example, specifying "pending" indicates a timeful task which cannot fail (the task may only return `cont` or `done`).

Additional metadata may be associated with a status; as an example, a 'cont' object may have a "delay" field indicating how long the task will take, or a success probability estimate.

Whereas a status may be an object, a task is not.

## Unknowns and probabilities

Uncertainty may be expressed through probabilities or  unknowns. The 'pro' type identifies probabilities, whereas `trilean` supersets bool and comprises the 'maybe' logical value.

Explicit unknowns and probabilities avoid confusion with statuses ("90% done is not the same as 90% likely").

There is lingering consideration for expressing unknowns via nullable values (ref); however "there is nothing here" and "I don't know what is here" are distinct situations which should not be conflated.

## Variables and typing

A variable is a name representing either a locally defined variable, or an accessible field or property.

## Variable declarations

A variable declaration implies the existence of a variable with optional typing and persistence requirements.

In lieu of type, use `var` to signify implicit typing, or `dynamic` to signify dynamic typing.

A variable may be declared inside an expression.

Variable modifiers:

`const` - specifies a variable which cannot be modified after creation

Provisional modifiers:

`ext` - specifies a variable attached to the underlying process. External variables persist between ticks and may be persisted across sessions.

`public` - specifies a variable attached to the underlying process. The host environment may read from this variable.

`mutable` - specifies a variable attached to the underlying process; the host environment may write to the variable (requires `public`).

Note: other than const, the above modifiers assume backing storage provided by the host environment. Alternatively, host side storage may be accessed through posing; a simple example of this consists in accessing a blackboard.

## Invocations (call)

A call signifies invoking a function; calls allow optionally named parameters.

The *once* modifier indicates that a call will not be invoked more than once. Without further qualification the call will be invoked once during the current session.

```
once ([after|per] [event])
```

## Operators

L3 supports common binary operators, and the ternary operator. Provisionally, binary operators may be implemented as n-ary expressions (as an example, + is implemented via the SUM n-ary operator).

The escape operators `!(exp)` and `![val](exp)` may prefix an expression `exp`; in such cases:

- if X is null, the parent function returns 'exp' or, if 'exp' is ommited the parent function emits the "fail" status.
- otherwise, the operator has no effect.

## Composite expressions

Composite expressions include blocks, selectors, sequences and activities.

Blocks define statement sequences; statements execute in order, regardless of return value, or task state.

Selectors and sequences implement behavior trees; that is, a sequence will execute until encountering failure, whereas a selector (aka "fallback operator") will execute until success is encountered.

Activities are similar to selectors and sequences; in the case of an activity, control iterates nodes until the cont state is encountered.

L3 supports ordered composites; ordered sequences, selectors and activities memorize an index pointing at the current task.

A label and description may be associated with a composite expression.

Descriptions are used for commenting; the first line in a description may not exceed N characters.
Labels may be used to dynamically replace composite expressions in a running L3 program.

## Temporal clauses and retrospective access

Temporal clauses may be used to either confirm prior outcomes or fence execution conditionally

```
did ... since ...
```


```
once since ... did ...
once per ...
```

As a parameter modifier used in conjuction with `once` or `n times`, `per ...` may be used to efficiently relate actions to specific arguments. Example

```
// Who's here?
Person p = GetPerson(nearby, seeingMe: true, friend: true);
// Greet pals once per day
Greet(p once per day);
```

Retrospective access is used to dereference past outcomes:

```
CALL{ when }
```

*when* may refer a tick, in which case a value of zero means "at the current iteration", "1" means, at the prior iteration, and so forth.

```
var current = GetPrice();
var justNow = current[0]  // same value as 'current'
var backTicked = current[1] // prior iteration
var past = current{ 10 days ago };
var delta = past - current
Log("That's {delta} less than days ago, what a deal!")
```

## Functions

A function declaration consists in a name, optionally typed parameters and an optional via clause. The auto modifier specifies an auto function. L3 supports overloading and default parameter values. Default parameter values are not limited to primitive types.

Functions may define additional blocks;

### Steady tasks

Often, a task requires working memory. In this cases we need the task to exist across frames. While this may be achieved through delegating to consolidated task objects, this approach may be tedious.

A *steady* function reiterates across frames, assuming a match among predecessors.

```
steady func(A, B, C) where COND(X, Y) until{COND}{
    init{}
    step{}
    exit{}
}
```

The *where* clause specifies a comparison between X (a predecessor, identified by its arguments) and Y (the incoming call). This clause allows deciding whether a new invocation of 'func', f' is a continuation of a prior invocation f.

The `init{}` clause may be used to perform explicit initialization when entering a steady function; thereafter step{} is repeatedly invoked. Finally upon moving away from a steady function, exit{} is called
Do not use step{} if both init and exit are omitted.

When init{} is not used, a variable declaration will skip after the first invocation; instead,

Finally the `until()` clause specifies a discard condition for the steady function.

NOTE: the correct approach to steady tasks consists in duly considering the alternative, and deciding whether there is a strong case for the language feature, in one form or another. Without getting into the details, the alternative is looking something like:

```
steady func(A, B, C){
    // build signature from "here" (that is, the current
    // stack).
    // match signature to proc storage, retrieve state
    // if applicable.
    // otherwise create proc-state, with correct signature.
}
```

Looking at the above, it feels like the use case is when we're doing just a perfect match, or simply ignoring specific arguments, while the "where" clause may be pedantic. And yet... `steady` may be what we need to clearly signal that a function uses working memory... in which case the stack object may be denied, unless steady is used.

While handing the stack does not feel wholesome, more investigation is needed to conclusively decide whether checking upstream is an anti-pattern (conventional view) or a key feature to help solving actual problems.

### Collaborations

When a function is declared using a 'via' clause the body of the function must be omitted and the function then provides an attachment point for async procedure calls (APCs); example:

```
// in Shopkeeper
Sleep() || Sell();
void Sell() via Purchase;

float RequestPrice(string item){ ... }

// Elsewhere...
void Purchase(agent shopkeeper){
    var p = shopkeeper.RequestPrice();
}
```

In this case `agent` refers an external process (relative to the process invoking the Purchase() function). Upon calling RequestPrice(), either of the following outcomes will occur:

- If Sell() is being traversed, RequestPrice() is called and the expected value is returned.
- Otherwise, RequestPrice() will return a failing status.

In practice a successful APC usually requires at least two ticks:
(1) `t[ i ]` - RequestPrice() is posted as a message binding the `Purchase` channel. If Sell() was traversed at the prior tick, the message is set (not queued); at this stage RequestPrice() returns `cont`.
(2) `t[ i | i' ]` - RequestPrice() runs "in context" (as part of the shopkeeper process).
(3) `t[ i' ]` - Upon reiterating RequestPrice(), the return value is fetched and returned to the caller (inside the customer process)

Collaborations may use RPCs on the back end side, however they should not be confused with RPC. Collaborations are designed to orchestrate commmunication between responsive agents who may be task switching, and may unexpectedly disengage communication.

As a key benefit, collaborations allow modeling non binding multi-agent interactions 'in third person'.

### Auto functions and planning

The auto modifier specifies an auto-function.

The outcome of an autofunction is resolved dynamically through search (problem solving); a search will iterate and invoke public members, using available parameters as inputs and/or combining parameters using friend operators.

Vendor/plugin specific implementations may leverage additional resources (such as LLMs or dedicated solvers) to optimize searches.

Absent other constraints, the search returns when an object matching the specified return type is discovered.

The "progressive" keyword may be used to balance-load auto-function execution. Then, if a result is not discovered right away, the function may take several frames to return (in the meantime, 'cont' is returned).

When an autofunction specifies a body, the body of the autofunction predicates the goal and the "out" keyword identifies candidate output.

Example: finding the length of a string:

```
// returns a hashcode... probably
auto int Count(string arg);

// returns the length of the string... probably
auto int Count(string arg) => out > 0;
```

With auto functions, model entities precede actual counterparts; for instance, a function may define a predictive 'mod' section.

Through model objects, an auto function evaluates a plan and discovers a model solution. After the planning phase has completed, the auto-function will extract the path and start executing the plan; whereas actual evaluations do not reiterate, actual evaluations then replace model evaluations.

Auto-functions are non special with respect to task evaluation; in particular, timely evaluations yielding `cont` will interrupt evaluation. Then, upon restoring control to the process, the auto-function will re-evaluate, and replanning will occur. Memoization and/or aliasing may be used to avoid replanning.

When using auto functions, it is assumed evaluation must use all provided parameters; the "opt" keyword specifies optional parameters (arguments which auto-function resolution may ignore)

Where planning is involved, auto functions generate additional notifications and intercepts (see "notifications").

[Provisional] Where 'traverse' is used, an auto function will traverse the implied graph (depth first); via `enter` and `exit` blocks, autofunctions implement the visitor pattern.

## Classes, instances and interfaces

In L3, a class is defined as having fields, properties, constructors and methods. An instance is an object, which has a type. Classes may have constructors; fields may have default initializers.

Interfaces declare properties and methods; an interface may also define methods, through leveraging the defined properties.

## Decorators

Decorators apply to expressions (including function calls), either as prefix or postfix operations.

In general, decorators are presented as function calls; a prefix decorator may use (thereby forcing resolution of) the parameters to the client function it is associated with, whereas a postfix decorator may use parameters to the client function, as well as the return value (via the 'out' keyword).

A guarding decorator will cancel the associate expression. In this case intercepts do not run, and the associate expression is ignored (the cancellation itself may generate a notification)

A value returned by a postfix decorator replaces the output of the associate function.

## Events

L3 defines categorical (class) and instance notifications. Instance notifications are registered to objects, whereas categorical notifications bind all instances of a given type.

### Immediate notifications and intercepts

Immediate notifications signal entering/exiting a function or labeled expression:

```
on enter|exit funcname([obj], args) {}
```

### Progressive notifications

Progressive notifications signal entering/exiting a task:

```
on start [task] { ... }
on started [task] { ... }
on [task] (done|failed|interrupted) { ... }
```

Progressive notifications occur when an agent apply themselves to one same task over time.

If a task was not traversing, then traverses at the next tick, the "start/started" notifications are emitted. Whereas, if a task was traversing, then does not traverse at the next tick, the done/failed/interrupted notifications are messaged.

Progressive task events are recorded.

### Prospective notifications

Prospective notifications are emitted when an auto function generates a plan. In this case a prospective notification signals an intended course of action.

Iteratively, agents may use prospective notifications to refine individual plans; this approach may be used either in adversarial or collaborative contexts.

## Appendix A - Required features

### Post-casting

Cast a type to an interface, either at compile time, or runtime, without the type itself explicitly declaring membership to the interface.

Reason why this is useful: because it is often the case that a developer "see" a useful interface after third party code has shipped.

In typical cases a developer simply want to reuse a design which exists in a library, but has not been formalized through an interface. The result is having to switch/wrap the original types, which is either ugly, or wasteful.

### Existential and potential clauses

Existential clauses mandate or negate the existence of objects, function calls, members, namespaces and types.

Within the runtime, existential clauses avoid the hassle of having to
- manually maintain mirrored graphs
- inline notifications to associate calls with other calls in a systematic manner.

At compile time, existential clauses extend the concept of interface:
- Through interfaces, programmers adopt requirements; however stubbing interfaces is a job IDEs are taking care of, in the best case.
- Good design often systematically associates types, without the type associations being formally mandated; as such good design is synonymous with drudgery, which is both time consuming and error prone.
- Therefore options are needed through which existential clauses mandate the existence of types though specified nomenclatures; mechanical resolution is required to materialize types and members, as opposed to (well, in addition of) compilers grumbling when things are missing.
- The above requirement implies annotating missing implementations, so that machine agents can quickly walk developers through "fill in" sessions.

Needless to say, MVC and its many avatars provide an example of how architectural patterns "work" for software, but contour an automation void which needs to be filled.

Another example of drudgery around both static and type associations is seen in the runtime/elements dissociation in the L3 implementation itself:
- The design is articulate and regular. Sure, we reserve the option to make exceptions, only proving that "there is a rule".
- There is no lightweight solution to aligning types at runtime (using, of all things, a switch).

Negative clauses are relatively straightforward, it's about simply just saying "don't do this". At runtime negative clauses can be combined with positive clauses to refine mirror graphs.

Potential clauses help extending libraries without mandating extensions. They may be useful to keep balls rolling, but also not viewed as a necessity.

### Rethink the meaning of "static"

Static is essentially a good idea gone wrong, with developers often going with "masking" strategies through patterns (such as the singleton) which tend to onboard the bad, mentors ignoring the appeal, and language design altogether sitting on its thumbs.

### Parametric loops

Parametric loops cover parameter ranges within function calls, applying the "content first" principle to domain traversals.
(Similar to range attributes in NUnit)
Parametric ranges may be explicit (such as "(0, n)") or implicitly determined through passing collections in lieu of singletons.

```
// implied range (friends is a collection)
Send(newYearGreetingCard, friends);  
// With a list of lists
Sort(lists); // sorting the list
// Sort each list, not the parent list
Sort(all lists)
```

## Appendix A - Attributes

[memoize] - applied to a function (?or a class), indicates that the target operates on a "same input, same output" basis and is suitable for memoization.

[alias n.n] - applied to a function (? or a class) indicates that the target will tolerate a lag up to n.n time units, assuming **same-looking** input.

## Appendix B - Embodiments

At the time of writing, there is no plan to implement an L3 parser. L3 productions are represented in XML. Until a stable version of the language is availed, focusing on the AST without writing a parser does help refining the language without committing to a specific syntax.

L3 is primarily designed as an interpreted, dynamic language; with certain restrictions, transpiling to other languages (such as C++, Python or C#) will be possible.

Availing L3 as a dynamic language allows machine agents to generate L3 productions at runtime, notably in the context of STGP (strong typed genetic programming) and higher HOTGP (higher order, typed genetic programming).
