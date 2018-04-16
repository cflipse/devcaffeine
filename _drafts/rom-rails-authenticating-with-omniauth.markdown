---
layout: post
title: "Rom-rails Authenticating With Omniauth"
date: 2018-04-07T15:19:10-04:00
tags:
  - rom
  - rails
  - tutorial
  - omniauth
---

This is the second step of a walkthrough of setting up a rom/rails project.
The goal here is to add and configure an omniauth integration, pulling and
storing user authentication data.  I'll also show how to restrict authentication
to a particular domain.

This is a follow-on to [step one] where we initialize and configure a new rails
app with rom-rails.


## Installation

[Omniauth] is a ruby gem for interacting with multiple OAuth providers.
It is one of the available backends for devise, and provides adapters for
working single-signon-providers such as Google, Facebook, Twitter and Github.

```ruby
# Gemfile

gem "omniauth"
gem "omniauth-google-oauth2"
```

I'm going to be using the google authentication strategy because the site I'm
building happens to handle it's email via google apps, I'll be able to wire up
the application to only accept auth requests from my specific domain.  If I
were writing a more open application, I could also add authentication
strategies to pull from Facebook, Twitter, Github, or even (for those who
prefer to avoid centralized accounts) a locally stored username and password.

For a start, though, I'm just going to wire up the `developer` strategy.

## Connecting authentication

I'll use an initializer to configure my omniauth middleware:

```
# config/initializers/authentication.rb

Rails.application.config.middleware.use OmniAuth::Builder do
  provider :developer unless Rails.env.production?
end
```

From here, if we start the Rails server, we can hit `/auth/developer` and 
receive a prompt for name and email.  What we get back is the _absolute basic_
amount of detail provided by OAuth, sent to a callback controller.  We'll need
to actually _do_ something with that, since we haven't yet created a controller.

Generally, the [Omniauth] readme serves as a good guide here.  We'll create a
`sessions` controller, and then use that to register authentication
information.  For the purposes of checking out what's going on, our `create`
method is, for the moment, going to skip the usual post-and-redirect pattern.
Worry not, this will change later.

```
# config/routes.rb
Rails.application.routes.draw do
  # ...
  match "auth/:provider/callback", to: "sessions#create", via: %i[get post]
end


# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  protect_from_forgery except: "create"
  def create
    @auth = request.env['omniauth.auth'].to_h
  end
end

# app/views/sessions/create.html.erb
<pre>
<%= @auth %>
</pre>
```

A run through this controller produces:

```
{"provider"=>"developer", "uid"=>"cflipse@example.com", "info"=>#<OmniAuth::AuthHash::InfoHash email="cflipse@example.com" name="Chris Flipse">, "credentials"=>#<OmniAuth::AuthHash>, "extra"=>#<OmniAuth::AuthHash>}
```

For the full list of what's possible under `info`, `credentials` and `extra`, 
we can look at the [schema](https://github.com/omniauth/omniauth/wiki/Auth-Hash-Schema).
It's worth noting that _only_ the `provider`, `uid` and `info[name]` keys are
required.  Literally everything else is gravy.  When we add providers, it's
important to check and see what is given.


## The simplest thing

The absolute simplest thing that could possibly work:

```
# app/controllers/application_controller.rb
helper_method :current_user, :logged_in?

def current_user
  @current_user ||= session[:current_user]
end

def logged_in?
  current_user.present?
end

def login_required
  redirect_to new_session_url unless logged_in?
end


# app/controllers/sessions_controller.rb

def create
  session[:current_user] = request.env['omniauth.auth'].uid
  redirect_to "/"
end
```

`login_required`, when used as a filter, checks to see if there is a current
user, which translates to looking for a session key.  If one isn't found,
bounce to `new_session_url`, which will be a simple page linking to the
different OAuth authentication endpoints we'll support.

There's nothing _necessarily_ wrong with this, but it lacks permanence.
Authenticated users exist only within the session; we're forced to use
the uid parameter, which _may not_ be the same between providers; there's no
way to realize that a twitter auth and a gmail auth are the same identity ...
it's completely ephemeral. We want better tracking, and better control. Plus,
there's that profiles table we created last session ...

[step one]

[Omniauth]: https://github.com/omniauth/omniauth/blob/master/README.md
