---
layout: post
title: "TIL: initialize an array with uniform values in C"
category: til
---

Sometime I admire C and then in the next moment I want to die out of
frustration. Immutable strings are a gift from this gods by the way. I can only
now appreciate them properly. It's a shame that C does not have them.

So back to the topic. This is such a great syntactic sugar.  When you want to
initialize an array with a uniform value you can do the following:

~~~c
char my_string[30] = {0};

/* Or for pointers */
char *my_string_array = { NULL };
~~~
