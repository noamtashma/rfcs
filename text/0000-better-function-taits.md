- Feature Name: `better_function_traits`
- Start Date: 2020-11-20
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Replaces these forms:
`Fn(A, ..) -> Res`, 
`FnMut(A, ..) -> Res`, and
`FnOnce(A, ..) -> Res`
by the forms:
`Fn(&Closure, A, ..) -> Res`, 
`Fn(&mut Closure, A, ..) -> Res`, and
`Fn(Closure, A, ..) -> Res`.

# Motivation
[motivation]: #motivation

The importance of function traits for Rust should not be understated. However, since its introduction, the differences
between Rust's `fn`, `Fn`, `FnMut` and `FnOnce` have been confusing to newcomers.
TODO: insert references to newcomer questions about these.
I believe that adopting this RFC will help newcomers that already understand borrowing rules, understand anonymous function borrow rules more easily, since it shows exactly why the they are the way they are.
    
In addition, I believe that even for experienced rust programmers, having a more natural syntax will be beneficial.

In Rust, there is the convention that the "self" of a method is reflected in its type, as its first argument. For example, say we have `object : T`. When you call `object.clone()`, the type of `clone` isn't `fn() -> T`, it is `fn(&T) -> T`. That is, the first parameter of the function is the object itself, and it is explicitly written inside the type of the function. This is important for a few reasons:
First, it accurately reflects both the semantics of the program, and its actual concrete implementation.
Second, it allows the programmer to specify whether the function should receive the object by value (`T`), by immutable reference (`&T`), or by mutable reference (`&mut T`), or by any other smart pointer, and specify the needed lifetimes.

Function traits, however, do not adopt this convention, and therefore, don't have these advantages. What do I mean by that?
First, `FnMut(i8) -> i8` seems to suggest a mere function from `i8` to `i8`, similar to `fn(i8) -> i8`. However, in implementation and semantics, This function receives its closure in addition to its other arguments. This makes the closure act like the "self" argument in method calls. Therefore, `FnMut(i8) -> i8` is actually more similar to a function `fn(&mut Closure, i8) -> i8`, where `Closure` is the anonymous structure that is the closure. In fact, `func()` actually desugars to `func.mut_call()`, which undeniably receives `self` as the first argument.
In addition, we lose the second advantage: We can't specify in which way we are accessing the closure. Instead, in order to specify that, we use the three ad-hoc variants `Fn`, `FnMut` and `FnOnce`.

Therefore, we suggest to change the syntax of function traits to reflect the fact that one of the inputs is the closure, much like the existing "self" inputs. For example, the current `Fn(i8)->i8` would be changed to `Fn(&Closure, i8) -> i8`, reflecting that the first input is actually the anonymous closure, taken by immutable reference. It should be noted that here `Closure` is a keyword, akin to `Self`. Similarly, `FnMut()->Res` would become `Fn(&mut Closure)->Res` and `FnOnce()->Res` would become `Fn(Closure)->Res`. Overall, with this proposal, the function traits become more unified with each other, are more in line with regular function calls, and reflect the actual implementation and semantics more closely.

In addition, I believe that function trait's borrow checking behavior is confusing. Even for novices that already understand the normal borrow checking rules, can be confused by the fact that an `FnMut` function has to be declared mutable for it to be called (TODO: give examples). I argue that showing the root cause of these borrow checking errors in the types, i.e. the implicit `Closure` input, will help clear things up for newcomers and help seasoned rustaceans as well.

The issue this RFC is all about can be seen as a trade-off between two view points on anonymous functions. The first viewpoint wants to see anonymous functions as pure as possible: it's mainly just a function that you can call - therefore it should have a type that looks like `fn(i8) -> i8`. For example, `FnMut(i8) -> i8`. This view wants to ignore the technical details that capturing the environment entails. Therefore, the closure structure behind the scenes is an implementation detail. And, it has nontrivial borrowing behavior, so its type has to be marked one of three variants, each of which has its own usage rules (only call an `FnOnce` once, can't borrow an `FnMut` multiple times, etc).

The second viewpoint is more concrete:
an anonymous function is actually an anonymous struct, called a closure, that implements a method as part of a function trait. Therefore, its borrow checking behavior is clearly determined by references and their lifetime parameters (specifically, `Fn(&Closure)` vs. `Fn(&mut Closure)`, etc), and the `self` parameter has to be marked as a regular input to the method (which is the function itself). 

