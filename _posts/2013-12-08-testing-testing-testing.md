---
layout: post
title: "Testing. Testing. Testing."
---

I dropped this little pearl in my code the other day.

    while (i < vec->len && vec->data[i] != value) ;

It is from a simple vector implementation's remove method. Do you see any
anything wrong with it? I did not for a while and oddly it behaved well. Too
well actually. For a while I did not put more than one element in the vector, so
this little loop never actually happened. The surprise came when there was two
elements in it and I wanted to remove the second one.

My biggest mistake here is that I did not write unit tests for this, because I
was confident enough I would not screw this up. Unfortunately my confidence was
without merits.

Testing is always time well spent.
