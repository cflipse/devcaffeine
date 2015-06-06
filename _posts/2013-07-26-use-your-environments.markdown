---
title: Use your environments
layout: post
comments: true
permalink: /:year/:month/:day/:title/
excerpt_separator: <!--more-->
categories: rails
---

I may be repeating something you already know, but this is not something I see
talked about a lot, and I've run into this more times than I should.

Every so often, when I'm combing through a Rails codebase, I'll find code that
looks something like this:

```ruby
def some_method
  if Rails.env.production?
    # Do something for real
  else
    # Do something fake
  end
end
```

This might be tied up in a Twitter callback, or it may be performing some other
sort of expensive network computation.  Or, it might be a switch to turn off a
feature while it's still in development.  There are good reasons for applying
this pattern ... but the _way_ it's applied leaves much to be desired.

<!--more -->

Tests:  In order to test something like this, you have to lie to your test
environment -- usually through a stub:

```ruby
describe "#some_method" do
  context "in production" do
    before { Rails.env.stub(:production?  => true) }
    it "does something"
  end
end
```

This isn't _necessarily_ all that bad; stubbing is a powerful tool in your
testing repertoire.  However, depending on what level you're testing at, this
could lead to some surprises.  You're basically tricking your application code
into believing it's running in a production environment ... without having done
any of the production setup.  This _could_ work perfectly fine, or it could
lead to nasty surprises and lengthy debugging sessions.

There's another downside as well.  Say we add a new deployment environment,
staging.  There are some things we need turned on in staging, and some things
we need turned off.  So, we chase through the code, looking for env checks,
and now they look like:

```ruby
if Rails.env.production?  || Rails.env.staging?
  # do something
end
```

Except, of course, not all of the checks will be looking for both environments.
Is that an oversight?  Is that because a feature shouldn't be on in staging?  Or
is it not yet ready for production?  Add a couple of months for the development
work to fade from memory, and the headaches are huge.  Plus, when we're finally
ready to deploy, we have to chase down and modify all of those environment
checks -- and hope that some over-enthusiastic DRY refactoring didn't clump two
unrelated features behind the same environment check.  Don't forget to update
the tests!


## Feature flags

The better way to set this up isn't all _that_ different -- it's just more
focused.  Instead of checking to see what environment is running, your code
checks to see if a particular feature is enabled:

```ruby
def some_method
  if config.foo_enabled?
    # do something for real
  else
    # do something fake
  end
end
```

The tests even look similar:

```ruby
describe "#some_method" do
  context "when foo is enabled" do
    before { config.stub(:foo_enabled? => true) }
    it "does something"
  end
end
```

It's a small difference, leaning on a feature flag rather than an environment
check.  And yet, the difference in flexibility is startling:

We no longer have to lie to the tests about what environment they're in;
instead of flipping _all_ the features as a side effect, we only enable the _one_
we're concerned with.

It's easy to enable in multiple different environments.  Instead of chasing down
environment checks in multiple locations, now we can just adjust the flag in the
new environment -- and suddenly it works, just as it did before.  No worries about
missed boolean checks, because your application code _hasn't changed_. That's a
desirable feature in a staging-to-production transition.

As a happy side effect, it also becomes trivially easy to see how a particular
environment is configured:  just look at `config/envrionments/production.rb`, and
everything that's enabled is set right there.  Creating a new staging
environment that acts like production "except"?  Copy the environment file, and
tweak what you need to.

Even a server-locked form of A/B testing becomes feasibly trivial:  tweak your
environment file to enable the feature on odd-numbered servers, or by peeking
at an `ENV` setting, or calling `rand` ... whatever your strategy is, it's now
_possible_, because you're triggering features individually by a method or variable,
rather than as a chunk, by environment.

## Config

I've been pretty vague on just what that configuration variable is.  In part,
that's because that's not really the important bit.  It could be a
configuration object built for your subsystem, it could be a constant (though I
wouldn't recommend that) or it could be flags in a hash.  The important part is
that it is effectively global -- you enable the setting once, and the rest of
the code behaves accordingly.

In a rails app, you could do worse than to hang it off of the `Rails.config`.
You'll want to set your flags in the config/environments files anyway, and it's
something that's generally available to most of your application.  Since Rails3,
the config object is open-ended as well, making it an easy possibility for nested
flags consistent with the rest of your environment files:

```ruby
My::Application.configure do
  config.enable_foobar = true
end
```

If you want nested namespace configurations, like what most of rails has, you
can do it like so:

``` ruby
My::Application.configure do
  config.foo = ActiveSupport::OrderedOptions.new
  config.foo.bar_enabled = true
  config.foo.baz_enabled = false
end
```

It's a little more verbose, but you do get some consistency with the rest of
the internal configuration flags that rails uses.  In your app, you can access
these configs in a couple of ways:

```ruby
My::Application.config.foo_enabled
Rails.application.config.foo_enabled
```

That's all it takes -- one small level of indirection.  We've given our flag a
name, and set the one place in the application to have the responsibility of
setting it's value.  A seemly minor change like this can pay great dividends in
flexibility, cohesion, and comprehensibility.
