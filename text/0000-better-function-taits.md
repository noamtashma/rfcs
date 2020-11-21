- Feature Name: `better_function_traits`
- Start Date: 2020-11-20
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

TODO: points to write in this documents:
* this suggestion is simpler, since it unifies the traits
* this suggestion paves the way for better integration for general self-argument-through-a-smart-pointer,
such as `Fn(Pin<Self>, ..)`
* this suggestion makes the closure less implicit. this is a trade-off between the more pure view that
these objects are first functions that also have implicit data, to the more concrete view that it's
data that has an associated function.
* this suggestion will (hopefully) make this less confusing to beginners - they only need to understand
moves and references first. note we do  not actually know if this is true, since we can't test it on any newcomers.


# Summary
[summary]: #summary

Replaces the current function trait family `Fn(A, ..) -> Res`, `FnMut(A, ..) -> Res`, `FnOnce(A, ..) -> Res` by the forms
`Fn(&Closure, A, ..) -> Res`, `Fn(&mut Closure, A, ..) -> Res` and `Fn(Closure, A, ..) -> Res`.

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?


the importance of function traits for rust should not be understated. however, since it's introduction, the difference
between rust's `fn`, `Fn`, `FnMut` and `FnOnce` have been confusing to newcomers.
<insert references to newcomer questions about these>.
we believe that adopting this RFC will help newcomers that already understand borrowing rules, understand the different anonymous function types more easily, since it shows more clearly what is being borrowed and how.

say we have `object : T`. in rust, when you call `object.clone()`,  the type of `clone` isn't `fn() -> T`, it is `fn(&T) -> T`. that is, the first parameter of the function becomes the object itself, and it is explicitly written in the type of the function. this is important for a few reasons:
first, it accurately reflects both the semantics of the program, and its actual concrete implementation.
second, it allows the programmer to specify if the function should receive the object by value (`T`), by immutable reference (`&T`), or by a mutable reference (`&mut T`), or by any other smart pointer, and specifiy the needed lifetimes.

function traits, however, are inconsistent with this convention, and currently, don't have these advantages. first, `FnMut(i8) -> i8` seems to suggest a mere function from `i8` to `i8`, even though in actual implementation and semantics this is more similar to a function `fn(&mut Closure, i8) -> i8`, where `Closure` is the anonymous structure that is the closure.
second, we lose the second advantage as well, and instead, in order to tell in how the closure is used, we use the ad-hoc variants `Fn`, `FnMut`, `FnOnce`. this, however, makes the borrow checking errors less clear than it could have been.

under this RFC, function traits will become more consistent with the rest of the Rust functions and the way they are typed. in addition, `Fn`, `FnMut` and `FnOne` will stop beng ad-hoc special cases and will become more unified.

the change we view here can be seen as a trade-off between two view points on anonymous functions. the first viewpoint wants to see anonymous functions as pure as possible: it's mainly just a function that you can call - therefore it should have a type that looks like `fn(i8) -> i8`. for example, `FnMut(i8) -> i8`. it has implementation details - that in mplementation i has to carry around a closure, and that it has three variants, each of which has its own peculiar usage rules (only call an `FnOnce` once, can't borrow an `FnMut` multiple times, etc).
the second viewpoint is more concrete:
an anonymous function is actually an anonymous struct, called a closure, that implements a method as part of a function trait.

the first view has its advantages, of course - in the specific example of `|x : i64| { x + y }`, the closure isn't really very important, and it should be thought of more like a pure function.
however, we believe the second view gives us a better trade-off overall.



# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

of course, the most obvious drawback is that this is a breaking change of Rust's syntax. this will require code to be edited to reflect the new syntax.
however, this can be done relatively easily: it is possible to convert between the old and new syntax with a simple textual search.

in addition, another drawback tat we require adding a new keyword to the language, `Closure`. first of all, adding new keywords to the language should not be taken lightly, since the language should be kept as clean as possible. secondly, this would mean anyone that used the word `Closure` previously as the name for a datatype would have to rename it to something new.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

