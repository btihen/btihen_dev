---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.0 Branca and Caching"
subtitle: "Organizing Rails Apps with Packages"
summary: ""
authors: ['btihen']
tags: ['Ruby', 'Packwerk', 'Architecture', 'Design']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2022-05-26T01:20:00+02:00
lastmod: 2022-05-26T01:20:00+02:00
featured: false
draft: true

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

## Organizing Code

Gems and Engines have long been used to organize Ruby and Rails code into workable small units.

Shopify has introduced a new system called 'packages' - they use the packwerk gem to help us.  In fact, it is designed to make it easy to take large (and likely highly coupled) large codebase and move toward 'packages' self-contained (or have explicit dependencies).  Ideally the code is in small enough units to help us keep the context in mind as we work on it.

I found it initially difficult to understand packwerk in the context of a complex codebase.  So instead I built a new 'play' app and then moved each piece into a package.  Hopefully, this will inspire you to use Gem, Engines or Packages to clarify dependencies and make the code a logical until that is easy to reason about.

## Environment

First you will need `libsodium` installed;
```bash
brew install libsodium
```

Using Rails 7 & Ruby 3.1.2 - I found that it is important to update my ruby environment - so before we start this is what I didn't remove errors:

```bash
# I've had the error several times without updating:
# /Users/btihen/.rbenv/versions/3.1.0/lib/ruby/gems/3.1.0/gems/bundler-2.3.8/lib/bundler/rubygems_ext.rb:18:in `source': uninitialized constant Gem::Source (NameError)
#
#       (defined?(@source) && @source) || Gem::Source::Installed.new
#                                            ^^^^^^^^
# Did you mean?  Gem::SourceList
# this seems to fix it:
# https://bundler.io/guides/bundler_2_upgrade.html
# https://stackoverflow.com/questions/4859600/bundler-throws-uninitialized-constant-gemsilentui-nameerror-error-after-upgr
rbenv local 3.1.2
gem update --system
gem install bundler
gem install rails
rbenv rehash
```

## Rails Project - Simple Blog

Since my other projects are using `esbuild` I use that here too

```ruby
rails new rails_branca -T --database=postgresql --css=bootstrap --javascript=esbuild
cd rails_branca
bin/rails db:create

# add the packwerk (packages) gem
bundle add branca-ruby
```

## Quick Test

Let's try this library in the console:
```ruby
bin/rails c

require 'branca'

# key must be at least 33 Characters long!
Branca.secret_key = 'supersecretkeyyoushouldnotcommit'.b
Branca.ttl = 30 # in seconds

token = Branca.encode 'btihen@example.com'
decoded = Branca.decode token
message = decoded.message
> 'btihen@example.com'

# now wait 20 Seconds and you should get the error
decoded = Branca.decode token
> Token is expired (Branca::ExpiredTokenError)
# now the token is encrypted and no longer usable


# Branca encodes strings - but we can convert to json and encrypt complex data
token = Branca.encode({email: 'btihen@example.com'}.to_json)
decoded = Branca.decode token
message = JSON.parse decoded.message
> {"email"=>"btihen@example.com"}
```

## Configure Rails

Let's setup rails to use Branca (and expire tokens after 15 Mins)

Let's get rails to read the env file (append dotenv-rails to the Gemfile):
```bash
# append gem to the GEMFILE with the >>
cat <<EOF>>Gemfile

gem 'dotenv-rails', groups: [:development, :test]
EOF
bundle
```

Now add the needed config files:
```ruby
# create the environmental variables
cat <<EOF> .env
BRANCA_SECRET='SuperSecretKeyHiddenInEnvFile'
BRANCA_TTL=900
EOF

# create the rails initializer for branca
touch config/initializers/branca.rb
cat <<EOF> config/initializers/branca.rb
# require is needed since the gem name is branca-ruby, but needs to load: branca
require 'branca'

Branca.configure do |config|
  # .b ensures 8 bit String Encoding
  config.secret_key = ENV['BRANCA_SECRET'].b
  config.ttl = ENV['BRANCA_TTL']
end
EOF
```

Lets be sure Rails loads Branca properly:
```ruby
bin/rails c
Branca.ttl
> "900" # hopefully :)

token = Branca.encode({email: 'btihen@example.com'}.to_json)
decoded = Branca.decode token
message = JSON.parse decoded.message
> {"email"=>"btihen@example.com"}
```

## Using Branca

## Resources

* branca - https://branca.io
* branca-ruby - https://github.com/thadeu/branca-ruby
* crossoverhealth-branca - https://github.com/c (didn't seem to work)

**Token Technologies**

* JWT - Common but be careful - https://jwt.io - (everything)
* Branca - Encrypted, simple & secure - https://github.com/thadeu/branca-ruby - (closure, .net, elixir, erlang, go, java, javascript, kotlin, php, python, ruby, rust)
* PASETO - Addresses security problems with JWT - https://paseto.io - (v3/v4: go, node, php, python, rust, swift) & (v1/v2: c, elixir, go, java, javascript, lua, .net, node, php, python, ruby, rust, swift) - https://dev.to/techschoolguru/why-paseto-is-better-than-jwt-for-token-based-authentication-1b0c
* Macaroon - better than cookies - http://macaroons.io & https://research.google/pubs/pub41892/ & https://github.com/localmed/ruby-macaroons & https://github.com/rescrv/libmacaroons - authorization with caveats and 3rd parties (c, .net, elixir, go, java, python, ruby, rust, php)
* Biscuit - has authentication dsl - https://www.biscuitsec.org & https://github.com/biscuit-auth/biscuit - (rust, web assembly, haskell, java, go)

## Rails Caching

* https://avdi.codes/using-hashes-as-caches/
* https://guides.rubyonrails.org/caching_with_rails.html
* https://www.honeybadger.io/blog/rails-low-level-caching/
* https://devcenter.heroku.com/articles/caching-strategies
* https://scoutapm.com/blog/a-complete-guide-to-rails-caching
* https://blog.appsignal.com/2018/04/17/rails-built-in-cache-stores.html
* https://www.toptal.com/ruby-on-rails/field-level-rails-cache-invalidation
* https://www.justinweiss.com/articles/a-faster-way-to-cache-complicated-data-models/

## Intersting Article

* using paseto - https://dev.to/techschoolguru/why-paseto-is-better-than-jwt-for-token-based-authentication-1b0c
* go course using paseto with Rest API - https://github.com/techschool/simplebank
