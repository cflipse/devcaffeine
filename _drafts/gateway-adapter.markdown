---
layout: post
title: "Gateways and Adapters"
date: 2018-01-15
draft: true
---

I recently had opportunity to refactor an adapter class wrapping an API client
gem. This client gem serves as an interface to web-service calls, but from the
perspective of our application, it's simply a third-party interface to an
external system.

In these situations, there are usually two approaches to using that library:
 * A centralized adapter class is built, funnling all interactions through a
   central codepath
 * Nothing at all is done; references to the client are liberally spread
   throughout the code

I've see the latter option _frequently_ when working with HTTP client libraries.
Using a single adapter class is the much better option, as it centralizes your
dependency on foreign code, but the adapter class can still end up a tangled
mess.  I'd like to explore what it looks like to add another class, called a
gateway, into the mix.


<!-- more -->

The initial adapter was the product of an exploratory / demo spike; I began the
refactoring as an effort to get the code under test and into a long-term
maintainable state.  Initially, it resembled something like this:

```ruby
module RemoteApiAdapter


end

```