originally, we wanted to use the keyword `Self` instead of `Closure`. this is because in Rust, whenever a method is called on an object, we use the name `Self` to refer to that object's type. this is true in `trait` declarations, trait `impl`ementations, and data `impl` blocks. therefore, using `Self` is more in line with the rest of the language, and uses an established keyword with a clear meaning.
however, it has an ambiguity problem that  caused us to change it: whenever using a function trait inside a trait definition or an `impl` block, `Self` could then refer to both the type that is implementing the whole block, and to the anonymous function itself. in `impl` blocks that can be worked around, since in this case, you can refer to the type of the `impl` block in its more specific type, and reserve `Self` for the function trait (even though this is inelegant and confusing). however, in trait defnitions this problem is worse, since the only way to refer to the type that implements the trait, is using `Self`.

we have also considered using a new keyword instead of `Fn` for the new syntax. for example, the old function traits could remain `Fn`, `FnMut`, `FnOnce`, while the new syntax will be written `Fun(&Closure,..)`, and so on. this was considered in order to remove any possibility for ambiguity during the change. however:
* rust already uses "fn" exclusively to denote functions, and sometimes using "fn" and sometimes something else, will be very confusing.
* there is already almost no ambiguity between the old syntax and the new proposed syntax. in fact, in order to tell between the two, one needs to look at all of the occurences of the ord `Fn`, andlook at the first input: if the first input is `Closure`, `&Closure` or `&mut Closure`, this is the new syntax. if it isn't, or if there isn't any first input, this is the old syntax.
the only ambiguity is in the case there is already a defined datatype named `Closure`.


# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

* what will be the exact timetable and steps of the transition from the old syntax to the new syntax?
since this is only a syntactic change, it's probably best to rell out the changes slowly, using warnings, depracations, and an upgrade script. it is important to roll it out gently enough not to cause problems or break backwards comptibility too quickly.

possible steps, each of which can be inserted in seperate releases one after the other, are:

* add a warning for anyonw using a datatype named `Closure`
* add Closure as a keyword, preventing people from using `Closure` as the name of a datatype (if it is decided to do that; see the relevant point)
* add the new syntax, as a synonym for the old syntax, and publish an upgrade script.
* add a warning to usage of the old syntax
* deprecate the old syntax
* remove the old syntax altogether, completing the RFC.


* should datatypes named `Closure` be outlawed completely, or should they be allowed, as long as the programmer doesn't use them in the types of anonymous functions?
this is a rade-off between backwards compatibility and consistency - allowing that name might help someone that used the name `Closure` as a datatype maintain backwards compatibility. however, having a specific datatype name in the language that can't be use in the type of anonymous functions is unseemly and inelegant.

* should more general lifetimes be possible inside the function types?
for example, should the type `For<'a, 'b> Fn(&'a Closure, &'a i64, &'b i64) -> &'b i64` be possible?
the closest current type is `for <'b> Fn(&i64, &'b i64) -> &'b i64`, which in our new syntax is equivalent to `For<'a, 'b, 'c> Fn(&'c Closure, &'a i64, &'b i64) -> &'b i64`.
(the difference is that the old type can't require that the closure input and the first input will be taken with the same lifetime).

advantages:
- it will be more in line with the way functions are typed in rust, and more logically consistent

disadvantages:
- there doesn't seem to be any direct use for this kind of type. currently it isn't possible to create anonymous functions with such types, since when creating a anontmous function, the user can't reference the closure directly. this means that if this is allowed, such a type signature could only be used for things that are accepting other anonymous functions, and therefore could just use the more standard `Fn(&Closure, &i64, &'b i64) -> &'b i64` type.
- it requires a change to the type checker, whereas otherwise, this RFC only requires a syntactical change to te language.

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.


* extending this to other smart pointers: for example, a natural application is `Fn(&Pin<Closure>, ...)` for generators.
