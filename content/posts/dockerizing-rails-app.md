+++
title = "Dockerizing A Rails App"
date = "2021-02-05T07:05:02-05:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["Docker", "Rails", "Automation"]
keywords = ["Docker", "Rails", "Automation", "Multi Stage Build", "Multi-Stage Build"]
description = "Dockerize a Rails application for all your needs"
showFullContent = false
+++

### Contents
1. [Intro](#intro)
1. [One Docker File to Rule Them All](#one-docker-file-to-rule-them-all)
1. [Common Dependencies](#common-dependencies)
1. [Ruby Gems/Node Packages](#ruby-gemsnode-packages)
1. [Local Development](#local-development)
1. [Deploying to Production](#deploying-to-production)
1. [Wrapping Up](#wrapping-up)

## Intro
So today we're going to look at taking a Rails 6 application, and dockerizing it via a single Dockerfile to manage all of
our environments (dev, test, and prod), and deploy that application to Heroku. So for a quick overview today we'll be using:

1. Docker
1. Ruby 2.7.2
1. Rails 6.1
1. Postgres 13
1. NodeJS 14

We'll be interacting with the Postgres instance running on the host machine, though
this could easily be switched out for Postgres running in another container (we'll leave that as an exercise for the reader for now).

You can find the example application is running [here](https://rails-docker-example.herokuapp.com/) and
you can find the source code [here](https://github.com/jm96441n/rails_docker).

## One Docker File to Rule Them All
First things first let's look at the Dockerfile we'll be using and then start breaking it down piece by piece,
so we can see how it all fits together.

```Dockerfile
# Dockerfile rails
FROM ruby:2.7.2-alpine as base_deps

# common deps
RUN apk add --update \
    build-base \
    git \
    tzdata

# Application deps
RUN apk add --update \
    nodejs \
    postgresql-client \
    postgresql-dev \
    yarn

WORKDIR /app
# install bundler
RUN gem install bundler
# install rails
RUN gem install rails

FROM base_deps as ruby_deps
WORKDIR /app
# Install gems
ADD Gemfile* ./
RUN bundle install

FROM ruby_deps as node_deps
WORKDIR /app
# Install node modules
ADD package.json *yarn* ./
RUN yarn install --check-files

FROM node_deps as test_deps
COPY --from=ruby_deps /usr/local/bundle/ /usr/local/bundle/
COPY --from=node_deps ./app/node_modules /app/node_modules/
RUN apk add --update \
    chromium \
    chromium-chromedriver  \
    python3 \
    python3-dev \
    py3-pip
RUN pip3 install -U selenium
RUN bundle install --with test

# A separate build stage installs test dependencies and runs your tests
FROM test_deps AS test
WORKDIR /app
ENV DATABASE_HOST=docker.for.mac.localhost
# The test stage installs the test dependencies
# The actual test run
CMD ["bundle", "exec", "rspec"]


# A separate build stage installs test dependencies and runs your tests
FROM node_deps AS dev_deps
COPY --from=ruby_deps /usr/local/bundle/ /usr/local/bundle/
COPY --from=node_deps ./app/node_modules /app/node_modules/
# The test stage installs the test dependencies
RUN bundle install --with test development

FROM dev_deps AS dev
WORKDIR /app
ENV DATABASE_HOST=docker.for.mac.localhost
CMD ["bin/dev_entry"]

FROM node_deps as prod
WORKDIR /app
COPY --from=ruby_deps /usr/local/bundle/ /usr/local/bundle/
COPY . ./
COPY --from=node_deps ./app/node_modules /app/node_modules/
RUN chmod +x ./bin/prod_entry
RUN bundle exec rake assets:precompile

CMD ["bin/prod_entry"]
```

Alright, that's one long Dockerfile, let's start breaking it down!

## Common Dependencies
In this stage we'll be installing all the common dependencies which will form the base for all the ensuing stages.

```Dockerfile
FROM ruby:2.7.2-alpine as base_deps

# common deps
RUN apk add --update \
    build-base \
    git \
    tzdata

# Application deps
RUN apk add --update \
    nodejs \
    postgresql-client \
    postgresql-dev \
    yarn

WORKDIR /app
# install Bundler
RUN gem install bundler
# install rails
RUN gem install rails

```

This first stage is where we want to keep our dependencies that are least likely to change, as whenever we make a change 
to a particular stage we have to rebuild all the following stages.

We're going to be using the Ruby 2.7.2 Alpine Docker image as our base, which ships with v3.13 of Alpine Linux which will help
keep our final container size smaller than if we had used Ubuntu. Following that we'll be installing `build-base`, which 
is the Alpine counterpart to `build-essential` on Ubuntu, and contains many of the essential applications we'll need to 
compile applications from source. In addtion to `build-base` we'll also be adding `git`, and `tzdata` which helps with timezone management.

Following those base dependencies we'll then move on to installing our application dependencies. We'll need to add `node` for
handling our JS engine, `postgresql-client` and `postgresql-dev` for handling our connections to Postgres, and `yarn` for JS package management.
We then move on to setting the working directory for our application and installing Bundler and Rails.

## Ruby Gems/Node Packages
Next up we'll be installing the base set of Ruby gems that we'll need across our environments and our node modules.

```Dockerfile
FROM base_deps as ruby_deps
WORKDIR /app
# Install gems
ADD Gemfile* ./
RUN bundle install

FROM ruby_deps as node_deps
WORKDIR /app
# Install node modules
ADD package.json *yarn* ./
RUN yarn install --check-files
```

This section is another pretty straightforward one, for both stages we'll be copying over the files responsible for
handling what dependencies to install (`Gemfile`/`Gemfile.lock` for Ruby, `package.json`/`yarn.lock` for JS). It's at this point that I'm
going to point out that we also have a `.dockerignore` file in our application directory that tells Docker to ignore the
`node_modules` directory from the host machine.

```.dockerignore
**/node_modules/
```

This is necessary because we don't want to copy our host `node_modules` directory at any point because that will cause
yarn to fail its integrity check due to the modules being run on a different base OS than what they were installed on.

From here on out our three environments are going to start differing in their builds, so we're going to approach each
environment separately and discuss the necessary steps to get each environment up and running. We'll discuss both dev and
test under the [Local Development](#local-development) section and then our production setup under the 
[Deploying to Production](#deploying-to-production) section.

## Local Development

### Development Environment Setup

#### The Docker Stages
Let's begin with looking at the Docker stages necessary for building the dev image.
```Dockerfile
# Setup development specific dependencies
FROM node_deps AS dev_deps
COPY --from=ruby_deps /usr/local/bundle/ /usr/local/bundle/
COPY --from=node_deps ./app/node_modules /app/node_modules/
# The test stage installs the test dependencies
RUN bundle install --with test development

# Run the local dev environment
FROM dev_deps AS dev
WORKDIR /app
ENV DATABASE_HOST=docker.for.mac.localhost
CMD ["bin/dev_entry"]
```

You'll see the first thing we do is copy our already installed dependencies from the previous stages, which allows us to utilize 
the already installed dependencies without having to reinstall them on each build (assuming they haven't changed.) From there 
we then install the test and development sections of our Gemfile and move on to the final stage. In the last stage we set the 
working directory and set an environment variable which allows us to talk to our local Postgres from within the app container. 
Finally, we set our run command to be `bin/dev_entry` which is a file that we're going to talk about in the next sub-section.

#### bin/dev_entry
```ruby
#!/usr/bin/env ruby
require 'fileutils'

APP_ROOT = File.expand_path('..', __dir__)

def system!(*args)
  system(*args) || abort("\n== Command #{args} failed ==")
end

FileUtils.chdir APP_ROOT do
  puts "\n== Preparing database =="
  system! 'bin/rails db:prepare'

  puts "\n== Removing old logs and tempfiles =="
  system! 'bin/rails log:clear tmp:clear'

  puts "\n== Restarting application server =="
  system! 'RAILS_ENV=development bundle exec rails s -b 0.0.0.0'
end
```

In the `bin` directory of Rails app we're going to add a file named `dev_entry` which is going to describe the steps we want to take
each time we start our dev server. First we'll want to ensure that all of our migrations have been run, then we'll clear out some old log files,
and lastly spin up our dev server.

Note: Don't forget to give yourself permission for this file: `chmod +x ./bin/dev_entry` so you can run it.

### Test Environment

#### The Docker Stages
```Dockerfile
# Setup test specific dependencies
FROM node_deps as test_deps
COPY --from=ruby_deps /usr/local/bundle/ /usr/local/bundle/
COPY --from=node_deps ./app/node_modules /app/node_modules/
RUN apk add --update \
    chromium \
    chromium-chromedriver  \
    python3 \
    python3-dev \
    py3-pip
RUN pip3 install -U selenium
RUN bundle install --with test

# Run the tests
FROM test_deps AS test
WORKDIR /app
ENV DATABASE_HOST=docker.for.mac.localhost
# The actual test run
CMD ["bundle", "exec", "rspec"]

```

For the test environment we need to copy over the already installed base dependencies just like we did for the development environment.
Following that we need to install the `chromium`, `chromium-chromedriver`, and some python packages in order to install selenium to run
our system tests through a chrome headless browser. Finally we install the specific gems for the test environment.

The actual `test` stage looks really similar to the `dev` stage, but in this case we'll be running `bundle exec rspec` as our run command.

### Running the Application Locally

In this next section we're going to look at another bin file we'll be adding along with a few changes we need to make to our `database.yml`
file to run the application locally and wrapping up with an additional selenium driver we'll want to add to run our tests from inside
the container.

#### bin/docker

This is a bin file that we'll be adding to handle building and running our application locally. One thing to notice is that
we'll be mounting the majority of the application directories as volumes rather than copying them in so that we get hot reloading
of changes to our app code. This gives us the benefit of not having to stop and restart our container for changes to be seen (except for changes
that require a restart to the application server.) For this section we'll just be discussing the path taken by the `dev` and `test` arguments,
later on in the [Deploying to Production](#deploying-to-production) section we'll discuss the `deploy` path.

```ruby
#!/usr/bin/env ruby
require 'fileutils'

# path to your application root.
APP_ROOT = File.expand_path('..', __dir__)

NON_MOUNTED_DEV_FILES = %w[node_modules yarn.lock Gemfile coverage README.md spec].freeze
NON_MOUNTED_TEST_FILES = %w[node_modules yarn.lock Gemfile README.md].freeze

def system!(*args)
  system(*args) || abort("\n== Command #{args} failed ==")
end

def parse_commands(args)
  if args.empty? || args[0] == 'help'
    usage
  elsif args[0] == 'dev' || args[0] == 'test'
    build_and_run(args[0])
  elsif args[0] == 'deploy'
    if ensure_deploy?
      build_and_deploy
    else
      pp 'Aborting deployment'
    end
  else
    incorrect_error
  end
end

def usage
  pp 'Usage'
  pp '"bin/docker dev" for running the container locally'
  pp '"bin/docker test" to run the tests'
  pp '"bin/docker deploy" to build and deploy to production'
end

# builds the image and run the container
def build_and_run(env)
  build(env)
  run(env)
end

# builds the Docker container
def build(env)
  system!("docker build --target #{env} -t jmaguire5588/rails_docker:#{env} ./")
end

# build the Docker command into a string to be run
def run(env)
  cmd = "docker run --rm --name=rails_docker_#{env} "
  cmd << '-p 3000:3000 -p 5432:5432 '
  cmd << mounted_volumes(env)
  cmd << "-it jmaguire5588/rails_docker:#{env}"
  system!(cmd)
end

# Builds a list of the current files in the directory to mount to your Docker container for
# local dev, does not mount node_modules
def mounted_volumes(env)
  non_mounted_files = env == 'dev' ? NON_MOUNTED_DEV_FILES : NON_MOUNTED_TEST_FILES
  files = Dir.glob('*')
  files.reduce('') do |vol_string, file|
    if non_mounted_files.include?(file)
      vol_string
    else
      vol_string << "-v #{APP_ROOT}/#{file}:/app/#{file} "
    end
  end
end

def ensure_deploy?
  pp 'WARNING: You will be building and deploying to production, is this what you intend? [Y/n]'
  input = STDIN.gets.chomp
  %w[Y y].include? input
end

def build_and_deploy
  build('prod')
  system!('docker tag jmaguire5588/rails_docker:prod registry.heroku.com/rails-docker-example/web')
  system!('docker push registry.heroku.com/rails-docker-example/web')
  system!('heroku container:release web -a rails-docker-example')
end

def incorrect_error
  pp 'Error! That is not a recognized command'
end

FileUtils.chdir APP_ROOT do
  parse_commands(ARGV)
end
```

The main source of complexity here is the files that we're mounting, and you'll notice we have two constants `NON_MOUNTED_DEV_FILES` and
`NON_MOUNTED_TEST_FILES`. As the names imply one is the files/folders we won't be mounting in development and the other are files we don't want 
to mount for testing. In both we want to exclude any dependency related files (package.json, yarn.lock, Gemile, and Gemfile.lock) as well as 
the README. You'll also notice we exclude the `node_modules` directory because it already exists from a previous stage. The difference between 
our dev and test environments is that in test we mount the spec directory and for dev we do not (as we don't need it).

The reason we go about this kinda obtuse way (manually mounting all directories that we want instead of mounting the entire app directory) is 
that we want to use the `node_modules` directory we built earlier and if we mount the entire app directory from the host machine 
we lose any existing files/folders in that directory. I did some investigation into installing the `node_modules` directory to a 
folder outside the app directory and it ended up being more trouble than it was worth.

Wrapping up, this command takes in a command line argument, either `help`, `dev`, `test`, or `deploy` and then builds and runs the container
for that particular target.

Note: Don't forget to give yourself permission for this file: `chmod +x ./bin/docker` so you can run it.

#### database.yml
```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: rails_docker_development
  host: <%= ENV['DATABASE_HOST'] %>
  username: <%= Rails.application.credentials.database[:username] %>
  password: <%= Rails.application.credentials.database[:password] %>

test:
  <<: *default
  database: rails_docker_test
  host: <%= ENV['DATABASE_HOST'] %>
  username: <%= Rails.application.credentials.database[:username] %>
  password: <%= Rails.application.credentials.database[:password] %>
```

The main change here is that we'll be specifying the development and test host as the environment variable that we set
in our Dockerfile (the `ENV DATABASE_HOST=docker.for.mac.localhost` variable that we set.) This just allows our Rails
application talk to our local Postgres.

#### Test Setup

We'll want to start by removing the `webdrivers` gem from out Gemfile as there are still some issues with the gem dealing with
the install location for the webdriver binaries on Alpine (https://github.com/titusfortner/webdrivers/issues/78).

Following that we want to add a new support file to run our integration tests through our installed version of chrome. To do that first
we have to tell rspec to include our support files by adding the following line to our `rspec_helper`.
```ruby
Dir[Rails.root.join('spec', 'support', '**', '*.rb')].sort.each { |f| require f }
```

Now we'll add our driver for chrome at `spec/support/chrome.rb`:
```ruby
# frozen_string_literal: true

driver = :selenium_chrome_headless

Capybara.server = :puma, { Silent: true }

Capybara.register_driver driver do |app|
  options = ::Selenium::WebDriver::Chrome::Options.new

  options.add_argument('--headless')
  options.add_argument('--no-sandbox')
  options.add_argument('--disable-dev-shm-usage')
  options.add_argument('--window-size=1400,1400')

  Capybara::Selenium::Driver.new(app, browser: :chrome, options: options)
end

Capybara.javascript_driver = driver

RSpec.configure do |config|
  config.before(:each, type: :system) do
    driven_by driver
  end
end
```

This looks complicated, but all it does is set up a headless chrome driver to run our system tests with.

## Deploying to Production

Now that we've got the hard parts out of the way, let's deploy our application to Heroku. We'll be using the
[Heroku Container Registry](https://devcenter.heroku.com/articles/container-registry-and-runtime) to host our container, though 
you can easily use any other container registry. Before we get into the actual deployment process, let's take a look at the last stage of our
Dockerfile which contains the `prod` build target.

```Dockerfile
FROM node_deps as prod
WORKDIR /app
COPY --from=ruby_deps /usr/local/bundle/ /usr/local/bundle/
COPY . ./
COPY --from=node_deps ./app/node_modules /app/node_modules/
RUN chmod +x ./bin/prod_entry
RUN bundle exec rake assets:precompile

CMD ["bin/prod_entry"]
```

Again, this looks really similar to our other environments, the main differences here are:
1. Giving the production server access to our `/bin/prod_entry` file (in a more formalized environment we would want to set 
up a separate user and only give that user access to that file)
1. Copying our `node_modules` directory over from the `node_deps` stage into the current stage *after* we copy in the application directory
so that we don't overwrite `node_modules`.
1. Pre-compiling our assets for production.
1. Using the `bin/prod_entry` file as the command to run.

Now that we've gone over the `prod` stage of the Dockerfile let's look at the last two parts: `bin/prod_entry` and the
`deploy` path of `bin/docker`.

### `bin/prod_entry`

```ruby
#!/usr/bin/env ruby
require 'fileutils'

APP_ROOT = File.expand_path('..', __dir__)

def system!(*args)
  system(*args) || abort("\n== Command #{args} failed ==")
end

FileUtils.chdir APP_ROOT do
  puts "\n== Preparing database =="
  system! 'bin/rails db:prepare'

  puts "\n== Restarting application server =="
  system! 'RAILS_ENV=production bundle exec puma -C config/puma.rb'
end
```

This should look very familiar as it's nearly the same as the `bin/dev_entry` file with a few exceptions: we're no longer
clearing out the logs, and here we're running the puma web server.

### `bin/docker`

We're going to take one final look at the `bin/docker`, specifically the `deploy` path:

```ruby
# some other code

def parse_commands(args)
  if args.empty? || args[0] == 'help'
    usage
  elsif args[0] == 'dev' || args[0] == 'test'
    build_and_run(args[0])
  elsif args[0] == 'deploy'
    if ensure_deploy?
      build_and_deploy
    else
      pp 'Aborting deployment'
    end
  else
    incorrect_error
  end
end

# ...rest of the code here...

def ensure_deploy?
  pp 'WARNING: You will be building and deploying to production, is this what you intend? [Y/n]'
  input = STDIN.gets.chomp
  %w[Y y].include? input
end

def build_and_deploy
  build('prod')
  system!('docker tag jmaguire5588/rails_docker:prod registry.heroku.com/rails-docker-example/web')
  system!('docker push registry.heroku.com/rails-docker-example/web')
  system!('heroku container:release web -a rails-docker-example')
end
```

So here we see two additonal functions, one is to ensure that we intend to deploy the prod and the other is to build
the production image and push it to the heroku container registry and then release it.


## Wrapping Up

So that about does it! After following these steps you now have a Rails app that is using Docker in your dev, test, and
production environments! The example application is deployed [here](https://rails-docker-example.herokuapp.com/) and
you can find the source code [here](https://github.com/jm96441n/rails_docker)
