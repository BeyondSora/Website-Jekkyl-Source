---
layout: post
title: "Implicit Function Declarations Will Bite"
tagline:
description: "Why can you call a function without defining it in C. Why is it bad?"
category: [Programming]
tags: [C, implicit function]
---

Is the following code correct in C?

    #include <stdio.h>
    int main () {
        int i = DoSomething();      /* does it work? */
        printf("%d\n", i);
        return 0;
    }
    int doSomething (int num) {
        return 5;
    }

I haven't done any survey on this, but I dare to guess that most modern day
coders would answer the above code would not compile.
Reasons are obvious.
`doSomething` is used without being declared first and the parameter defined
in its function signature is missing from the caller.
At the very least, when I first saw it, I was convinced that it would not work.

However, it turns out that it is indeed
*legal*<sup id="ref_f1"><a href="#f1">1</a></sup> C code (albeit legacy).
Running it against gcc without any option produced no error nor any warnings.
And executing the resulted binary did indeed produce an output of 5.
This is known as *the implicit function declaration rule*.

So, how does this rule apply?
How does the compiler handle an unresolved function call?
Finally, what are the pitfalls?

####A Thing of the Past

It should be noted that this feature is only in the C89 standard due to historical reasons.
As some may recall, prior to C89 the C language as defined in K&R's
*The C Programming Language*<sup id="ref_f2"><a href="#f2">2</a></sup>
did not support function prototypes.
An ordinary C function declaration in K&R may look like the following.

    foo (num, input)    /* assumed to return int and no parameter types */
        int num;        /* types are specified here */
        char *input;
    {
        return 5;
    }

During standardization, C obtained the feature of employing function prototypes,
as it had been shown to be quite beneficial from various uses in C++.
However, the vast amount of C code written prior to standardization had no access
to such features.
To cope with this and other limitations, K&R C was very lax on various things.
For one, declaring a variable without an actual type preceding it was perfectly legal.
Any such variable simply needed to be qualified with `auto`
and would then be implicitly assumed to be an integer (known as *the implicit int rule*).
Allowing a function to be used without first being declared was another compromise.

####Compiler Does Its Magic

When the compiler encounters a name preceding a pair of parenthesis within a function body,
it looks up the name within the current scope and tries to match it with one of the
function declarations, if any, defined prior to this name.
If not matched, the compiler then, applying the implicit function declaration rule,
goes on assuming that this name belongs to a function which takes in whatever arguments
the caller supplies and returns an integer.
The compiler performs on type checking on implicitly declared functions.

Hence, the signature of `doSomething` is to be inferred by the compiler s the following.

    int doSomething (void);

Is this the correct signature? No, but it doesn't matter.
The compiler will implicitly declare the function as such at the place where
it is first used and, after the translation unit is produced, pass the
control to the linker.
The linker will then search for the name in the translation unit and in this case
match it with the *real* `doSomething` defined right after `main`.

It this the correct behavior? Not quite, but close enough.
`doSomething` does not actually use its parameter for anything.
It simply returns the integer 5 and calls it a day.
Thus, in this case it doesn't affect the outcome of the resulting binary.

So what really happened?
The compiler finds an unresolved function call, assumes it is of a particular form,
produces a translation unit, and passes along to the linker.
Then, the linker does all in its power to prepare the caller to call
*that particular form* of the function, not the *real form*.
In our case, it really just *happens* to work.

####And When Luck Runs Out

What if the code looks something like this instead.

    #include <stdio.h>
    int main () {
        int i = DoSomething();      /* does it still work? */
        printf("%d\n", i);
        return 0;
    }
    int doSomething (int num) {
        return num;
    }

This code will still compile without any complaints from the compiler.
However, the produced binary unfortunately will no longer give the correct output.
The binary will now likely, depending on various conditions, produce 0, 1, or 15003,
crash, trash your entire movie collection, or even kill your cat!

So, what's wrong?

Well, things have started being weary as soon as the compiler assumes
`doSomething` is of a particular form.
We know that this function takes in an integer argument and uses that argument in its body,
but the compiler, at the point of the caller, does not yet know this.
So, it assumes that `doSomething` takes in no arguments and returns an integer, and then
generate code based on that assumption.
Note that when the compiler comes around to the actual function definition, it will
indeed generate correctly behaving code.
The only culprit is the statement which calls the function.

