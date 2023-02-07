Setting up a MS development environment is a bit involved, but it's not too bad, and it works well once everything is set up. If you run into problems, feel free to ask for help in [Charcoal HQ](https://chat.stackexchange.com/rooms/11540/charcoal-hq).

# Prereq: Supported Platforms
metasmoke can be installed and will work on the following platforms:

 - Ubuntu
 - Debian
 - macOS
 - probably most Linux flavors

metasmoke _does_ work on WSL, but it's significantly slower and takes much more effort in terms of solving the random errors you get from WSL's lack of support for things. You'll need a Windows build over 16170; builds before that [don't support mdns](https://github.com/Microsoft/WSL/issues/2245#issuecomment-310546134), which crashes the server on boot.

If you can make metasmoke work on Windows, you're a better developer than most of us, but Windows is not and will not be officially supported.

<!-- language-all: lang-none -->

# Install dependencies

### Mac: 

    brew install mysql
    brew services start mysql

### Linux:

    curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
    echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
    sudo apt update
    wget https://repo.mysql.com//mysql-apt-config_0.8.24-1_all.deb
    sudo apt update
    sudo apt install git mysql-client mysql-server libmysqlclient-dev libsqlite3-dev yarn

## Install NodeJS (no macOS instructions yet)

```
curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash - 
sudo apt-get install nodejs 
```

## Install Ruby

The easiest way to install Ruby is with [RVM](https://rvm.io). To install Ruby 2.5 (other ruby versions may work, but metasmoke runs on 2.5.0 as of 3/9/2018):

    $ gpg2 --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
    $ curl -sSL https://get.rvm.io | bash -s stable
    $ source ~/.rvm/scripts/rvm
    $ rvm instal ruby-2.7.4

You can also use a less popular alternative, [rbenv](https://github.com/rbenv/rbenv#basic-github-checkout) with [ruby-build](https://github.com/rbenv/ruby-build#installation):

    $ git clone https://github.com/rbenv/rbenv.git ~/.rbenv
    $ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
    $ ~/.rbenv/bin/rbenv init

Now follow the instructions. They will typically involve appending something to your `.bash_profile` or other shell config file. Then, install ruby build with:

    $ mkdir -p "$(rbenv root)"/plugins
    $ git clone https://github.com/rbenv/ruby-build.git "$(rbenv root)"/plugins/ruby-build

Then actually install ruby:

    $ rbenv install 2.5.0

# Clone metasmoke

To clone using HTTPS:

    $ git clone https://github.com/Charcoal-SE/metasmoke

To clone using SSH:

    $ git clone git@github.com:Charcoal-SE/metasmoke

# Install metasmoke dependencies

    $ cd metasmoke
    $ gem install bundler:1.17.3
    $ bundle install
    $ yarn install



If there are any errors with the previous step, you will need to correct those before proceeding. The most common one is with MySQL and is usually solved by running and then repeating the previous step

    sudo apt-get install libmysqlclient-dev
    gem uninstall mysql2
    gem install mysql2

# Set up redis

Since most package managers have old versions, you'll need to maually build redis from git. Fear not, there are no dependencies, and it usually builds in under 5 minutes. You'll want to be in the `metasmoke` directory, which you should already be in from the last step (if not, `cd metasmoke`):

    $ git clone https://github.com/antirez/redis redis-git
    $ cd redis-git
    $ make

And, if you feel extra careful, you can run the tests. Some tests may fail if you try to use multiple jobs, so don't do that.

    $ sudo apt-get install tcl
    $ make test

Then, you'll need to build our redis module. You'll want to be in the `metasmoke` directory, so if you're continuing from the last step, `cd ..`:

    $ git clone https://github.com/Charcoal-SE/redis-zhregex zhregex
    $ cd zhregex

The module depends on the c pcre library, which can be installed on macOS with:

    $ brew install pcre

or on ubuntu with

    $ apt-get install libpcre3-dev

On other platforms, you'll need to figure out some way to get it installed. 

Then, build the module:

    $ make

Now you've built everything you'll need for redis. We'll come back to run it in a later step.

## Getting a data dump

This step is technically optional, but metasmoke will not work correctly without it. You really should use both a database dump and a redis dump when getting a metasmoke instance set up.

Download a redis dump from https://metasmoke.erwaysoftware.com/dumps, and unzip it to `"#{Rails.root}/dump.rdb"`. That's where redis will look for it, in the root of your rails app with the name "dump.rdb".

## Adding redis to PATH (optional)

You may want to add `redis-git/src/redis-cli` and `redis-git/src/redis-server` to your PATH variable so that you can run the correct version of the CLI/server without prefixing it with all that mess. This step is fully optional, just makes your life easier if you're working closely with redis.

# Set up a database

    $ mysql -u root -p

You will be prompted for your MySQL root password; this should be the same password you entered during installation. (When I tried it, MySQL refused to authenticate me; [this Stack Overflow answer](https://stackoverflow.com/a/46908573/3476191) fixed it for me).

In the MySQL command line:

    mysql> CREATE DATABASE metasmoke;
    mysql> CREATE DATABASE metasmoke_test;
    mysql> CREATE DATABASE metasmoke_production;
    mysql> CREATE USER metasmoke@localhost IDENTIFIED BY '<password to set for the metasmoke database>';
    mysql> GRANT ALL PRIVILEGES ON metasmoke.* TO metasmoke@localhost;
    mysql> GRANT ALL PRIVILEGES ON metasmoke_test.* TO metasmoke@localhost;
    mysql> GRANT ALL PRIVILEGES ON metasmoke_production.* TO metasmoke@localhost;
    mysql> FLUSH PRIVILEGES;

Now exit the MySQL command line using ^D or `exit`.

# Configure metasmoke to use the newly-created database

    $ cd config
    $ cp config.sample.yml config.yml
    $ cp database.sample.yml database.yml

Now edit `database.yml`, and change it to the following:

    default: &default
      adapter: mysql2
      database: metasmoke
      encoding: utf8
      username: metasmoke
      password: <the metasmoke database password you set in the last step>
      host: 127.0.0.1
      port: 3306

    development:
      <<: *default

    # Warning: The database defined as "test" will be erased and
    # re-generated from your development database when you run "rake".
    # Do not set this db to the same as development or production.
    test:
      <<: *default
      database: metasmoke_test
    
    production:
      <<: *default
      database: metasmoke_production

Change the `default` section to have the same contents as well.

If you want integrations to work, you will need to update `config.yml` with real credentials for SES, Github, and/or StackExchange. The redirect URLs in the Stack Exchange settings should point back to your instance. This is not necessary, unless you're trying to work on a feature which requires them (almost never).

Now `cd` back up out of `config` and run:

    $ rails db:create
    $ rails db:schema:load
    $ rails db:migrate # in case the schema is out of date. it shouldn't be, but you never no.
    $ rails db:seed

If this fails, run `rails db:drop` and then run the previous commands again.

---

At this point, you should be good to go! Run `rails c` to get a REPL, and `foreman start` to launch a server on `localhost:5000`.

To create an account, start the server, and click "sign up" in the top bar.  Email doesn't matter, pick a username, and pick a password (security doesn't really matter). To give yourself admin privileges, open a console with `rails c` and run:

    User.where(username: "<the username you picked>").first.add_role :developer
    User.where(username: "<the username you picked>").first.add_role :admin

(There will be a significant amount of output from Rails after each of these commands, showing how the roles table is updated with a new role, and then the user is assigned this role.)

You can then go into Tools->User Permissions and grant yourself all the roles you'd like.

# Appendix: Required Versions
This is a list of known version requirements of various software that metasmoke requires, usually derived from required features or bugs in other versions.

 - WSL: Windows >= 16170 ([mdns](https://github.com/Microsoft/WSL/issues/2245#issuecomment-310546134))
  - This is obviously only an issue if you are actually using WSL to provide a Linux-compatible platform on Windows
 - Ruby: >= 2.3 ([SNO](http://mitrev.net/ruby/2015/11/13/the-operator-in-ruby/))
 - MySQL: >= 5.7.6 ([`CREATE USER IF NOT EXISTS`](https://stackoverflow.com/a/16592722/3160466))

---
<sup>Adapted from [Setting up metasmoke development](https://chat.stackexchange.com/rooms/11540/conversation/setting-up-metasmoke-development). Tested on macOS and Linux.</sup>
