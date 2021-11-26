---
title: Self-hosting
description: Instructional guide on creating your own koyu.space-powered website.
menu:
  docs:
    weight: 20
    parent: admin
---

# Self-hosting koyu.space

## Why would you want to run your own koyu.space server?

- Absolute control over your own voice on the web, not subject to anyone else's rules or whims. Your server is your property, with your rules. It will exist as long as you want it to exist.
- You are *not* isolated on your own server. You can follow anyone on any other server, and they can follow you and you can exchange messages just like if you were on the same server.
- You can either limit sign-ups to be the only one on the server and run it like personal (micro)blog, maintain an invite-only community for family or friends or run a server anyone can sign up on, it's up to you!

{{< hint warning >}}
Please mind that providing a public internet service involves moderation work and community management, and that such work becomes more complicated the larger your server grows.
{{< /hint >}}

## So you want to run your own koyu.space server

Here is what you need:

- A **domain name**. This is how you and others will access your server and how you and your users will be identified on the network.

  **How to get**: Namecheap, Gandi, any of the infinite number of domain name registrars. Comes with a yearly cost that varies depending on domain name choice.
- A **VPS**. Something that will run the koyu.space code that will always be connected to the internet.

  **How to get**: DigitalOcean, Hetzner, Exoscale, Scaleway, any of the infinite number of hosting providers. Comes with a monthly or yearly cost that varies depending on hardware specifications.
- An **e-mail provider**. koyu.space needs to send confirmation links and various notifications through e-mail, and hosting your own SMTP server, while possible, is much more difficult to do reliably than to simply use a third-party provider.

  **How to get**: Mailgun, SparkPost, Postmark, Sendgrid, any of the infinite number of e-mail hosting providers that expose a SMTP API. Comes with a monthly cost based on volume of e-mails sent.
- Optional: **Object storage provider**. koyu.space can save files that you and your users upload on the hard disk drive of the VPS it runs on, however, the hard disk drive is usually not infinite and difficult to upgrade later. An object storage provider gives you practically infinite metered file storage.

  **How to get**: Amazon S3, Exoscale, Wasabi, Google Cloud, anything that exposes either an S3-compatible or OpenStack Swift-compatible API. Comes with a monthly cost based on the amount of files stored as well as how often they are accessed.

That however does assume a single-machine setup. koyu.space scales quite well horizontally. If your needs outgrow the capacity of a single machine, koyu.space can be divided into multiple app servers, background workers, multiple Redis backends, PostgreSQL replicas.

If you're interested in installing everything on your own, proceed reading.

## Pre-requisites {#pre-requisites}

* A machine running **Ubuntu 18.04** or higher that you have root access to
* A **domain name** \(or a subdomain\) for the koyu.space server, e.g. `example.com`
* An e-mail delivery service or other **SMTP server**

You will be running the commands as root. If you aren’t already root, switch to root:

### System repositories {#system-repositories}

Make sure curl is installed first:

#### Node.js {#node-js}

```bash
curl -sL https://deb.nodesource.com/setup_14.x | bash -
```

#### Yarn {#yarn}

```bash
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list
```

### System packages {#system-packages}

```bash
apt update
apt install -y \
  imagemagick ffmpeg libpq-dev libxml2-dev libxslt1-dev file git-core \
  g++ libprotobuf-dev protobuf-compiler pkg-config nodejs gcc autoconf \
  bison build-essential libssl-dev libyaml-dev libreadline6-dev \
  zlib1g-dev libncurses5-dev libffi-dev libgdbm-dev \
  nginx redis-server redis-tools postgresql postgresql-contrib \
  certbot python-certbot-nginx yarn libidn11-dev libicu-dev libjemalloc-dev
```

### Installing Ruby {#installing-ruby}

We will be using rbenv to manage Ruby versions, because it’s easier to get the right versions and to update once a newer release comes out. rbenv must be installed for a single Linux user, therefore, first we must create the user koyu.space will be running as:

```bash
adduser --disabled-login mastodon
```

We can then switch to the user:

```bash
su - mastodon
```

