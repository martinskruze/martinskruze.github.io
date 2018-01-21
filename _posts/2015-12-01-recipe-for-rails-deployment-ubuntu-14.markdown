---
layout: post
title:  recipe - Deploy Rails at Ubuntu 14.04. (Dec 2015)
date:   2015-12-01 11:00:00 +0200
categories: recipes
---

## THIS WON'T WORK ANYMORE
This Won't Work at later Ubuntu versions and since 2018 mina setup too needs to be changed: [mina migrating guide](https://github.com/mina-deploy/mina/blob/master/docs/migrating.md).

## Ubuntu 14.04., GIT, Ruby on Rails, PostgreSQL, Ngnix, Unicorn, Mina
As general rule - if action is sudo - it should be done by user that can do sudo actions. You can make users that does not do sudo actions as user under which this application will run, but then just do all non-sudo actions under that user!

### some sites from which this is compiled modified
https://coderwall.com/p/yz8cha/deploying-rails-app-using-nginx-unicorn-postgres-and-capistrano-to-digital-ocean
https://olmonrails.wordpress.com/2008/08/12/switching-rails-to-postgresql/
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-rails-app-with-unicorn-and-nginx-on-ubuntu-14-04


### dns side
1. point your domain to server :)

### your local side
1. In gemfile:
```bash
gem "mina"
```
2. In console:
```bash
bundle install
```
3. This will create config/deploy.rb file. In console:
```bash
mina init
```

### server side
(example for Ubuntu Server with running Nginx (if you haven't got Nginx just install it after step 3))
1. update & upgrade the machine
```bash
sudo apt-get update
sudo apt-get upgrade
```
2. install node.js (system will need it)
```bash
sudo apt-get install nodejs
```
3. install other usefull (read it - you will need it for succesfull install or maybe you will need it someday) stuff
```bash
sudo apt-get install build-essential bison openssl libreadline6 libreadline6-dev curl git-core zlib1g zlib1g-dev libssl-dev libyaml-dev libxml2-dev autoconf libc6-dev ncurses-dev automake libtool libgmp-dev libsqlite3-dev sqlite3 libgdbm-dev pkg-config libffi-dev
```

### server side in same user that runs webapps (may be not rooted)
1. install RVM on your machine: follow instructions: [RVM page](https://rvm.io/) and read what it says when installing
2. reload RVM and install ruby
```bash
rvm reload
rvm install [your verson number here] e.g. rvm install 2.2.2
```
3. check for rubies in yor RVM list (just for hek of it)
```bash
rvm list
```
4. create your projects gemset
```bash
rvm gemset create [your_project_name]
```
5. switch to your gemset
```bash
rvm use [your_ruby_version]@[your_project_name]
```
4. install bundler and rails (or with your specific rails version)
```bash
gem install bundler rails --no-rdoc --no-ri
```
5. install unicorn
```bash
gem install unicorn
```

### in your app
1. add unicorn gem to Gemfile
```bash
  gem 'unicorn'
```
```bash
  bundle install
```
2. create/edit file config/unicorn.rb ([unicorn.rb exapmle](../post_files/2015-12-01-recipe-for-rails-deployment-ubuntu-14-unicorn.rb))
3. edit config/deploy.rb ([deploy.rb exapmle](../post_files/2015-12-01-recipe-for-rails-deployment-ubuntu-14-deploy.rb))

### server side
1 configure nginx (with example in your_apps_domain_nginx_configuration)
  sudo vim /etc/nginx/sites-available/[your_apps_domain]
  sudo ln -s /etc/nginx/sites-available/[your_apps_domain] /etc/nginx/sites-enabled/[your_apps_domain]
2 restart nginx
  sudo service nginx restart
3 if everything is ok so far (and you have pointed dns records to your server), you should go to your page example.com and see
  502 Bad Gateway
4 unicorn starting/stoping and restarting script #> in this file edit and copy/paste from example unicorn_your_app_name script
  sudo vim /etc/init.d/unicorn_[your app name]
5 add permisions and make it start stop as it should do
  sudo chmod 755 /etc/init.d/unicorn_appname
  sudo update-rc.d unicorn_appname defaults
6 create place for sockets
  sudo mkdir /var/sockets
  sudo chmod 777 /var/sockets/

### on server
  mkdir [your app path]
  chown -R username [your app path]

### on local machine
1 set up authorised keys (https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2)
  ssh-copy-id [your login]@[your server IP or app domain]

### on server
1 if private github repo - you will need to add ssh key from your server to github (https://help.github.com/articles/generating-ssh-keys/)

### in your app
mina setup

### on server
1 clarify your rvm path
  rvm info
2 take from printout start info - usually "/home/[your user]/.rvm/" and add "/scripts/rvm" and that will be your path
3 install postgres
  sudo apt-get install postgresql postgresql-client postgresql-contrib libpq-dev
  gem install pg -- --with-pg-config#/usr/bin/pg_config
4 create new postgres user
  sudo -u postgres psql
  create user [username] with password '[password]';
  alter role [username] superuser createrole createdb replication;
  create database [projectname]_production owner [username];
5 exit from psql
  \q

### on server
1 edit database file in [your app path]/shared/config/database.yml
2 edit your .bashrc file and add line
  source "$HOME/.rvm/scripts/rvm"
3 restart server
  sudo reboot now

### in your app
  mina deploy

## if you need stop/start unicorn without service
### in project current directory on server
  ps ax | grep unicorn
  kill -QUIT [first process number]
  RAILS_ENV#production bundle exec unicorn -D -c config/unicorn.rb

### some errors
#### Paperclip::Errors::CommandNotFoundError (Could not run the `identify` command. Please install ImageMagick.):
  sudo apt-get install imagemagick

### some extra stuff
#### to dump database
  pg_dump db_name -U username -h localhost -F c
#### to restore db from dump
  pg_restore --verbose --clean --no-acl --no-owner -h myhost -U myuser -d mydb latest.dump
#### to restore db if dumped as text file
psql -h localhost -d database_name -U db_username -f filename.sql
