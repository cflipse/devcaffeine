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

Omniauth is a ruby gem for interacting with multiple OAuth providers.
It is one of the available backends for devise, and provides adapters for
working single-signon-providers such as Google, Facebook, Twitter and Github.

Since my app is going to be connecting to a google-hosted provider, I'm going
to use that; other strategies an easily be added.

```ruby
# Gemfile

gem "omniauth"
gem "omniauth-google-oauth2"
```




[step one]
