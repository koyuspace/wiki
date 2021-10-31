---
title: Setting up a dev environment
description: Instructions on how to start developing for koyu.space.
---

# Setting up a dev environment

## Pre-requisites

* A machine running **Ubuntu 20.04** or later that you have root access to

### System repositorie

#### Node.js

```bash
curl -sL https://deb.nodesource.com/setup_12.x | sudo bash -
```

#### Yarn

```bash
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
```

### System packages

```bash
sudo apt update
sudo apt install -y \
  imagemagick ffmpeg libpq-dev libxml2-dev libxslt1-dev file git-core \
  g++ libprotobuf-dev protobuf-compiler pkg-config nodejs gcc autoconf \
  bison build-essential libssl-dev libyaml-dev libreadline6-dev \
  zlib1g-dev libncurses5-dev libffi-dev libgdbm-dev \
  redis-server redis-tools postgresql postgresql-contrib \
  yarn libidn11-dev libicu-dev libjemalloc-dev
```

### Installing Ruby

We will be using rbenv to manage Ruby versions, because it’s easier to get the right versions and to update once a newer release comes out.

```bash
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
cd ~/.rbenv && src/configure && make -C src
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec bash
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
```

Once this is done, we can install the correct Ruby version:

```bash
RUBY_CONFIGURE_OPTS=--with-jemalloc rbenv install 2.7.4
rbenv global 2.7.4
```

We’ll also need to install bundler:

```bash
gem install bundler --no-document
```

## Setup

Run following commands in the project directory `bundle install`, `yarn install`.

In the development environment, koyu.space will use PostgreSQL as the currently signed-in Linux user using the `ident` method, which usually works out of the box. The one command you need to run is `rails db:setup` which will create the databases `mastodon_development` and `mastodon_test`, load the schema into them, and then create seed data defined in `db/seed.rb` in `mastodon_development`. The only seed data is an admin account with the credentials `admin@localhost:3000` / `mastodonadmin`.

> Please keep in mind, by default koyu.space will run on port 3000. If you configure a different port for it, the generated admin account will use that number.

If `rails db:setup` gives you the Postgres error:

    ActiveRecord::NoDatabaseError: FATAL:  role "your_user_name" does not exist

(where `your_user_name` is your username), then run:

    sudo -u postgres createuser your_user_name --createdb

This will create the necessary Postgres user with the permission to create a database.

## Running

There are multiple processes that need to be run for the full set of koyu.space functionality, although they can be selectively omitted. To run all of them with just one command, you can install Foreman with `gem install foreman --no-document` and then use:

```text
foreman start
```

In the koyu.space directory. This will start processes defined in `Procfile.dev`, which will give you: A Rails server, a Webpack server, a streaming API server, and Sidekiq. Of course, you can run any of those things stand-alone depending on your needs.

## Testing

| Command | Description |
| :--- | :--- |
| `rspec` | Run the Ruby test suite |
| `yarn run test` | Run the JavaScript test suite |
| `rubocop` | Check the Ruby code for conformance with our code style |