The first view is more in line with a programmer that wants to write some function, or wants some action to be done, or that writes functions that are relatively simple. It's shorter to write, and when it works, it works.
The second view is better suited to a programmer that wants to understand why the compiler is refusing to compile his code.



# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This is basically a short retelling of the explanation of the differences between `Fn`, `FnMut` and `FnOnce`, even though this already exists in the language.

During compilation, in order to implement anonymous functions, every anonymous function is replaced by an anonymous struct, called "the closure", that contains the data it needs to run. For example, consider:

```
fn main() {
    let mut arr = vec![1, 2, 3];

    let mut push_to_arr = |val| {
      arr.push(val);
    };

    //println!("can't use arr here: {:?}", arr);
 
    push_to_arr(9);
    push_to_arr(10);

    assert!(arr == vec![1,2,3,9,10]);
}
```

In order to implement this anonymous function, the compiler will generate a new struct for the closure, in a way roughly equivalent to this:

```
struct AnonClosure<'a> {
  arr_ref : &'a mut Vec<usize>,
}

impl<'a> AnonClosure<'a> {
  fn call(&mut self, val : usize) {
    self.arr_ref.push(val);
  }
}

fn main() {
    let mut arr = vec![1, 2, 3];

   // let mut push_to_arr = |val| {
   //   arr.push(val);
   // };

   let mut push_to_arr = AnonClosure { arr_ref : &mut arr };

    //println!("can't use arr here: {:?}", arr);
 
    //push_to_arr(9);
    push_to_arr.call(9);
    //push_to_arr(10);
    push_to_arr.call(10);

    assert!(arr == vec![1,2,3,9,10]);
}
```

Note that in the function `call`, we require getting the closure through a mutable reference. This is because, we can't change values through the closure if we only have
an immutable reference, and we can't take it by value, because then we would consume the closure and won't be able to call out function twice.

TODO: If we took it through an immutable pointer, it couldn't change the closure values, and therefore it would work like an immutable function, etc etc.

To summarize: anonymous functions can access their closures in a few ways:
* through an immutable reference: this will happen if your function doesn't mutate its captured variables
* through a mutable reference: this will happen if your function mutates its capured variables
* by value: this will result in a function that is only callable once. this can be useful in order to transport values in some circumstances.

We have a way of typing out this difference, using traits: these are called function traits.
each funcion trait requires a function like `call` to be implemented, with the requied type.
after this RFC, they will be written like this:
`Fn(&Closure, inputs..) -> result`, or `Fn(&mut Closure, inputs..) -> result` or `Fn(Closure, intputs..) -> result`,
Reflecting the actual type of the auto-generated `call` method.
In the example, the right function trait would be `Fn(&mut Closure, usize)` since the `call` function has type `fn(&mut AnonClosure, usize)`.
These three options tell you which way the Closure is accessed. Which one to use depends on which of the three previous cases your function belongs to.
Specifically, `Closure` is a keyword, that can be thought of as being swapped with the actual anonymous closure struct at compile time.

TODO: insert the example but using a function trait

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

After the RFC will be fully implemented, we will have these changes:

We change the syntax of the `Fn` traits:
`Fn(..) -> res` becomes `Fn(&Closure, ..) -> res`
`FnMut(..) -> res` becomes `Fn(&mut Closure, ..) -> res`
`FnOnce(..) -> res` becomes `Fn(Closure, ..) -> res`

and also the same change with the forms that have no return types.

The two ways of writing the traits will be completely equivalent in every way. At the beggining, the two forms will coexist for some time in order to be backwards compatible while easing the transition. (see proposed steps in unresolved questions section).
Writing specific lifetime variables in the `Closure` references is forbidden (at least in this draft: see more general lifetimes in future possibilities).

The two forms can be (almost) unambiguously be distinguished from each other: When the compilers encounters `Fn`, if its first input type contains `Closure` in some way, then it mush be of the new form. Otherwise, it must be of the current form.

Using types with the name of `Closure` will become a compilation error, in order to resolve the ambiguity. (or, alternatively, only using types with the name `Closure` inside function traits will become a compilation error: see unresolved questions).

