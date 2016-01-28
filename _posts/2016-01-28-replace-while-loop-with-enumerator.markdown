---
layout: post
title: Replace While Loop With Enumerator
date: 2016-01-28T11:36:34-05:00
categories: ruby
tags: refactoring, enumerator
---

Sometimes, it looks like it is not possible to avoid using an accumulating
array, a pattern that feels unnatural in Ruby.

Recently, I've need to chase down and unroll pagination links over a
JSON / REST api.  I don't know how many pages there will be, and it's probable
(but not guaranteed) that I need to retrieve and use all of the content.
Since each page is dependant on results from the previous page, there is no
obvious Enumerable parallel.  Here, I'll demonstrate a quick refactoring that
will provide in a clean, lazy enumerable object.

<!-- more -->

This being a HATEOAS API, the next page link is embedded in the response JSON,
tucked under a `["links"]["next"]` key.  To fetch all of the data, I end up with
code that looks something like:

```ruby
def retrieve_all_pages(url)
  widgets = []

  while url
    response = connection.get url
    json     = JSON.parse response.body

    url      = json["links"]["next"]

    widgets.concat json["widgets"]
  end

  widgets
end
```

As ruby goes, this is pretty ugly.  When there's something distinct to
enumerate over, it's recommend to replace the accumulating `widgets` array with
a `#map` call, or working with some other `Enumerable` method.  Unfortunately,
there isn't a clear parallel for this case.

Another issue is that this loop fetches every single page, _before_ returning
control, and regardless of how many results I actually end up using.

Fortunately, there is `Enumerator`[1], Ruby's answer to producing generators.
The enumerator class produces an enumerable, backed with any arbitrary
generation logic.

[1]: http://docs.ruby-lang.org/en/2.3.0/Enumerator.html

I can refactor the while loop to look more like this:

```ruby
def retrieve_all_pages(url)
  Enumerator.new do |yielder|
    while url
      response = connection.get url
      json     = JSON.parse response.body
      url      = json["links"]["next"]

      Array(json["widgets"]).each do |widget|
        yielder << widget
      end
    end
  end
end
```

We've gotten rid of the accumulator array, and instead have something
that looks much closer to idiomatic ruby.  Additionally, the method begins
yielding immediately after fetching the first page, only retrieving additional
pages when needed, without client code needing to understand the mechanics of
the underlying pagination.

It is important to note that instead of returning an `Array`,
`retreive_all_pages` now returns an `Enumerable`, but generally quacks the same
-- it's rather unlikely that any client of the original implementation was
using direct array semantics; if so, a simple `to_a` converts an Enumerable to
a normal Array.

Overall though, I find the resulting enumerator to be far easier to work with
in standard ruby, providing a more flexible and versatile encapsulation of the
remote interface.


