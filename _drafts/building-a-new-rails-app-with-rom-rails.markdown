---
layout: post
title: "Building a New Rails App With Rom-rails"
categories:
tags:
  - rom
  - rails
  - tutorial
---

This is step one of a walkthrough for building a new app with `rom-rails`.
I'll talk about my justifications and philosophy in another post; my intention
here is to walk through the initial creation of a `rom-rails` application.

The end goal of this application is to serve as a file repository for use with
another site.  As I work throgh building the app, I'll list out followons for
some specific subjects of interest.  _This_ post highlights the initial
generation and through to pulling some 'hello world' content from a data store.

Some things will look simliar to the normal rails setup, but there are some
variations that I'll be illuminating here.  As I've worked with Ruby and
Rails for a long, long time, I (like Rails itself) Have Opinions, and since
this is my blog, I'll not be shy about sharing them. Hopefully, some of that
will serve as jumping off points for posts I've been meaing to write for ages.

<!-- more -->

## Step the first: Generation

The stock Rails generator is what we'll use here, with a few options specified:

```shell
flip@kona:~/src$ rails new warehouse --skip-active-record  --skip-spring --skip-coffee --skip-test
      create
      create  README.md
      create  Rakefile
# ... gobs of output elided

flip@kona:~/src$ cd warehouse/
flip@kona:~/src/warehouse (master #)$ git add .
flip@kona:~/src/warehouse (master +)$ git commit -m 'initial app generation'
```

I've skipped activerecord for what I hope are obvious reasons.  `coffeescript`
has never been a JS flavor I'm wild about, and I suspect there are better
options these days.  I've skipped minitest so that I can install rspec.

Skipping `spring` is a big one.  I consider it a band-aid that often causes more
problems than it helps with. ROM's interctions with `rails` autoloading is
still finiky at best, and adding _another_ layer of load confusion ... as
maintainer of `rom-rails`, I've done basically nothing to support `spring`.  If
it's a thing you need, contributions are encouraged.

<aside markdown='block'>
At this point, because I have rubocop installed globally, I'm going to copy
in my default configurations and add `rubocop-rails` to get my preferred
defaults. Nothing to see here, so let's move along... For now.  There are
points where ROM standards and rubocop's defaults disagree; I'll try to
remember to call those out along the way.
</aside>

We now have an intial app generated. At present, it's nothing a web router and
renderer; we have no storage, and no tests (because rails won't generate rspec
installs).  Let's fix that:

```ruby
gem "pg"
gem "rom-rails"
gem "rom-sql"
gem "rom-repository"

group :development, :test do
  gem "dotenv-rails"
  gem "rspec-rails", "~> 3.0"
end

```

* `dotenv` is useful for loading environment variable configurations from
 configuration files; we'll use it in development and test to configure
things, while production will have to rely on _real_ environment configurations.

* `rspec-rails` because I have strong affinity for rspec's testing philosophies,
and every test::spec suite I've ever seen has expended great effort on
reinventing things from rspec, poorly.

* `pg` because postgres.  We're going to be using some embedded JSON down
the way, and postgres makes this easy to work with.  Ordinarily, the rails
generator allows you to specify via `-d postgres` but since
 we `--skip-active-record`, the rails installer ignores our database preference.

* `rom-rails` gives us basic rom dependencies, integration with rails, and a
few handy generators.  It does _not_ assume which backends we're going to use,
and so:

* `rom-sql` implements an SQL adapter and relation, which we will use to
access our database.  There are other adapters available, covering things from
cassandra to yaml to REST api calls.  At first, we're simply interested in our
database.

* `rom-repository` implements a nicer interface on top of the rom relation
backends.  Repositories serve as a fantastic place to translate between your
storage implementations and the language of your domain.


We'll finish off the gem installations by generating our rspec configuration
and comitting:

```shell
flip@kona:~/src/warehouse (master)$ bin/rails g rspec:install
      create  .rspec
      create  spec
      create  spec/spec_helper.rb
      create  spec/rails_helper.rb
```


## Configuring rom-rails

The next part is, unfortuntely, very twitchy.  Ordering is important, or
exceptions will be thrown.

