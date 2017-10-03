---
layout: post
title: "Boundries"
categories: ruby
tags: ddd, rails
---

In my experience, one of the reasons that older Rails applications become hard
to maintayin is that they exhibit a poor sense of boundries.  Models become
inappropriatly intimate with records across they system, creating a tangled web
of dependency. Libraries are built into the core of the application, invoked
from dozens of different locations and gradually becoming more and more
difficult to upgrade or reyplace. Unit tests, in pursuit of speed and isolation,
mock methods and responses and become a brittle, frustrating ediface,
preventing refactoring and only barely trusted.

Maintaining a large app can feel like a constant battle with entropy, as you
fight with changing requirements, security patches, and that once-helpful
library that has not been updated in last three years. I've become increasingly
certain that the reason for this is that we are very bad at drawing useful
boundries.

<!-- more -->

## Apps have layers

Vanilla Rails is very good about driving the developer towards constructing
their application in layers -- routing is separated from rendering is separated
from domain logic, while controllers steer it all. (The nature of `ActiveRecord`
tends to mean that persistence and domain logic are throughly intermingled, but
that's a rant for another time.) Each layer is separated into it's colored box,
and the lines connecting them are generally clear and effective.

The problem is that there is very little done to encourage separation _within_
the layer where most of the work happens. The model layer, where business logic
and persistence are intermingled, is an undifferentiated mush. Sometimes, a
service layer is added. This _can_ help ... or it can be little more than
a place to dump multi-model functions as we struggle to avoid god-classes.

## The primordial ooze

In the early stages of an app, things are still fresh, and it's easy to make
changes. We can keep the structure of the app in our head, our libraries are
all brand new, and there is no legacy to fight against. We chase relations
through association chains, we mix-in the current labor-saving libraries and
_we make progress_. Tickets are falling, and an app is taking shape. Our tests
are starting to slow down, but that's no problem: Just stub the finder chains,
we know that ActiveRecord works. Add that post-publish tweet? No problem for
an `after_save` hook -- just make sure we mock-out the twitter library in our
specs!

I think that most of the problems we find in older Rails apps have their roots
in fecund periods like this. That early rush of _progress_, of _getting things
done_ is engrossing ... but it can often be a careless time where we lay down
traps for our future selves.
