---
title: "Dependency Inversion"
tags:
 - object design
---

`Dependency Inversion`, or `Dependency Injection`, is one of those high-sounding object design priniciples that sounds really confusing and complicated.  For those of us who fled the Java world, it brings to mind heavy-weight configuration frameworks, filled with XML files and `FactoryBuilderFactory` classes.  The reality, however, is that this principal is much simpler and more straightforward than these frameworks would suggest. To me, it suggests a very simple rule: Objects should avoid calling `new` wherever feasible.

<!-- more -->

Not all objects are going to be able to obey this rule, of course -- otherwise,
you're not going to have a very interesting system.  The trick here is
mostly about _isolating_ the places where this is allowed.  Controller and
Orchestrator type objects are allowed to initialize all the tools they need,
but this is their _primary_ duty:  build and introduce the objects that
will do the actual work.