This RFC can be implemented as a syntactical change only, if the compiler writers choose to do so.

# Drawbacks
[drawbacks]: #drawbacks

Of course, the most obvious drawback is that this is a change of Rust's syntax. This will require code to be updated to reflect the new syntax.
However, this can be done relatively easily: It is possible to convert between the current and new syntax with a simple textual search, and the two syntaxes can be allowed to coexist for a transition period in order to ease the transition.

In addition, another drawback tat we require adding a new keyword to the language, `Closure`. Adding new keywords to the language should not be taken lightly, since the language should be kept as clean as possible. In addition, this would mean anyone that used the word `Closure` previously as the name for a datatype would have to rename it to something new. (Or perhaps types of this name could be forbidden inside function traits, but allowed elsewhere: see unresolved questions).

Introducing the `Closure` input introduces new positions for lifetime parameters (in the immutable and mutable reference cases). In general, I believe it is a good thing - see the `Future possibilities` section about more general lifetimes. However, the current RFC basically ignores that possibility - even writing any lifetime parameter there is an error. This is in order to minimize the change the RFC would bring. Changing that would necessitate a change to the type system, which this RFC does not require. 
On one hand, a consequence of this is that the lifetime elision rules for function traits have to remain consistent with what they were before this RFC. On the other hand, another input with a lifetime slot was added (`&Closure`), that looks like it should act like `&Self`. These two point clash, and make the elision rules become less consistent and less intuitive.
However, It remains to be seen if this will have much impact in practice. And in addition, this will be remedied eventually if the `Future possibilities` section is taken as well.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

### Rationale
I believe the rationale itself is explained clearly in the Motivation section.

### alternatives

Of course, there is the alternative of doing nothing, and remaining with our current syntax. Other than that alternative, there are also:
TODO

originally, we wanted to use the keyword `Self` instead of `Closure`. this is because in Rust, whenever a method is called on an object, we use the name `Self` to refer to that object's type. this is true in trait declarations, trait implementations, and data `impl` blocks. therefore, using `Self` is more in line with the rest of the language, and uses an established keyword with a clear meaning.
however, it has an ambiguity problem that  caused us to change it: whenever using a function trait inside a trait definition or an `impl` block, `Self` could then refer to both the type that is implementing the whole block, and to the anonymous function itself. in `impl` blocks that can be worked around, since in this case, you can refer to the type of the `impl` block in its more specific type, and reserve `Self` for the function trait (even though this is inelegant and confusing). however, in trait defnitions this problem is worse, since the only way to refer to the type that implements the trait, is using `Self`.

we have also considered using a new keyword instead of `Fn` for the new syntax. for example, the old function traits could remain `Fn`, `FnMut`, `FnOnce`, while the new syntax will be written `Fun(&Closure,..)`, and so on. this was considered in order to remove any possibility for ambiguity during the change. however:
* rust already uses "fn" exclusively to denote functions, and sometimes using "fn" and sometimes something else, will be very confusing.
* there is already almost no ambiguity between the old syntax and the new proposed syntax. in fact, in order to tell between the two, one needs to look at all of the occurences of the ord `Fn`, andlook at the first input: if the first input is `Closure`, `&Closure` or `&mut Closure`, this is the new syntax. if it isn't, or if there isn't any first input, this is the old syntax.
the only ambiguity is in the case there is already a defined datatype named `Closure`.

thee was also an idea along the following lines: if we have an anonymous function of type `T`, then `T` is actually the type of the closure itself. therefore, we could replace `T : FnMut(i8) -> i8` with the syntax `T : Fn(&mut T, i8) -> i8`. this is indeed an elegant solution that reflects the reality of the implementation of function traits, and doesn't require a new keyword. however, this syntax can't be used inside `dyn` or `impl`. this is because, in these circumstances, we don't have a name for the type of the anonymous function, and we can't write `dyn Fn(&mut T, i8) -> i8` because we don't know what `T` is.


# Prior art
[prior-art]: #prior-art

I do not know of any prior art: the problem that there are a few different semantics to functions that have closures, seems to be unique to rust's borrowing rules and lifetime systems.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### What will be the exact timetable and steps of the transition from the old syntax to the new syntax?
since this is only a syntactic change, it's probably best to rell out the changes slowly, using warnings, depracations, and an upgrade script. it is important to roll it out gently enough not to cause problems or break backwards comptibility too quickly.