During runtime, `doSomething`, under the assumption that its argument can be found
from one of the processor registers or somewhere on the stack frame, will simply assign
`num` with the value read at the argument's supposed location, and return it to the caller.
In this case, however, this action is undefined and could trigger anything.

####Bad, Bad, Bad

This is serious trouble.
Implicitly declaring a function of the wrong signature has little to no symptoms
(no warning unless you turn on `--Wimplicit-function-declaration` in gcc),
which menas in a large, complex codebase, such code can potentially go unnoticed
for a long, long time and bite hard when and where least expected.

For example, given the following code, what is the expected output?

    int foo (UserInput *input, SystemTime time) {
        double result = BeggingForTrouble(input, time);
        if (result == 2.0)
            BurnDownYourHouse();
        else
            KillYourCat();
        return result;
    }
    double BeggingForTrouble (int input) {
        double result;
        /* do stuff with input... */
        return result;
    }

The caller here is attempting to pass in two arguments, one is a pointer to `UserInput`
and the other is an instance of `SystemTime`, to `BeggingForTrouble`, which actually
takes an integer as its sole argument.
Does this code compile? Yes, the compiler won't complain a thing.
Will it burn down your house or kill your cat? Both, either, or something else.
It is undefined behavior and thus anything can happen.

The analysis can go deeper. Two things are wrong at the place of the caller.
`BeggingForTrouble` is assumed to take in whatever arguments the caller supplies.
So, storage space for input and time will be allocated when preparing the calling
stack for the function.
But the function only takes one argument and that argument's type matches neither of the
two arguments passed from the caller.
The function will still try to grab the argument from the location in where it resides
if the function is invoked properly.
Clearly, only garbage can be found from the supposed location.
The other thing that is wrong is that `BeggingForTrouble` returns a double floating-point
number, but is assumed to return an integer.
So, the compiler would generate code that converts a region of memory that is the size
of an integer into a double floating-point number and assign to result.
So, the likely-is-already-garbage data will thus be altered again.
Though, likely it won't matter much at all, as the returned data at this time is already
beyond saving anyways.

####A Quick Summary
At where the implicit function declaration rule is used,
the resulting executable would only be correctly behaving
if the assumed signature matches exactly both the signature
the caller uses and the actual signature of the function.

Any code that leverages the implicit function declaration rule is at high risk of breaking,
as the compiler will not conduct any type checking on it.
No warning will be issued if there is a mismatch between the assumed signature,
the signature used in caller, and the signature of the actual function itself.

####Which Brings Us To
So, after coming to understand the implicit function declaration rule,
two additional questions must be raised. Is this feature still necessary today?
Is it still relevant?

My answer to the first question is No.
The C programming language has come a long way since its advent in the early 70s.
Language features such as this are obsolete and their presence in the standard are
mostly for compatibility reasons with legacy code.
The implicit function declaration rule was rendered useless by the introduction of
function prototypes into the standard since C89. As a matter of fact,
support for implicit function declarations were dropped from the C99 standard.

But is it still relevant? Especially to the newer generation C programmers
(such as myself), among whom many were born after C had been standardized,
are legacy features such as this still relevant?
My answer is Yes. An enormous amount of legacy C code is still in production today,
and with the C99 standard not nearly as widely supported as the standard committee had hoped,
C89 is still in wide use today. This means, not just maintaining legacy code,
even writing new code could accidentally spur the uses of implicit function declarations.
And hence the reason to why I am convinced that new C programmers should get themselves
familiar with once-widely-used language features such as this.
Only can a programmer immediately spot bugs introduced by the implicit function declaration rule,
if he/she understands it well enough. Same can be said to other legacy features in C.


<hr>
\*<sup>1</sup> Implicit function declaration is no longer supported as of C99.
<a id="f1" href="#ref_f1">&#8617;</a>

\*<sup>2</sup> First edition only. Second edition was updated to describe ANSI C (C89).
<a id="f2" href="#ref_f2">&#8617;</a>