First, ensure your databases have been created.  I'm going to wave the flag
and call configuring your postgres install out of scope for this tutorial;
at the end, you want to have both a test and development instance accessible.

Here's the commands I threw locally:

```
flip@kona:~/src/warehouse (master)$ createdb warehouse_development
flip@kona:~/src/warehouse (master)$ createdb warehouse_test
```

Next, we'll add some environment configuration:

```shell
# .env.development
DATABASE_URL=postgres://localhost/warehouse_development
```

```shell
# .env.test
DATABASE_URL=postgres://localhost/warehouse_test
```

`rom-sql` generally prefers to use a connection\_uri string for determining
how to connect to your database.  If you specified a username and password
then the string will look more like: `postgres://user:pass@localhost/warehouse_development`

To access migration generators[^1], add this line to your rakefile:

```ruby
# Rakefile
require "rom/sql/rake_tasks"
```

The migration syntax is using the `Sequel` gem; nothing changes from there.

Finally, we generate our configurations and commit:

```
flip@kona:~/src/warehouse (master)$ bin/rails g rom:install
      create  config/initializers/rom.rb
      create  lib/types.rb
      create  app/models/application_model.rb
```

This last step generates the configurations necessary to tell rom about it's
database gateways.  The generator _could_ be smarter; at the moment,
it _assumes_ that there is an `sql` adapter loaded and that you'd like for
it to be the default gateway. `config/initializers/rom.rb` is the file to edit
if you are adding additional gateways, or if you need to set any additional
configurations.


Finally, we can verify see if everything is wired by running our specs:

```
flip@kona:~/src/warehouse (master +)$ bundle exec rspec -rrails_helper
No examples found.


Finished in 0.00039 seconds (files took 1.3 seconds to load)
```

If everything has been wired correctly, and the database exists, than the specs
will run.  If an exception is thrown, then either ROM does not know where the
database can be found (check your `.env`s) or the database does not exist.

## Adding Profiles

As a final step, we're going to add a profiles table to the database,
add some content to it, and display it in a barebones controller.

First, we create the migration:

```shell
flip@kona:~/src/warehouse (master)$ bin/rails db:create_migration[create-profiles]

<= migration file created db/migraate/20180413141632_create-profiles.rb
flip@kona:~/src/warehouse (master)$
```

This creates a new migration file in the familiar timestamped form. The syntax
of the migration [comes from `Sequel`][migration]:

```ruby
ROM::SQL.migration do
  change do
    create_table :profiles do
      primary_key :id

      column :email, String, null: false, unique: true, index: true
      column :display_name, String

      column :private_email, TrueClass, default: true

      column :bio, :text
      column :state, String, default: 'active', index: true
    end
  end
end
```

running the familiar `rake db:migrate` will create our table.

Now, let's generate a relation and a repository so we can access the table:

```
flip@kona:~/src/warehouse (master)$ be rails g rom:relation profiles
      create  app/relations/profiles_relation.rb
flip@kona:~/src/warehouse (master)$ bin/rails g rom:repository profiles
      create  app/repositories/profile_repository.rb
flip@kona:~/src/warehouse (master)$
```

Now if we crack open the console, we'll be able to add some quick test data to the table:

```irb
irb(main):001:0> profiles = ProfileRepository.new(ROM.env)
=> #<ProfileRepository struct_namespace=Warehouse auto_struct=true>
irb(main):004:0> profiles.create([
{email: 'test@example.com', display_name: "Test Guy"},
{email: 'old@example.com', display_name: "Old profile", state: 'disabled'}
])

=> #<Warehouse::Profile id=1 email="test@example.com" display_name="Test Guy" private_email=true bio=nil state="active">
irb(main):005:0> ROM.env.relations[:profiles].count

=> 2
irb(main):006:0>
```

By default, there are no reader methods defined in a repository; we want to
tune those specficially for our project domain, something I will go over in a
later post.

## Next steps

This is the bare bones; The initial app configuration is done, but there
is not yet anything useful going on. In the next step, we're going to
demonstrate commands by adding authentication to the application.


[^1]: I _definatly_ need to fix this in the rom-rails code; it should be part
of the normal installation, or even pushed into the "normal" generator flow.

[migration]: https://github.com/jeremyevans/sequel/blob/master/doc/schema_modification.rdoc
