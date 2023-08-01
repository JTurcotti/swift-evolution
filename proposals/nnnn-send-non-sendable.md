# Send Non-Sendable

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Joshua Turcotti](https://github.com/jturcotti)
* Review Manager: TBD
* Status: **Awaiting implementation** or **Awaiting review**
* Vision: *if applicable* [Vision Name](https://github.com/apple/swift-evolution/visions/NNNNN.md)
* Roadmap: *if applicable* [Roadmap Name](https://forums.swift.org/...))
* Bug: *if applicable* [apple/swift#NNNNN](https://github.com/apple/swift/issues/NNNNN)
* Implementation: available on public Github: [SendNonSendable.cpp](https://github.com/apple/swift/blob/main/lib/SILOptimizer/Mandatory/SendNonSendable.cpp)
* Upcoming Feature Flag: `SendNonSendable`
* Review: ([pitch](https://forums.swift.org/...))

## Introduction

Swift Concurrency splits memory into the "isolation domains" of various actors and tasks. Computations isolated to distinct domains can execute concurrently, so to prevent data races it is vital that no mutable state is simultaneously accessible from multiple domains. The Swift type system ensures this separation property by allowing only references to deeply immutable values to be communicated between isolation domains - such values conform to the `Sendable` protocol. 

Unfortunately, requiring `Sendable` conformance for all values communicated between isolation domains is very restrictive in practice. For example, mutable objects cannot be constructed by one actor and then sent to another, even if they are never accessed except to be constructed then sent. A flow-sensitive analysis that determines whether values of arbitrary type can be safely without introduing data races sent would allow this pattern, and thus provide a large increase in the expressivity of Swift concurrency.

The `SendNonSendable` pass, currently available as an experimental feature, implements a flow-sensitive analysis that tracks non-`Sendable` values, allowing them to be sent between isolation domains but marking them as "consumed" when they are. This ensures that any potential data races resulting from concurrent access in the sending and receiving domains are avoided. The crux of this analysis is grouping values that could alias or reference each other into static "regions". Sending a value should consume the value itself and all other aliasing or referencing values in its region, as accessing those values after the send could also yield a race.

This pass allows for greater flexibility of programming with Swift concurrency, without sacrificing data-race freedom or ergonomics.

## Motivation

Consider simple code like this:

```swift
// an object that can be displayed to the user - not Sendable
class DataView { ... }

// a function that displays the passed object
@MainActor func addToDisplay(dataView : DataView) { ... }

// a function that compute a view of passed data and displays it
@MainActor func displayData(data : Data) {
  let builtView : DataView = buildView(data : data)
  addToDisplay(builtView)
}
```

`addToDisplay` displays a `DataView` object to the user, a UI action, so should be main actor isolated. The function `displayData` that builds up a `DataView` and passes it to `addToDisplay` must also be main actor isolated because `DataView` is not `Sendable`, so could not be passed from outside the main actor into it. This is functionally correct, but could have very poor performance if `buildView` is an expensive operation - as it will be forced to run on the main actor. With current Swift concurrency, there is no safe solution that allows the `DataView` to be constructed off the main actor, and displayed by the main actor, and thus an unnecessary tradeoff between performance and safety exists.

## Proposed solution

Ideally, the function `displayData` would not have to be main actor isolated. The expensive `buildView` operation could happen asynchronously on another actor, and the main actor could be invoked only when the `DataView` has been built and is ready to be displayed, such as in the following code:

```swift
func displayData(data : Data) async {
  let builtView : DataView = buildView(data : data)
  
  await MainActor.run {
    addToDisplay(builtView)
  }
}
```

Because `MainActor.run` currently takes a `@Sendable` closure, this code would not be allowed; the passed closure cannot capture `builtView` of non-sendable type. The `SendNonSendable` pass allows this code by allowing `MainActor.run` to take any closure, including non-sendable ones, but ensuring that any values captured in that closure are rendered inaccessible after the isolation-crossing invocation of `MainActor.run`. For example, the following code would produce diagnostics:

```swift
func displayData(data : Data) async {
  let builtView : DataView = buildView(data : data)
  
  await MainActor.run {
    addToDisplay(builtView)
  }
  
  builtView.updateContents() // error: access here could race
}
```

There are more subtle cases in which races could arise as well, in which the value being access is not exactly one captured by a closure that crosses isolation domains:

```swift
func displayData(data : Data) async {
  let builtView : DataView = buildView(data : data)
  
  listOfViews.append(builtView)
  
  let bestView = listOfViews.max(by: { $0.value > $1.value })
  
  await MainActor.run {
    addToDisplay(builtView)
  }
  
  bestView.updateContents() // error: access here could race
}
```

Here, accessing `bestView` after the invocation of `MainActor.run` is still a potential race, as `bestView` could alias `builtView`. Thus it is necessary not just to mark `builtView` as consumed after the invocation - but a whole *region* of values including `listOfViews` and `bestViews` that could potentially alias or reference `builtView`. The job of the `SendNonSendable` pass is to track aliasing and other references between values in scope so that they may be partitioned into regions. At any program point, the `SendNonSendable` pass provides this partitioning. This allows safe sending of non-sendable values between isolation domains because it is statically known which values must be marked inaccessible after the send - all other values in its region.

 Thus the job of the pass is, in three parts:

- track which **non-sendable values** belong to which regions
  - For example, track that `builtView`, `listOfViews` and `bestView` are all in the same region above
- mark whole regions of **non-sendable values** as consumed when any of their member values are sent between isolation domains
  - For example, mark that the region containing `builtView` is consumed after `MainActor.run` above
- prevent access to any **non-sendable values** in consumed regions.
  - For example, prevent the access to `bestView` above because that value is in a consumed region

Note the emphasis passed on **non-sendable values** - values of sendable type are ignored by this pass and subject to typechecking exactly as currently implemented.

## Detailed design

This proposal does not add new syntax to Swift, nor change any types. The only change it makes to existing behavior is to prevent the emission of the `non_sendable_call_argument` diagnostic that is emitted when a call that is determined to cross isolation domains has non-sendable arguments. Preempting this diagnostic is what allows non-sendable values to now be passed across isolation domains, such as from a nonisolated context to a main actor isolated context as shown above. 

Instead of emitting the `non_sendable_call_argument` diagnostic, the `TypeCheckConcurrency` Sema pass now decorates `ApplyExpr`s with whether they represent an application that crosses isolations. The `SendNonSendable` mandatory SIL pass is then responsible for emitting diagnostics around the subset of isolation-crossing applications that are could yield races, but only that subset. 

The key question then becomes: how does the SIL pass determine which isolation-crossing applications could yield races? 

### <a name="regionrules"></a> At a high level: simple rules for regions

All non-sendable values within a function body are tracked to determine what region they reside in. Some simple rules regarding value initialization:

- If the initializer for a non-sendable value takes any non-sendable arguments, then the regions of all its arguments will be merged together so that they all occupy a single, shared region, after the initializer returns. The initialized value will also reside in that region.
-  If the initializer is passed no arguments, or is passed only sendable arguments, then a fresh region shared with no existing values will be created for the initialized value. 

It is important to keep in mind that these regions are a purely static abstraction, so allocating fresh regions means only choosing a new identifier for that region, not performing a dynamic allocation of any sort. The following code points out region-creation behavior.

```swift
class NonSendable {
	var x : SendableType
  var y : OtherNonSendableType?
  
  init(_ x : SendableType, _ y : OtherNonSendableType? = none) {
    self.x = x; self.y = y
  }
}

// starting region partition: none

let x = SendableType() // x does not have a region because it is sendable
let y = OtherNonSendableType() // y gets a fresh region
let ns1 = NonSendable(x) // ns1 gets a fresh region
let ns2 = NonSendable(x, y) // ns2 gets the same region as y

// ending region partition: {ns1}, {y, ns2}
```

Functions in general actually exhibit the same semantics: non-sendable arguments are merged into a common region, and non-sendable results come from that region or a fresh one in the case of no non-sendable arguments. This is necessary because, without further information, we have to assume the called function creates references between its passed non-sendable arguments.

```swift
func compareBoxContents(_ box0 : Box, _ box1 : Box) {
  if (box0.contents == box1.contents)
  	print("the same")
  else
	  print("different")
}

func reassignBoxContents(_ box0 : Box, _ box1 : Box) {
  box0.contents = Contents()
  box1.contents = box0.contents
}

// upon initialization, box0 and box1 occupy separate regions
let (box0, box1) = Box(), Box()

compareBoxContents(box0, box1) // this call merges the regions of box0 and box1

// this consumes the region of box0
Task {
  box0.contents.increment()
}

// this is an error, because the region of box1 was consumed
// (same region as box0)
Task {
  box1.contents.increment()
}
```

This function convention is illustrated above. Though the above code is actually perfectly safe, as the call to `compareBoxContents` does NOT introduce aliasing or referencing between `box0` and `box1`, the function convention has to be safe with respect to implementations like `reassignBoxContents`, that have the same static type as `compareBoxContents`, but DO introduce aliasing between the contents of `box0` and `box1`. If  `reassignBoxContents` were emplaced in the above code, then the creation of the two tasks would indeed yield a data race, so the code must be marked unsafe.

### Summary : simple rules for regions

In summary, regions are "created" when non-sendable values such as classes are intialized without any non-sendable arguments. Regions expand in the following ways:

- `let y = x.f`: reading a non-sendable-typed field of `x` yields a value `y` in the same region as `x`
- `let y = x`: creating an alias `y` yields a value in the same region as `x`
- `y.f = x`: creating a reference from a non-sendable value `y` to a non-sendable value `x` merges their regions
- `z = { ... x ... y ...}`: capturing non-sendable values `x` and `y` in a closure merges their regions, and yields a new closure `z` in the same region as `x` and `y`
- `if/for/while/switch` (control flow): if `x` and `y` are in the same region in any predecessor basic block of a function, then they will be merged into the same region in all successors of that basic block

The result of this is that at the point of a send of a non-sendable value between isolation domains, all other values that could potentially be aliased or referenced from/by it, even along just one control flow path to the send, will be known to be in the same region as it, and will be marked "consumed" and be rendered inaccessible after the send.

### Crossing Isolation Domains

The function conventions outlined above apply to function calls that will execute synchonrously in the same isolation domain as the caller. Calls that can cross isolation domains have slightly different semantics: instead of just being merged, the regions of all non-sendable arguments to a cross-isolation call are *consumed* as well. The reason this is necessary is different for the two types of calls that pass values across isolation domains.

#### Calls into actors

When a non-sendable value is passed to an actor-entering call, i.e. a call to an actor method from any isolation except that actor's own isolation, the value could "escape" into the storage of that actor. So even after the (necessarily async) call returns, the actor could process a new request concurrently with the caller executuing more code. Since the originally passed value is now accesssible to the actor through its storage, the caller must be banned from accessing it to prevent a data race. Thus actor-entering calls must consume their arguments. The following code illustrates this:

```swift
actor NationalPark {
	var visitors : [Visitor]
  
  func admitVisitor(_ visitor : Visitor) {
    visitors.append(visitor)
  }
  
  func admitAndGreetVisitor(_ visitor : Visitor) {
    // this call does not cross isolations,
    // so it does not consume `visitor`, it just merges its region
    // with the region of `self`
    admitVisitor(visitor) // callsite 1 - safe
    
    // so access to `visitor` is permitted
    visitor.greet()
  }
}

func visitParks(_ parks : [NationalPark], _ visitorName : String) {
	let visitor = Visitor(visitorName)
  
  for park in parks
  	// this call enters the `park` actor, so it
    // consumes `visitor`
    // accessing `visitor` in a loop is thus an error
  	await park.admitVisitor(visitor) // callsite 2 - ERROR
}
```

Note that the call to `admitVisitor` within the actor method `admitAndGreetVisitor` labelled `callsite 1` does NOT consume its argument, and continued access after the callsite is permitted. This is because that call is NOT a cross-actor call, so there is no risk of another actor holding a reference to the argument.

On the other hand, the call to `admitVisitor` in the nonisolated function `visitParks` labelled `callsite 2` enters an actor, so it DOES consume its argument. This makes the written code unsafe, and indeed it yields an error. If the code were allowed, then multiple actors could concurrently reference the same `visitor` object from their storage, racing on it. 

#### Task creation

Special functions that create new tasks, such as `Task.init` or `Task.detached`, or functions that otherwise cause passed closures to execute concurrently with the caller such as `MainActor.run`, must also consume their argument. In particular, the argument will be a closure, so any values captured in the closure will be in the same region as that closure, and consuming it will consume those values. This consumption prevents races on those values between the caller and the executor of the passed closure. This prevents the following possible racy code from being expressible as well:

```swift
func visitParksParallel(_ parks : [NationalPark], _ visitorName : String) {
  let visitor = Visitor(visitorName)
  
  for park in parks
  	Task {
      await park.admitVisitor(visitor) // will error - closure creation consumes `visitor`
    }
}
```

User-defined functions that takes non-sendable closures should not, in general, exhibit isolation-crossing semantics and consume their arguments, so it will be necessary to inform the `SendNonSendable` analysis of the special functions such as `Task.init`, `Task.detached`, and `MainActor.run` that execute their passed closure under different isolation than or concurrently with the caller. Two possible approaches for this are:

1. Hard-code a list of such functions - possible but likely considered bad practice
2. Mark such functions with an annotation such as `@IsolationCrossing` at the source level where they're defined. This is likely the better-practice approach but could potentially lead to API/ABI breaks.

Additionally, to achieve the desired level of expressivity, it will be necessary to *remove* the source-level `@Sendable` annotation on the types of the arguments to these functions. Without the introduction of the `SendNonSendable` pass, race-freedom is attained for functions such as `Task.detached` only by ensuring they only take `@Sendable` args. This prevents entirely the passing of closures that capture non-sendable values to these functions (as such closures are necessarily `@Sendable`). By implementing the `SendNonSendable` pass, sendable closures will still be able to be passed freely to these functions, but non-sendable closures will not be outright banned - rather they will be subject to the same flow-sensitive region-base checking as all other non-sendable values passed to isolation-crossing calls: if their region has not been consumed the call will be allowed, and otherwise it will throw an error. This change (removing `@Sendable` from the argument signatures of these functions) is necessary, but unfortunately may be an API/ABI break.

### Diagnostics

The easiest way to generate diagnostics for the `SendNonSendable` pass would be to note each code site at which a value is accessed, but is known by the pass to be in a consumed region. These diagnostics would read something like:

```swift
func giveBoxToActor(a : MyActor) {
  let b = Box()
  
  a.accessBox(b)
  
  if (b.contents >= 7) { // warning: value of non-sendable type `Box` accessed here, but could've been sent to concurrently executing code above, yielding a potential raace
		....
}
```

Entirely correct semantics for this pass could be implemented with this diagnostic style, i.e. diagnostics could be thrown iff the pass discovers an error, but this style of diagnostics is potentially confusing to programmers. Although it is not the site at which the error is discovered, the site at which the isolation-crossing send actually took place is likely a much more logical place to put the warning. This is likely because the calls that cross isolations are trivially known to be sites at which concurrency, and concurrent communication, comes into play. The sites at which values in consumed regions are accessed, however, could be any valid AST node in the Swift language. At a high level, it seems reasonable to expect programmers to think the most carefully about the values being sent at the points concurrent communciation is performed. Thus, although a race arises from the combination of a value sent at such a point with a value in the smae region accessed later in the function, highlighting the point at which the value sent allows the programmer to continue to focus their debugging efforts largely on the points at which concurrency communication is performed. In line with this reasoning, the `SendNonSendable` implementation instead emits diagnostics like the following:

```swift
func giveBoxToActor(a : MyActor) {
  let b = Box()
  
  a.accessBox(b) // warning: passing argument of non-sendable type 'Box' from nonisolated context to actor-isolated context at this call site could yield a race with accesses later in this function (1 access site displayed)
  
  if (b.contents >= 7) { // note: access here could race
		....
}
```

If there are multiple sites at which values in the region of the sent value are accessed, they will all be displayed:

```swift
func compareListElems(a : MyActor) {
  let myList : [NonSendableElem] = genList()
  
  let elem0 = myList.min(by : {$0 < $1})
  let elem1 = myList.max(by : {$0 < $1})
  
  a.displayElem(elem0) // warning: passing argument of non-sendable type 'NonSendableElem' from nonisolated context to actor-isolated context at this call site could yield a race with accesses later in this function (2 access sites displayed)
  
	print("min: \(elem0)") // note: access here could race
  print("max: \(elem1)") // note: access here could race
}
```

There is one other category of diagnostic emitted by this pass. As described in the [section above](#regionrules), the default function convention assumes that functions do not consume the regions of their arguments (including `self`). Thus making an isolation-crossing call passing any non-sendable values in the same region as a non-sendable `self` value, or any non-sendable args, will be an error.

<a name="passtoactor"></a>

```swift
func passToActor(a : MyActor, v : NonSendableValue) {
  a.foo(v) // warning: call site passes `self` or a non-sendable argument of this function to another thread, potentially yielding a race with the caller
}
```

As the diagnostic message indicates, this warning is necessary because function conventions allow code that continues using non-sendable arguments after they are passed to a non-isolation-crossing call.

```swift
func genAndPassTo(a : MyActor) {
  let v = NonSendableValue()
  
  // call does NOT consume v
  passToActor(a, v)
  
  // access here allowed
  print(v)
}
```

To allow the function `passToActor` above to typecheck, a `consuming`-style annotation is needed. See the section [Consuming args](#consumingargs) below.

### At an implementation level

`SendNonSendable` is implemented as a raw SIL pass. The SIL pass performs a flow-sensitive, intraprocedural analysis for each function body that assigns a partition to the non-sendable values in scope at each point in its body. This partition groups the values into regions, with some values grouped into the special "consumed" region indicating that their region was consumed by an isolation-crossing application and can no longer be accessed. Only values that have been initialized by a given program point are tracked in the partition for that program point. Each SIL instruction is mapped to an operation on these partitions that manipulates the set of tracked values and their assignments to regions. Each operation takes arguments, whose values are not directly SILValues but rather a parallel set of values referred to in the code as `TrackableSILValue`s. The `SendNonSendable` pass maps each SILValue to a `TrackableSILValue`, often mapping two SILValues to the same `TrackableSILValue` if one is a projection of the other or refers to the same underlying storage. To indicate this layer of indirection `%%` (double percentage signs) are used to indicate `TrackableSILValue`s.

There are 5 operations on partitions that source SILInstructions get translated to:

- `Assign %%0 = %%1`
  Before this operation, `%%1` must already be tracked by the partition, and assigned to a non-consumed region. `%%0` need not be tracked or non-consumed if it is tracked. After this operation, `%%1`'s assignment in the partition will not change, and `%%0` will be tracked and assigned to the same region as %%1.

- `AssignFresh %%0`
  Before this operation, `%%0` need not be tracked or non-consumed if it is tracked. After this operation, `%%0` will be assigned to a region that no other values in the partition are assigned to.

- `Consume %%0`
  Before this operation, `%%0` must be tracked by the partition but may or may not be consumed. After this operation, `%%0` will be marked as consumed in the partition. If `%%0` was assigned to a non-consumed region before the operation, then all other values in that region will also be marked as consumed after the operation.

- `Merge %%0 with %%1`
  Before this operation, both `%%0` and `%%1` must be tracked and assigned to non-consumed regions. After this operation, all values in both of those regions will be assigned to a single, non-consumed region.

- `Require %%0`
  Before this operation, `%%0` must be tracked and assigned to a non-consumed region. This operation has no effect on the partition.

TODO: add examples of SILInstructions that translate to each PartititonOp

At entry to a function body, `self` (if non-sendable) and all non-sendable arguments are assigned to a single, non-consumed region. Using fixpoint iteration, partitions are then computed at entry and exit to each basic block of the function that satisfy two properties:

1. Applying all operations of a basic block, in order, to its entry partition yields its exit partition
2. Each entry partition is the "join" of the exit partitions of each predecessor to its basic basic block. The join of a set of partitions is the finest partition in which any two values that are assigned to the same region in some partition in the set are assigned to the same region, and all values that are consumed in some partition in the set are consumed.

Once these partitions are computed, the preconditions of each operations can be checked against them, and diagnostics are reported if they do not hold.

## Source compatibility

All previously valid, well-typed Swift code will still be valid, well-typed Swift code. 

## ABI compatibility

No currently planned ABI changes, except as indicated above to potentially change the signatures of functions like `MainActor.run`. In the future, we could possibly have to add more information to function signatures to support more general operations on the region partition, such as ensuring that result values from functions come from fresh regions or the regions of arguments; or allowing two arguments to a class method to come from a region distinct from `self` but the same as each other.

## Implications on adoption

The `SendNonSendable` pass, if adopted, would encourage the development of libraries that pervasively use Swift concurrency, but do not enforce sendability on all types communicated between isolation domains. This is a large expressivity win, but could fundamentally change the way libraries are written, and this have a large impact if the feature were adopted then rolled back. 

## Future directions

### <a name="consumingargs"></a>Consuming args

In the current implementation of `SendNonSendable`, functions cannot consume their arguments. This greatly constricts allowed programming patterns. To send a value to another isolation domain, that value *must* have been initialized locally, not read from an argument or from `self`'s storage. Allowing values from `self`'s storage to be safely sent to other threads is the subject of the `iso` fields extension (discussed below)[#iso] and is a bit involved, but allowing arguments to be sent (i.e. consumed), is much simpler. All that is necessary is to add a `consuming` annotation to function signatures that indicates that certain arguments could be consumed by the end of the function body. As a simplest example, the following code shows the behavior of the `consuming` annotation:

```swift
func passToActorConsuming(a : MyActor, consuming v : NonSendableValue) {
  a.foo(v)
}

func genAndPassToConsuming(a : MyActor) {
  let v = NonSendableValue()
  
  // call consumes v
  passToActorConsuming(a, v)
  
  // access here NOT allowed
  print(v)
}
```

Unlike the prior function `passToActor` (defined above)[#passtoactor], `passToActorConsuming` *does* typecheck now, but `genAndPassToConsuming` does not - the inverse situation of the non-consuming functions.

`consuming` parameters are a very natural programming pattern. Without them, it is only possible for non-sendable values to ever make a single hop between domains. They are also very easy to implement, and might even be done so before the acceptance of this proposal. The largest difficulty with the addition of this feature is ergonomic interplay with the existing `consuming` annotation that exists in the Swift language. The existing `consuming` keyword focuses on non-copyable types - and specifies ownership conventions for handling refcounts and deallocation. This is related to the idea of `consuming` needed by `SendNonSendable`(let's call it `region-consuming` for now), and in fact, any case in which a parameter is `region-consuming`, it should also be `consuming` (in the existing Swift, non-copyable, sense). Unfortunately, the converse does not hold. For example, parameters to initializers and setters are usually `consuming`, but they should not be `region-consuming`. The reason for this is that the region-based typechecking of `SendNonSendable` is able to track the fact that by passing values to a setter or initializer, ownership has been transferred, but to a known target: the region of `self` of the called method. Thus by not marking such parameters as `region-consuming`, they can still be used after being passed to a setter or initializer as long as the set or initialized value is not consumed. If all such methods' parameters were `region-consuming`, then even if `self` were not consumed, the arguments to the methods would not be accessible after the call.

It is worth noting that for any methods meant to be called only in isolation-crossing contexts, it is a strict gain of expressivity to mark their non-sendable arguments as `consuming`; at callsites, consumption is enforced anyways, and within the function body, the freedom to consume the arguments again by passing to a third isolation domain would be attained. This provides further evidence that exposing it as an annotation would be useful.

To concretely illustrate the semantics of this extension, `consuming` parameters would:

- consume the regions of their non-sendable arguments at callsites *whether or not* the callsite is isolation-crossing (in contrast to non-`consuming` parameters that only consume their arguments' regions if the callsite is isolation-crossing)
- be allocated separate regions from `self` and all other arguments in the region partition used at the entry to function bodies.

It is of note that there is technically an even more general approach that expands on the second point of semantics above by allowing two `consuming` parameters to be assumed to come from the same region as each other. Without this feature, `consuming` parameters would always have to be passed values known to be in a separate region from all other arguments at the callsite. With this feature, two values that are possibly aliases of each other, or possibly reference each other, could be passed to two `consuming` arguments. The benefits of this further extension are less concrete that `consuming` itself, so it is not likely this will be introduced soon.

### Returning fresh

Another way that function signatures could be made more expressive is through their results. In the current `SendNonSendable`, there is no way to return non-sendable values from an isolation-crossing function. In fact, the diagnostics produced for obtaining non-sendable results across isolation boundaries that arise from `TypecheckConcurrency` are not even suppressed. This makes code such as the following impossible:

```swift
func generateFreshPerson(_ name : String, _ age : Int, _ ancestryMap : AncestryMap) -> Person {
  let person = Person(name, age)
  if (ancestryMap.containsChild(person)) { ... /* do some logic */ }
  
  return person
}
```

This code basically wraps an initializer with extra logic, and aims to return a non-sendable result to the caller in a fresh region. Unfortunately, the current `SendNonSendable` pass does not allow its result to be used. A useful extension would allow functions with non-sendable results to allow those results to be accessible to cross-isolation callers in the following cases:

- For methods that are not actor methods, the result of isolation-crossing calls to those methods are always available to the caller. 
  - By default, results are provided in the same region as `self` and any other arguments (except `consuming` arguments (see above)[#consumingargs])
  - If the `fresh` keyword is placed on the result type in the function signature (e.g. `func generateFreshPerson(...) -> fresh Person`), then the result is provided in a fresh region
- For actor methods:
  -  If the result is not annotated with the `fresh` keyword, then it is never accessible to cross-isolation callers, as the result is in the same region as actor-isolated storage, which is not accessible to the caller.'
  - If the result is annotated with `fresh`, then it is provided to the caller in a fresh region just as for non-actor methods.

Running the `SendNonSendable` pass now involved also ensuring that any values used as `fresh` non-sendable results do indeed come from a region distinct from the region of self and any args.

### <a name="iso"></a>`iso` fields

In the system outlined up to this point, a notable weak point lies in that regions tend to grow very large. In fact, any data structures, including actors and all their stroage, will comprise a single region. As shown in the rest of this proposal, this still allows a large variety of concurrent programming patterns currently impossible in Swift, but it also bans many more. For example, it prevents actors from sending parts of their state to other actors and replacing it with fresh values, such as the following:

```swift
actor IndecisiveBox {
  var contents : NonSendable
  
  func setContents(_ val : NonSendable) {
    contents = val
  }
  
  func swapWithOtherBox(_ otherBox : IndecisiveBox) async {
		let otherContents = await otherBox.contents
    await otherBox.setContents(contents) // warning: call site passes `self` or a non-sendable argument of this function to another thread, potentially yielding a race with the caller
    contents = otherContents
  }
}
```

This could be a perfectly safe pattern, but in the current `SendNonSendable` pass will produce diagnostics as shown above. The issue is that it is not statically known that it is safe to send `contents` to another thread while continuing to access the rest of the actor's storage. To this end, the system could introduce the `iso` keyword. 

TODO: expand on this section.

### Async let, and task completion

## Alternatives considered

### Leaving sendability requirements as is

### Relying on move-only (i.e. purely linear) types

### Non-inferred regions

### A finer capabilities lattice

## Acknowledgments

This proposal is based on joint work with Mae Milano and Andrew Myers, published in the PLDI 2022 paper "A Flexible Type System for Fearless Concurrency". Doug Gregor and Kavon Farvardin assisted with the development of the `SendNonSendable` implementation and this proposal as well.
