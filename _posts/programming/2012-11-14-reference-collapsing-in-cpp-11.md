---
layout: post
title: "Reference Collapsing"
description: "Rvalue and Lvalue. What're the rules to reference collapsing?"
tagline:
category: [Programming]
tags: [C++11, Rvalue, Reference]
---

Reference collapsing is a set of rules dictating
whether a reference to a type is deduced to be a reference to an **lvalue** or
**rvalue**<sup id="ref_f1"><a href="#f1">1</a></sup>.
It is a very interesting part of C++11 as it is an operation that only the compiler can perform.
It also plays a key part in the composition of higher order functions.

The rules are such that a template type parameter `T` can be
either an lvalue, a reference to an lvalue, or a reference to an rvalue.
But when it's in the form of `T&`, an lvalue reference to a template type parameter,
it always collapses into a reference to an lvalue.

The rule for an rvalue reference to a template type parameter, `T&&`, is more confusing.
It can collapse into either an lvalue reference or an rvalue reference.
For `T` is an lvalue `S` or an rvalue reference `S&&`,
`T&&` becomes a reference to an rvalue,
and when `T` is an lvalue reference `S&`, `T&&` becomes a reference to an lvalue.

The following table summarizes the rules:

<table class="table table-condensed table-bordered" style="width: 400px;">
<tr class="success">
<td><span class="label pull-left">type</span></td>
<td><span class="label pull-left">lvalue</span></td>
<td><span class="label pull-left">lvalue ref</span></td>
<td><span class="label pull-left">rvalue ref</span></td>
</tr>
<tr class="info">
<td></td>
<td colspan="3" style="text-align: center;">
<strong>Collapses Into</strong>
</td>
</tr>
<tr>
<td><code>T</code></td>
<td><code>int</code></td>
<td><code>int&amp;</code></td>
<td><code>int&amp;&amp;</code></td>
</tr>
<tr>
<td><code>T&amp;</code></td>
<td><code>int&amp;</code></td>
<td><code>int&amp;</code></td>
<td><code>int&amp;</code></td>
</tr>
<tr>
<td><code>T&amp;&amp;</code></td>
<td><code>int&amp;&amp;</code></td>
<td><code>int&amp;</code></td>
<td><code>int&amp;&amp;</code></td>
</tr>
</table>

To emphasize, an lvalue reference is *infectious* and applying it to any type
will turn that type into its own kind.
On the other hand, an rvalue reference is less powerful and only captures
a regular type or another of its own kind.
A plain non-reference type is the weakest, which is expected.

Although the chart shows combinations such as a reference to
another reference, this is not actually valid syntax
(i.e. `int && &Rref_LRef_Int` is illegal).
But it is perfectly legal to have a reference to a typedef'ed type that
expands to a reference to another type.

For example:

{% highlight c %}
struct Type {};
typedef Type&  LRef_Type;
typedef Type&& RRef_Type;

void foo () {
    Type        t;
    Type       &a = t;          // lvalue reference
    Type      &&b = move(t);    // rvalue reference
    LRef_Type &&c = t;          // lvalue reference
    RRef_Type  &d = t;          // lvalue reference
}
{% endhighlight %}

Reference collasping is also what makes
**perfect forwarding**<sup id="ref_f2"><a href="#f2">2</a></sup> in C++ possible.

{% highlight c %}
vector<Result> process (vector<StudentGrade> &grades);
vector<Result> process (vector<StudentGrade> &&grades);

template <typename T>
auto forward (T&& arg) -> decltype(process(declval<T>())) {
    return process(arg);
}
{% endhighlight %}

The above example demonstrates the use of perfect forwarding.
The deduced type of `arg` will be the same as its actual type.
This is significant as the compiler will pick the suitable `process`
to invoke depending on what kind of reference you pass to `forward`.
No template specialization required.

The code below shows some uses of such perfect forwarding templates.

{% highlight c %}
void foo () {
    vector<StudentGrade> grades = getGrades();
    forward(grades);        // calls process(vector<StudentGrade> &grades);

    vector<StudentGrade> &LRefGrades = grades;
    forward(LRefGrades);    // calls process(vector<StudentGrade> &grades);

    vector<StudentGrade> &&RRefGrades = move(grades);
    forward(RRefGrades);    // calls process(vector<StudentGrade> &&grades);
}
{% endhighlight %}

In C++11, reference collapsing are only present in the context of
`auto`, `decltype`, `typedef`, or a template type parameter.
The rules are this way largely to make perfect forwarding possible.
They serve little purpose when applied outside of a template.

<hr>
\*<sup>1</sup> An excellent article on rvalue and move semantics can be found
<a href="http://www.cprogramming.com/c++11/rvalue-references-and-move-semantics-in-c++11.html" target="_blank">here</a>.
<a id="f1" href="#ref_f1">&#8617;</a>

\*<sup>2</sup> Find out more about perfect forwarding and move semantics
<a href="http://www.codeproject.com/Articles/397492/Move-Semantics-and-Perfect-Forwarding-in-Cplusplus" target="_blank">here</a>.
<a id="f2" href="#ref_f2">&#8617;</a>
