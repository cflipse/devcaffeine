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
# config/initializers/omniauth.rb

Rails.application.config.middleware.use OmniAuth::Builder do
  provider :developer unless Rails.env.production?
end
```

[step one]

[Omniauth]: https://github.com/omniauth/omniauth/blob/master/README.md
