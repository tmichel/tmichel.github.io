---
layout: post
title: "TIL: JSF + Guava not such a good idea"
category: til
---

The other day I was working on a page that presents hierarchical data and I
wanted to retrieve each new level with an Ajax request. So far so good, but JSF
makes simple things harder than they should be.

I created a view-model for each level of data. It was very basic:

~~~java
class EntityViewModel {
    private Entity entity;
    private List<EntityViewModel> children;
}
~~~

After querying the database for the entities, I had to transform and insert the
retrieved data into the view-model object tree. The obvious solution is to wrap
every entity in a view-model and set the parent's `children` property to the new
list.

My mistake was that I used Guava's fancy `Lists#transform` method instead of
a simple `foreach`. For the first level, everything worked fine, but when I
wanted to retrieve the second level nothing came back in the response.
Everything seemed to be fine, but somehow JSF did not pick up the changes in the
object-tree and did not render anything. All the responses were empty.

After a few hours of staring at the screen helplessly and blaming JSF, I
realized that the problem was with the `List<>` implementation that Guava
returned. The `#transform` method only returns a _view_ of the original list, so
JSF did not have a chance to notice when the `children` property of an
object changed, thus it never rendered anything below the first level.

The takeaway? Don't shy away from using the good ol' `foreach`. It will probably
never let you down.
