---
layout: post
comments: true
permalink: /:year/:month/:day/:title/
categories: ruby
---

I'm working on a rails project where I'm experimenting with a much more
object-oriented approach than Rails usually encourages.  I'm intending on
writing more in depth about that, when I can better gather my thoughts about
the experience. I found something recently that I can write a brief post
about and hopefully get the ball rolling.

I'm trying to keep a fairly hard-line divide between my persistence layer and
the actual domain objects.  What this means for today's purposes is that my
domain objects know nothing about ActiveRecord (and some of them never will).

This lead me to a philisophical quandry when trying to figure out validations.
My domain objects are already imbuded with ActiveModel attributes, because 
they communicate with the view layer in ActionPack. (ActiveModel is,
essentially, the reference API that ActionPack expects.) However, validations
can cross both persistence and domain concerns.  I've also been pretty heavily
burned in the past by the mess that is conditional validations.

In debating whether to embed my validations in the domain model, or in the
ActiveRecord objects that my Repositories map the domain entites to, I realized
that I had a third option:  Separate the validations.  Structuring the
validations as their own class opens up a lot of the traditional OO flexibility
that I've been aiming for in this application, and allows me to ignore the
validations in situtations where they are not needed, without a lot of
stub/mock gymnastics.

<!--more -->

After a little bit of experimentation, I found that we're even able to
structure the class so it uses the validation DSL we're all used to from within
ActiveRecord.  There are a couple of rough edges in the code below, but it
gives a good run at a proof of concept.


First, the domain model.  Most of this is a straightforward copy of "minimal"
required by the documentation in
[ActiveModel::Errors][1]

{% gist cflipse/2961010 Post.rb %}

Next, we can build a validation.  In this case, I want to be able to validate
a post in two different states -- draft and published.  A published post requires
an author, a title, and a publication date: 

{% gist cflipse/2961010 published_post_validator.rb %}

while a draft post only requires a title,
but also requires that the publication date, if set, be sometime in the future:

{% gist cflipse/2961010 draft_post_validator.rb %}

These work pretty well, using a trick with [SimpleDelegator][2] where the
validation objects wrap and forward unknown methods, such as data accessors, to
the domain model.  Since mixing in [ActiveModel::Validations][3] provides
a `#valid?` method to the validator object, that is not forwarded.

{% gist cflipse/2961010 gistfile1.rb %}

Unfortunately, mixing in `ActiveModel::Validations` *also* adds a `#errors` method
to the validator, meaning that any errors found get added to the validator object,
instead of the domain object.  If, once wrapped and validated, we only ever pass the
wrapped domain object, then this is possibly fine.  I don't particularly want
to have to pass the whole validator package around, so I want the errors to go down
to the domain object.  This is the reason for the `__getobj__.errors` method in the
validators above.

Also worth noting is that the validators here only go one level deep; if we're
looking to wrap and decorate, we should probably throw a few `super`s onto a
couple of methods so that the validations will actually cascade.  I'm not
currently interested in nesting the validations, so I'll leave that as an
exercise for anyone interested.


[1]: http://api.rubyonrails.org/classes/ActiveModel/Errors.html
[2]: http://www.ruby-doc.org/stdlib-1.9.3/libdoc/delegate/rdoc/SimpleDelegator.html
[3]: http://api.rubyonrails.org/classes/ActiveModel/Validations.html
