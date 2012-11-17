---
layout: post
title: "The 'Goes-to' Operator?"
tagline:
description: "A 'magic' operator, demonstrating how fun C/C++ can be!"
category: [Programming]
tags: [C, C++, operator precedence]
---

    #include <stdio.h>
    int main () {
        int x = 10;
        while( x --> 0 ) {    // x goes to 0
            printf("%d ", x);
        }
    }

####What??

My first impression after seeing the above code was WTH. Never did I ever hear of such "goes-to" operator in C or C++. Is this some kind of magic?


Well... No.

    while ( x --> 0 )

This is simply the same as:

    while ( (x--) > 0 )

Since `operator --` takes higher precedence than `operator >`, the code would post decrement x first,
and then compare x to 0. Coupled with the while loop,
this creates a "goes-to" illusion of some sort of range operation.

Now, isn't C simply brilliant?
