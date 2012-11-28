---
layout: post
title: "Reference Collapsing"
description: "Rvalue and Lvalue. What're the rules to reference collapsing?"
tagline:
category: [Programming]
tags: [C++11, Rvalue, Reference]
---

Reference collapsing is a set of rules dictating
whether a reference to a type is deduced to be a reference to an `lvalue` or `rvalue`.
It is a very interesting part of C++11 as it is an operation that only the compiler can perform.
It also plays a key part in the composition of higher order functions.

The rules are such that a template type parameter `T` can be
either an lvalue, a reference to an lvalue, or a reference to an rvalue.
But when it's in the form of `T&`, an lvalue reference to a template type parameter,
it always collapses into a reference to an lvalue.

The rule for an rvalue reference to a template type parameter, `T&&`, is more confusing.
It can expand to either an lvalue reference or an rvalue reference.
For `T` is an lvalue `S` or an rvalue reference `S&&`,
`T&&` becomes a reference to an rvalue,
and when `T` is an lvalue reference `S&`, `T&&` becomes a reference to an lvalue.

The following table summarizes the rules:

<table class="table table-condensed table-bordered" style="width: 400px;">
<tr class="success">
<td><code>T</code><span class="label pull-right">type</span></td>
<td><code>T&amp;</code><span class="label pull-right">lvalue ref</span></td>
<td><code>T&amp;&amp;</code><span class="label pull-right">rvalue ref</span></td>
</tr>
<tr class="info">
<td colspan="3" style="text-align: center;">
<strong>Collapses Into</strong>
</td>
</tr>
<tr>
<td><code>int</code></td>
<td><code>int&amp;</code></td>
<td><code>int&amp;&amp;</code></td>
</tr>
<tr>
<td><code>int&amp;</code></td>
<td><code>int&amp;</code></td>
<td><code>int&amp;</code></td>
</tr>
<tr>
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

Reference collasping is also what makes **perfect forwarding** in C++ possible.

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
