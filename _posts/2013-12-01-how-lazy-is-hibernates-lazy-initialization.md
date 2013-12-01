---
layout: post
title: "How lazy is Hibernate's lazy initialization?"
category: til
tags: java hiberate json rest serialization
date: 2013-12-01 20:00
---

Originally I was going to give a more catchy title, but then I figured there is
no need to pick on Hibernate. It actually does its job well when you manage to
tame it.

* * *

Part of our application is using AngularJS which talks to a REST API, and the
data travels as JSON. So we needed a way to serialize our entities into JSON
strings, so we can send it back to the client. We are using JAX RS for the REST
API, which uses Jackson by default. It seemed like a straightforward decision to
use Jackson as our JSON serializer library.

Our domain model is a bit problematic and has lots of bidirectional
associations, which is not best kind to have when you want to serialize it.

I knew upfront that Hibernate proxies will cause some trouble, but I would have
never imagined how much. When the serializer tried to access a lazily
initialized property without a hibernate session, it threw an exception. There
is no surprise there. Luckily others have encountered this before, so there is a
[handy library](https://github.com/FasterXML/jackson-datatype-hibernate).
Unfortunately there is a [known issue](https://github.com/FasterXML/jackson-datatype-hibernate/issues/25)
about this as well.

This is the point where I made a mistake. I assumed that `Hibernate4Module` can
handle Hibernate proxies correctly, so I reattached the entity to a Hibernate
session and hoped for the best. It was like raining blood. Everything blew up.
The whole thing got stuck in an infinite loop because of the circular nature of
the associations in our domain model.

I went down the debugging rabbit hole to see where all the Hibernate proxies get
handled and it seemed fine. It seemed to return null for the uninitialized
proxies just as it was advertised in the docs. But somehow those proxies got
initialized during serialization despite all of my efforts.

It seems that Hibernate's lazy initialization is a bit more greedier than I
thought. I had the impression that it only loaded an entity or a collection when
you accessed one of its properties. So for an `Employee ---> Workspace` relation
calling a `employee.getWorkspace()` would not fire a query to the database. When
working with Jackson this was not the case. It loaded everything and caused me
some serious headache. It might have been a bug in the `Hibernate4Module` that I
did not notice or my setup was somehow broken.

I ended up using [mixins](http://wiki.fasterxml.com/JacksonMixInAnnotations) and
ignored all the properties that caused the trouble. I was lucky because I did
not need those properties on the client.
