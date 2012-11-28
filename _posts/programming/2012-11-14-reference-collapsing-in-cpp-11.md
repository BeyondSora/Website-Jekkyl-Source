---
layout: post
title: "Reference Collapsing"
description: "Rvalue and Lvalue. What're the rules to reference collapsing?"
tagline:
category: [Programming]
tags: [C++ 11, Rvalue, Reference]
---

Reference collapsing is a set of rules dictating
whether a reference to a type is deduced to be a reference to an `lvalue` or `rvalue`.
It is a very interesting part of C++11 as it is an operation that only the compiler can perform.
It also plays a key part in the composition of higher order functions.

The rules are as follows:

<table class="table table-condensed table-bordered" style="width: 400px;">
<tr class="success">
<td><code>T</code></td>
<td><code>T&amp;</code></td>
<td><code>T&amp;&amp;</code></td>
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

As shown, `lvalue reference` is *infectious* and applying it to any type
will turn that type into its own kind.
On the other hand, `rvalue reference` is less powerful and only captures
a regular type or another of its own kind.
A plain non-reference type is the weakest, as expected.

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
    Type       &a;        // lvalue reference
    Type      &&b;        // rvalue reference
    LRef_Type &&c;        // lvalue reference
    RRef_Type  &d;        // lvalue reference
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

    vector<StudentGrade> &lRefGrades = grades;
    forward(lRefGrades);    // calls process(vector<StudentGrade> &grades);

    vector<StudentGrade> &&RRefGrades = grades;
    forward(RRefGrades);    // calls process(vector<StudentGrade> &&grades);
}
{% endhighlight %}

In C++11, reference collapsing are only present in the context of
`auto`, `decltype`, `typedef`, or a template type parameter.
The rules are this way largely to make perfect forwarding possible.
They serve little purpose when applied outside of a template.