possible steps, each of which can be inserted in seperate releases one after the other, are:

* add a warning for anyonw using a datatype named `Closure`
* add Closure as a keyword, preventing people from using `Closure` as the name of a datatype (if it is decided to do that; see the relevant point)
* add the new syntax, as a synonym for the old syntax, and publish an upgrade script.
* add a warning to usage of the old syntax
* deprecate the old syntax
* remove the old syntax altogether, completing the RFC.


### Should we change the rustc error messages when it encounters a misplaced `Fn(Closure, ..)` type?
one might suspect that after this change programmers will be moe inclined to type an `Fn(Closure, ..)` type instead of a more appropriate and more common `Fn(&Closure, ..)` type, since now it has a shorter and simpler name. therefore, it might be needed to update rustc's error's to add a new hint.

### Should datatypes named `Closure` be outlawed completely, or should they be allowed, as long as the programmer doesn't use them in the types of anonymous functions?
this is a rade-off between backwards compatibility and consistency - allowing that name might help someone that used the name `Closure` as a datatype maintain backwards compatibility. however, having a specific datatype name in the language that can't be use in the type of anonymous functions is unseemly and inelegant.


# Future possibilities
[future-possibilities]: #future-possibilities

### Extending this to other smart pointers
for example, require for the generators feature, is a new function trait `FnPin` which, in our syntax, would look like `Fn(&mut Pin<Closure>, ...)`.
this then becomes a natural extention of the syntax, and removes the clunkiness of adding a new function trait into the language.
in fact, one might allow all smart pointer types to be the first argument. what will be the uses of that, and what will be the syntax for creating that kind of anonymous function, will need to be determined.

### More general lifetimes for function traits

Introducing the `Closure` input introduces new positions for lifetime parameters (in the immutable and mutable reference cases). What should be done with them?
The current RFC basically ignores that possibility - even writing any lifetime parameter there is an error. This is in order to minimize the change the RFC would bring. Changing that would necessitate a change to the type system, which we do not require now. However, using that lifetime parameter in types can be beneficial.

For example, consider this code:
```
let mut state = T();
let mut state_machine = move | input : i8 | {
    state.update(input);
    return &state;
};
```
Where we attempt to create a state machine. Whenever we call `state_machine` it updates its state and then exposes it to the user. (we use `move` so that the state machine could live independently of its creator). We would like to expose the state by reference to avoid an expensive copy.
In current Rust, this is a compilation error: "captured variable cannot escape `FnMut` closure body".
However, imagine that our function had the type `Fn(&'a mut Closure, i8) -> &'a T`. That is basically the same type we would have if we defined our own state-machine struct and had `call` as a method. Therefore, the code is now safe and sound.

This works generally whenever an anonymous function wants to return a reference to one of its values. Then that reference could only be used until the next call to the function.

This is one example of a use for this lifetime parameter slot.  However, this should apply to most cases of "captured variable cannot escape `FnMut` closure body". In addition, using this new lifetime might add uses I haven't thought of.

In addition, implementing this change should be very easy, since it already fits the syntax, and the borrow checking rules for it match exactly the rules for regular functions already.

Advantages:
- It will be more in line with the way functions are typed in rust, and more logically consistent

Disadvantages:
- It requires a change to the type checker, whereas otherwise, this RFC only requires a syntactical change to te language.

Currently, I am writing this RFC in a way such that only a syntactical change to the language is needed. However, if it is deemed easy and useful to implement this as well, this can be added to this RFC.

### If the previous point is picked, should lifetime elision rules for anonymous functions be changed?

Currently, the lifetime elision rules for function traits are the same for regular functions. That means that after adding the `Closure` argument, the lifetime rules are the same <i>except</i> that they ignore the `Closure` argument, giving it a distinct lifetime variable, and then apply the usual elision rules. This is desireable because:
    * This way is backwards compatible
    * This way is more in line with viewing your function as "just a function"
However, after adding the `Closure` argument, this becomes somewhat awkward, because it becomes inconsistent with the regular lifetime elision rules.
Alternatively, one could update the lifetime elision rules not to ignore the `Closure` argument, or perhaps treat it specially in another way entirely.