And proceed to install rbenv and rbenv-build:

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
RUBY_CONFIGURE_OPTS=--with-jemalloc rbenv install 3.0.3
rbenv global 3.0.3
```

We’ll also need to install bundler:

```bash
gem install bundler --no-document
```

Return to the root user:

```bash
exit
```

## Setup {#setup}

### Setting up PostgreSQL {#setting-up-postgresql}

#### Performance configuration \(optional\) {#performance-configuration-optional}

For optimal performance, you may use [pgTune](https://pgtune.leopard.in.ua/#/) to generate an appropriate configuration and edit values in `/etc/postgresql/9.6/main/postgresql.conf` before restarting PostgreSQL with `systemctl restart postgresql`

#### Creating a user {#creating-a-user}

You will need to create a PostgreSQL user that koyu.space could use. It is easiest to go with “ident” authentication in a simple setup, i.e. the PostgreSQL user does not have a separate password and can be used by the Linux user with the same username.

Open the prompt:

```bash
sudo -u postgres psql
```

In the prompt, execute:

```sql
CREATE USER mastodon CREATEDB;
\q
```

Done!

### Setting up koyu.space {#setting-up-koyuspace}

It is time to download the koyu.space code. Switch to the mastodon user:

```bash
su - mastodon
```

#### Checking out the code {#checking-out-the-code}

Use git to download the latest stable release of koyu.space:

```bash
git clone https://github.com/koyuspace/mastodon --recurse-submodules live && cd live
```

#### Installing the last dependencies {#installing-the-last-dependencies}

Now to install Ruby and JavaScript dependencies:

```bash
bundle config deployment 'true'
bundle config without 'development test'
bundle install -j$(getconf _NPROCESSORS_ONLN)
yarn install --pure-lockfile
```

{{< hint info >}}
The two `bundle config` commands are only needed the first time you're installing dependencies. If you're going to be updating or re-installing dependencies later, just `bundle install` will be enough.
{{< /hint >}}

#### Generating a configuration {#generating-a-configuration}

Run the interactive setup wizard:

```bash
RAILS_ENV=production bundle exec rake mastodon:setup
```

This will:

* Create a configuration file
* Run asset precompilation
* Create the database schema

The configuration file is saved as `.env.production`. You can review and edit it to your liking. Refer to the [sample configuration file](https://github.com/koyuspace/mastodon/blob/main/.env.production.sample).

You’re done with the mastodon user for now, so switch back to root:

```bash
exit
```

### Setting up nginx {#setting-up-nginx}

Copy the configuration template for nginx from the koyu.space directory:

```bash
cp /home/mastodon/live/dist/nginx.conf /etc/nginx/sites-available/mastodon
ln -s /etc/nginx/sites-available/mastodon /etc/nginx/sites-enabled/mastodon
```

Then edit `/etc/nginx/sites-available/mastodon` to replace `example.com` with your own domain name, and make any other adjustments you might need.

Reload nginx for the changes to take effect:

### Acquiring a SSL certificate {#acquiring-a-ssl-certificate}

We’ll use Let’s Encrypt to get a free SSL certificate:

```bash
certbot --nginx -d example.com
```

This will obtain the certificate, automatically update `/etc/nginx/sites-available/mastodon` to use the new certificate, and reload nginx for the changes to take effect.

At this point you should be able to visit your domain in the browser and see the melting koyu.space icon error page. This is because we haven’t started the koyu.space process yet.

### Setting up systemd services {#setting-up-systemd-services}

Copy the systemd service templates from the koyu.space directory:

```sh
cp /home/mastodon/live/dist/mastodon-*.service /etc/systemd/system/
```

If you deviated from the defaults at any point, check that the username and paths are correct: 

```sh
$EDITOR /etc/systemd/system/mastodon-*.service
```

Finally, start and enable the new systemd services:

```sh
systemctl daemon-reload
systemctl enable --now mastodon-web mastodon-sidekiq mastodon-streaming
```

They will now automatically start at boot.

{{< hint info >}}
**Hurray! This is it. You can visit your domain in the browser now!**
{{< /hint >}}
