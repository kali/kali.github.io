---
layout: post
title: Playing with Rust
---

Why Rust
--------

Among various items, last spring at the developer conference, Apple announced the 
release of a new language to succeed the venerable Objective C. As a Scala user, I was
amazed at how similar the first Swift samples Apple shared with the world
[looked familiar](https://leverich.github.io/swiftislikescala/).

<!--more-->

It's certainly not a coincidence. After years of being ostracized, functional programming
makes a big come-back and industry acknowledges its benefits by making functional and imperative
style co-existing in new languages. To my knowledge, Scala is the first successful language
designed around the idea of style cohabitation.

Another trend is the emergence of new languages compiling to metal. Since the apparition
of C++, and with, maybe, the exception of the very academic OCaml, only Go has really 
found some traction. Now Swift existence and release is a great indication that something
is happening in the language area.

When Apple released Swift, I felt very frustrated that no statement was made about its
availability on alternative platforms. But it prompted me to have a look at the LLVM
ecosystem and that's how I came to give some attention to Rust.

<a href="https://www.flickr.com/photos/aigle_dore/5677012485" title="Rust by Moyan Brenn, on Flickr"><img src="https://farm6.staticflickr.com/5104/5677012485_f585f48689_z.jpg" width="640" height="399" alt="Rust"></a>

Rust is designed by the Mozilla Research group. It has evolved a lot since its first
release and it still has a very experimental and shaky feel. Input and feedback from the
community is welcomed, and many big changes are still occurring, up to important syntax
modification. So Rust is not ready from prime time, but has some specificity that are
interesting, if not (yet ?) practical.

Rust acknowledges the need for local functional support, as Swift does, but where Swift
relies on runtime management of reference life cycle, Rust choose to make it a part of the
language. The idea is that the developer will have the opportunity to implement all the
usual reference management that is implicit in C or Java code, but instead of assuming
the developer does the right thing or running a GC, rust is able to 
*validate statically* reference manipulations at compilation time. The developer will need
to decorate the references in the code to express his intentions about ownership and
mutability of references. The compiler will check that the manipulation are correct, and
generate code that will be (in theory at least) as compact and efficient as a C compiler
would, with no runtime witchery (aka GC) required.

Of course, it comes with a price...

Some code
---------

I chose to give a shot at a Turing Machine implementation. If I manage that, I can do
anything, right?

Some parts are actually very nice. The machine definition (its program) is hard coded
for now, but I'll try to write a simple parser and load them from a file soon. Anyway.
For those of you who don't remember, a Universal Turing Machine program maps combination
of "mind state" and tape value to: a new mind state, a value to write on the tape, and
a tape move (left, right, or stay). And there is, of course, a magic "halt" state.

{% highlight Rust %}
enum MachineMove {
    Left = -1,
    Stay = 0,
    Right = 1
}

struct MachineTransition {
    write:bool,
    move:MachineMove,
    switch:int
}

struct MachineState {
    zero:MachineTransition,
    one:MachineTransition
}

struct MachineDefinition {
    states:Vec<MachineState>
}
{% endhighlight %}

So far, so good. Now for a hard coded machine definition. It's actually a
[two-state busy beaver](http://en.wikipedia.org/wiki/Busy_beaver#Examples)...

{% highlight Rust %}
    let def = MachineDefinition {
        states: vec![
            MachineState {
                zero:MachineTransition {
                    write:true, move:Right, switch:1 },
                one:MachineTransition {
                    write:true, move:Left, switch:1 }
            },
            MachineState {
                zero:MachineTransition { 
                    write:true, move:Left, switch:0 },
                one:MachineTransition { 
                    write:true, move:Right, switch:-1 }
            }
    ] }
{% endhighlight %}

OK, not too bad. *vec![...]* is a bit weird: *!* at the end of an identifier denotes
a macro. The rest would look quite the same in scala with case classes. One difference
though: the nested structure are *included*, not referenced.

Now things start to get a bit funky. I do not want to implement the Turing Machine in
functional immutable style: it is feasible, but copying the whole tape at each iteration
feels inefficient. So the Machine will a mutable structure. In Rust, a struct is not
intrinsically mutable or immutable: mutability is acquired through reference decoration.

Of course, I want a machine to know its definition (the "program" it's running) and
want to share this program among several machine, so I need "definition" to be a
reference and not a copy of the whole structure. The machine variable will be mutable,
but the definition will be immutable: mutability spreads to included structure, but not
through references.

{% highlight Rust %}
struct Machine {
    definition: &MachineDefinition,
    state: int,
    position: int,
    tapeLeft:Vec<bool>,
    tapeRight:Vec<bool>
}
/* ... and in main() ... */
    lef def = /* the definition, just as above */
    let mut machine:Machine = Machine { definition: &def, state:0,
            position: 0, tapeLeft:vec![], tapeRight:vec![false] };
{% endhighlight %}

No problem with this code in terms of ownership: the main() function will hold both
the definition and the machine in the same scope, so I can not get in a situation where
a machine has a pointer to a discarded definition. Well that's true, but Rust needs a little
help with figuring what I'm doing. And this take the form of... lifetime specifiers.

{% highlight text %}
beaver.rs:27:17: 27:36 error: missing lifetime specifier [E0106]
beaver.rs:27     definition: & MachineDefinition,
                             ^~~~~~~~~~~~~~~~~~~
{% endhighlight %}

I had a hard time understanding what that meant, but after a while, I managed to pull something
useful out of google: this area of the language having gone through important refactoring,
posts more than a few months old are next to useless. Anyway, I had to decorate my machine
structure that way:

{% highlight Rust %}
struct Machine<'a> {
    definition: &'a MachineDefinition,
{% endhighlight %}

The quote+identifier syntax is a lifetime specifier. My understanding of this — which I have not
been able to formally confirm with a piece of documentation — is that this is a parameterization
of the Machine structure (similar to a Java generic, or a C++ template) that allows me to define
Machine objects in any scope where a *definition* exists. You can have more than one of these
lifetime parameter in a given *struct* if your structure references other structures with different
lifetimes.

I did not have to decorate the structure instantiation and variables in the main(). Somehow the
compiler managed to validate what I was doing with that. This whole thing felt quite weird,
particularly because the "fix" does not seem to provide that much additional information to
the compiler. Maybe it's one of these case where this decoration should be assumed as a default
by the compiler when nothing is provided by the developer. I guess this comes with the territory.

There was a bit more of this mumbo-jumbo required to make work the "impl" part of my code.

{% highlight Rust %}
impl<'a> Machine<'a> {
    fn dump(&self) -> String {
        ...
    }
    fn step(&mut self) {
        ...
    }
}
{% endhighlight %}

I struggled a bit in a few places to manage to get the code working, and I'm not really happy with
it (it should be waaaaaay dryer) but this post is already too long :)

The code is [here](https://github.com/kali/rust-sandbox/blob/5d1be995d5194f3b6d2bdb0da70933defd2720f0/busy-beaver/beaver.rs).
