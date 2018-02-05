---
layout: post
title:  recipe - create simple Docker on Rails in development
date:   2018-02-01 15:45:00 +0200
categories: recipes
---
### setup
- You will need [Docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/) and [Docker Compose](https://docs.docker.com/compose/install/).
- This is for Ruby 2.5 and Rails 5.1.4 -> change it in any mentioned files, if you need another version
- The specific app's name is `app-name-here` -> change it in any mentioned files for your app's name

### general steps:
- create empty folder titled: `app-name-here`
- create `Dockerfile` inside your new folder:
```
FROM ruby:2.5
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev nodejs imagemagick && rm -rf /var/lib/apt/lists/*
RUN mkdir /app-name-here
WORKDIR /app-name-here
COPY Gemfile /app-name-here/Gemfile
COPY Gemfile.lock /app-name-here/Gemfile.lock
RUN bundle install
COPY . /app-name-here
```
- create temporary `Gemfile` inside your new folder (this will be overwritten by rails new):
```ruby
source 'https://rubygems.org'
gem 'rails', '5.1.4'
```
- create an empty `Gemfile.lock` inside your new folder
- create `docker-compose.yml` file inside your new folder:
```yml
version: '3'
services:
  db:
    image: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data
  web:
    build: .
    command: bundle exec rails s -p 3000 -b '0.0.0.0'
    volumes:
      - .:/app-name-here
    ports:
      - "3000:3000"
    depends_on:
      - db
volumes:
  postgres-data:
    driver: local
```
- add `.dockerignore` file inside your new folder (file will let know docker to ignore things that are not needed for running app):
```
.git
.gitignore
README.md
```
- to create the new Rails app run:
```bash
docker-compose run web rails new . --force -T --database=postgresql --skip-bundle
```
- only if on Linux: `sudo chown -R $USER:$USER .` (if you check files with `ls -l` - you will see that all of them will be `u:root g:root`);
- let's now build the app (we must rebuild each time we change `Gemfile` or `Dockerfile`): `docker-compose build`
- we should make changes to allow app to connect to database:

```yml
default: &default
  adapter: postgresql
  encoding: unicode
  host: db
  username: postgres
  password:
  pool: 5
development:
  <<: *default
  database: app-name-here_development
test:
  <<: *default
  database: app-name-here_test
```
- boot app with `docker-compose up`
- in another terminal run `docker-compose run web rake db:create`
- app will be ready in `http://localhost:3000`
- you can stop application either with `docker-compose down` or with `Ctrl+C`

## setup testing with rspec
- add to `Gemfile`:
```ruby
...
group :development, :test do
  ...
  gem 'rspec-rails'
end
```
- stop and rebuild app: `docker-compose build` (afterwards you can start it again)
- generate all necessary generations: `docker-compose run web rails g rspec:install`
- if linux and you run any comand for new folders sudo chown the folder!
- test your tests: `docker-compose run web bundle exec rspec`

## create alias
- to make easier to work with docker edit `~/.bashrc` file and add aliases:
```bashrc
alias dc=docker-compose
alias dc-web='docker-compose run web'
alias dc-migrate='docker-compose run web rake db:migrate'
alias dc-rspec='docker-compose run web rspec'
```
- to ensure loading the updated `.bashrc` file you should do this: `. ~/.bashrc`

## errors
### error about unable to monitor direcories:
- error:
```bash
web_1  | FATAL: Listen error: unable to monitor directories for changes.
web_1  | Visit https://github.com/guard/listen/wiki/Increasing-the-amount-of-inotify-watchers for info on how to fix this.
web_1  | => Booting Puma
web_1  | => Rails 5.1.4 application starting in development
web_1  | => Run `rails server -h` for more startup options
web_1  | Exiting
strongestcity_web_1 exited with code 1
```
- solution:
```bash
dc-web echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
```

## inspiration:
1. [docker dcumentation for rails & postgres](https://docs.docker.com/compose/rails/)
2. [RubySnack #60](https://www.youtube.com/watch?v=KH6pcHb6Wug)
3. [RubySnack #61](https://www.youtube.com/watch?v=M5YS5_sSwGk)
4. [RubySnack #62](https://www.youtube.com/watch?v=vmjz_I9gAM0)
5. [How to Setup Rspec Instead of Test::Unit in Rails](https://nrakochy.github.io/rspec/rails/2015/05/27/How-To-Setup-Rspec-Instead-Of-Test-Unit-Rails/)
