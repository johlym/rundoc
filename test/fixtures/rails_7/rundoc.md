```
:::-- rundoc
email = ENV['HEROKU_EMAIL'] || `heroku auth:whoami`

Rundoc.configure do |config|
  config.project_root = "myapp"
  config.filter_sensitive(email => "developer@example.com")
  config.filter_sensitive(Dir.pwd => ".")
end
```

<!--
  rundoc src:
  https://github.com/schneems/rundoc/blob/main/test/fixtures/rails_7/rundoc.md

  Command:
  $ bin/rundoc build --path test/fixtures/rails_7/rundoc.md
-->

Ruby on Rails is a popular web framework written in [Ruby](http://www.ruby-lang.org/). This guide covers using Rails 7 on Heroku. For information on running previous versions of Rails on Heroku, see the tutorial for [Rails 6.x](getting-started-with-rails6) or [Rails 5.x](getting-started-with-rails5).


Before continuing, it’s helpful to have:

- Basic familiarity with Ruby, Ruby on Rails, and Git
- A locally installed version of Ruby 2.7.0+, Rubygems, Bundler, and Rails 7+
- A Heroku user account: [Signup is free and instant](https://signup.heroku.com/devcenter).
- A locally installed version of the [Heroku CLI](heroku-cli#download-and-install)

## Local setup

With the Heroku CLI installed, `heroku` is now an available command in the terminal. Log in to Heroku using the CLI:

```term
$ heroku login
heroku: Enter your Heroku credentials
Email: schneems@example.com
Password:
Could not find an existing public key.
Would you like to generate one? [Yn]
Generating new SSH public key.
Uploading ssh public key /Users/adam/.ssh/id_rsa.pub
```

Press Enter at the prompt to upload an existing `ssh` key or create a new one. 

>info
>After November 30, 2021, Heroku [will no longer support the SSH Git transport](​​https://devcenter.heroku.com/changelog-items/2215). SSH keys will serve no purpose in pushing code to applications on the Heroku platform.

## Create a New or Upgrade an Existing Rails App 

Ensure Rails 7 is installed with `rails -v` before creating an app. If necessary, install Rails 7 with `gem install`:

```term
$ gem install rails --no-document --pre
Successfully installed rails-7.0.0.alpha2
1 gem installed
```

Create an app and move it into its root directory:

```term
$ rails new myapp --database=postgresql
```

Move into the application directly and add the `x86_64-linux` and `ruby` platforms to `Gemfile.lock`.

```term
$ cd myapp
$ bundle lock --add-platform x86_64-linux --add-platform ruby
Fetching gem metadata from https://rubygems.org/............
Resolving dependencies...
Writing lockfile to ./myapp/Gemfile.lock
```

Create a local database:

```
$ bin/rails db:create
Database 'myapp_development' already exists
Database 'myapp_test' already exists
```

## Add the pg gem

For new or existing apps where `--database=postgresql` wasn’t defined, confirm the `sqlite3` gem doesn’t exist in the `Gemfile`. Add the `pg` gem in its place. 

Within the `Gemfile` remove:

```ruby
gem 'sqlite3'
```

And replace it with:

```ruby
gem 'pg'
```

> callout Heroku highly recommends using PostgreSQL locally during development. Maintaining [parity between development](http://www.12factor.net/dev-prod-parity) and deployment environments prevents subtle bugs from being introduced because of the differences in those environments. 
>
> [Install Postgres locally](heroku-postgresql#local-setup) now if not present on the system, already. For more information on why Postgres is recommended instead of Sqlite3, see [why you can’t use Sqlite3 on Heroku](sqlite3).

With the `Gemfile` updated, reinstall the dependencies: 

```ruby
$ bundle install
```

Doing so updates `Gemfile.lock` with the changes made previously.

In addition to using the `pg` gem, ensure that `config/database.yml` defines the `postgresql` adapter. The development section of `config/database.yml` file will look something like this:

```term
$ cat config/database.yml
# PostgreSQL. Versions 9.3 and up are supported.
#
# Install the pg driver:
#   gem install pg
# On macOS with Homebrew:
#   gem install pg -- --with-pg-config=/usr/local/bin/pg_config
# On macOS with MacPorts:
#   gem install pg -- --with-pg-config=/opt/local/lib/postgresql84/bin/pg_config
# On Windows:
#   gem install pg
#       Choose the win32 build.
#       Install PostgreSQL and put its /bin directory on your path.
#
# Configure Using Gemfile
# gem "pg"
#
default: &default
  adapter: postgresql
  encoding: unicode
  # For details on connection pooling, see Rails configuration guide
  # https://guides.rubyonrails.org/configuring.html#database-pooling
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: myapp_development

  # The specified database role being used to connect to postgres.
  # To create additional roles in postgres see `$ createuser --help`.
  # When left blank, postgres will use the default role. This is
  # the same name as the operating system user running Rails.
  #username: myapp

  # The password associated with the postgres role (username).
  #password:

  # Connect on a TCP socket. Omitted by default since the client uses a
  # domain socket that doesn't need configuration. Windows does not have
  # domain sockets, so uncomment these lines.
  #host: localhost

  # The TCP port the server listens on. Defaults to 5432.
  # If your server runs on a different port number, change accordingly.
  #port: 5432

  # Schema search path. The server defaults to $user,public
  #schema_search_path: myapp,sharedapp,public

  # Minimum log levels, in increasing order:
  #   debug5, debug4, debug3, debug2, debug1,
  #   log, notice, warning, error, fatal, and panic
  # Defaults to warning.
  #min_messages: notice

# Warning: The database defined as "test" will be erased and
# re-generated from your development database when you run "rake".
# Do not set this db to the same as development or production.
test:
  <<: *default
  database: myapp_test

# As with config/credentials.yml, you never want to store sensitive information,
# like your database password, in your source code. If your source code is
# ever seen by anyone, they now have access to your database.
#
# Instead, provide the password or a full connection URL as an environment
# variable when you boot the app. For example:
#
#   DATABASE_URL="postgres://myuser:mypass@localhost/somedatabase"
#
# If the connection URL is provided in the special DATABASE_URL environment
# variable, Rails will automatically merge its configuration values on top of
# the values provided in this file. Alternatively, you can specify a connection
# URL environment variable explicitly:
#
#   production:
#     url: <%= ENV["MY_APP_DATABASE_URL"] %>
#
# Read https://guides.rubyonrails.org/configuring.html#configuring-a-database
# for a full overview on how database connection configuration can be specified.
#
production:
  <<: *default
  database: myapp_production
  username: myapp
  password: <%= ENV["MYAPP_DATABASE_PASSWORD"] %>
```

Be careful here. If the value of `adapter` is `postgres` and not `postgresql` (note the `sql` at the end), the application won’t work.

## Create a Welcome Page

Rails 7 no longer has a static index page in production by default. Apps upgraded to Rails 7 keep their existing page configurations, but new Rails 7 apps do not have an automatically generated welcome page. Create a `welcome` controller to hold the homepage:

```term
$ rails generate controller welcome
```

Create `app/views/welcome/index.html.erb` and add the following snippet:

```html
<h2>Hello World</h2>
<p>
  The time is now: <%= Time.now %>
</p>
```

With a welcome page created, create a route to map to this action. Edit `config/routes.rb` to set the index page to the new method:

In file `config/routes.rb`, on line 2 add:

```ruby
  root 'welcome#index'
```

Verify the page is present by starting the Rails web server:

```term
$ rails server
```

Visit [http://localhost:3000](http://localhost:3000) in a browser. If the page doesn’t display, [reference the logs](#view-logs) Rails outputs within the same terminal where `rails server` started to debug the error.

## Heroku gems

Previous versions of Rails (Rails 4 and older) required the [rails_12factor](https://github.com/heroku/rails_12factor) gem to enable static asset serving and logging on Heroku. New Rails applications don’t need this gem. The gem can be removed from existing, upgraded applications provided the following code is present in `config/environments/production.rb`:

```ruby
# config/environments/production.rb
config.public_file_server.enabled = ENV['RAILS_SERVE_STATIC_FILES'].present?

if ENV["RAILS_LOG_TO_STDOUT"].present?
  logger           = ActiveSupport::Logger.new(STDOUT)
  logger.formatter = config.log_formatter
  config.logger = ActiveSupport::TaggedLogging.new(logger)
end
```

## Specify the Ruby Version

Rails 7 requires Ruby 2.7.0 or above. Heroku installs a recent version of Ruby buy default. Specify an exact version with the `ruby` DSL in `Gemfile` like the following example of defining Ruby 3.0.2:


```ruby
ruby "3.0.2"
```

Always use the same version of Ruby locally, too. Confirm the local version of ruby with `ruby -v`. Refer to the [Ruby Versions](ruby-versions) article for more details on defining a specific ruby version.

## Store The App in Git

Heroku relies on [Git](http://git-scm.com/), a distributed source control management tool, for deploying applications. If the application is not already in Git, first verify that `git` is on the system with `git --help`:

```term
$ git --help
usage: git [--version] [--help] [-C <path>] [-c <name>=<value>]
           [--exec-path[=<path>]] [--html-path] [--man-path] [--info-path]
           [-p | --paginate | -P | --no-pager] [--no-replace-objects] [--bare]
           [--git-dir=<path>] [--work-tree=<path>] [--namespace=<name>]
           [--super-prefix=<path>] [--config-env=<name>=<envvar>]
```

Git is not present if the command produces no output or `command not found`. Install Git on the system.

After verifying Git is functional, navigate to the root directory of the Rails app. The contents of the Rails app looks something like this when using `ls`:

```term
$ ls
Gemfile
Gemfile-e
Gemfile.lock
README.md
Rakefile
app
bin
config
config.ru
db
lib
log
public
storage
test
tmp
vendor
```

Within the Rails app directly, initialize a local empty Git repository and commit the app’s code:

```term
$ git init
$ git add .
$ git commit -m "init"
```

Verify everything was committed correctly with `git status`:

```term
$ git status
On branch main
nothing to commit, working tree clean
```

With the application committed to Git, it is ready to deploy to Heroku.

## Deploy the Application to Heroku

Inside the Rails app’s root directory, use the Heroku CLI to create an app on Heroku:

```term
$ heroku create
Creating app... done, protected-caverns-64520
https://protected-caverns-64520.herokuapp.com/ | https://git.heroku.com/protected-caverns-64520.git
```

The Heroku CLI adds the Git remote automatically. Verify it is set with `git config`:

```term
$ git config --list --local | grep heroku
remote.heroku.url=https://git.heroku.com/protected-caverns-64520.git
remote.heroku.fetch=+refs/heads/*:refs/remotes/heroku/*
```

Git returns `fatal: not in a git directory` if the current directory is incorrect or Git is not [initialized](#store-the-app-in-git). If Git returns a list of remotes, it is ready to deploy. 

>note
>Following changes in the industry, Heroku [updated the default branch name](​​https://devcenter.heroku.com/changelog-items/1829) to `main`. If the project uses `master` as its default branch name, use `git push heroku master`.

Deploy the code. The output looks something like this:

```term
$ git push heroku main
remote: Compressing source files... done.        
remote: Building source:        
remote: 
remote: -----> Building on the Heroku-20 stack        
remote: -----> Determining which buildpack to use for this app        
remote: -----> Ruby app detected        
remote: -----> Installing bundler 2.2.21        
remote: -----> Removing BUNDLED WITH version in the Gemfile.lock        
remote: -----> Compiling Ruby/Rails        
remote: -----> Using Ruby version: ruby-3.0.2        
remote: -----> Installing dependencies using bundler 2.2.21        
remote:        Running: BUNDLE_WITHOUT='development:test' BUNDLE_PATH=vendor/bundle BUNDLE_BIN=vendor/bundle/bin BUNDLE_DEPLOYMENT=1 bundle install -j4        
remote:        Fetching gem metadata from https://rubygems.org/        
remote:        Fetching gem metadata from https://rubygems.org/............        
remote:        Fetching rake 13.0.6        
remote:        Installing rake 13.0.6        
remote:        Fetching concurrent-ruby 1.1.9        
remote:        Fetching minitest 5.14.4        
remote:        Fetching erubi 1.10.0        
remote:        Fetching builder 3.2.4        
remote:        Installing builder 3.2.4        
remote:        Installing erubi 1.10.0        
remote:        Installing minitest 5.14.4        
remote:        Fetching mini_portile2 2.6.1        
remote:        Installing concurrent-ruby 1.1.9        
remote:        Fetching racc 1.5.2        
remote:        Installing mini_portile2 2.6.1        
remote:        Fetching crass 1.0.6        
remote:        Installing crass 1.0.6        
remote:        Fetching rack 2.2.3        
remote:        Fetching nio4r 2.5.8        
remote:        Installing racc 1.5.2 with native extensions        
remote:        Installing rack 2.2.3        
remote:        Installing nio4r 2.5.8 with native extensions        
remote:        Fetching websocket-extensions 0.1.5        
remote:        Installing websocket-extensions 0.1.5        
remote:        Fetching marcel 1.0.1        
remote:        Fetching mini_mime 1.1.1        
remote:        Installing marcel 1.0.1        
remote:        Installing mini_mime 1.1.1        
remote:        Fetching msgpack 1.4.2        
remote:        Installing msgpack 1.4.2 with native extensions        
remote:        Using bundler 2.2.22        
remote:        Fetching method_source 1.0.0        
remote:        Installing method_source 1.0.0        
remote:        Fetching thor 1.1.0        
remote:        Installing thor 1.1.0        
remote:        Fetching zeitwerk 2.5.0.beta3        
remote:        Installing zeitwerk 2.5.0.beta3        
remote:        Fetching pg 1.2.3        
remote:        Installing pg 1.2.3 with native extensions        
remote:        Fetching redis 4.3.1        
remote:        Installing redis 4.3.1        
remote:        Fetching rack-test 1.1.0        
remote:        Installing rack-test 1.1.0        
remote:        Fetching i18n 1.8.10        
remote:        Installing i18n 1.8.10        
remote:        Fetching tzinfo 2.0.4        
remote:        Installing tzinfo 2.0.4        
remote:        Fetching sprockets 4.0.2        
remote:        Installing sprockets 4.0.2        
remote:        Fetching websocket-driver 0.7.5        
remote:        Installing websocket-driver 0.7.5 with native extensions        
remote:        Fetching mail 2.7.1        
remote:        Installing mail 2.7.1        
remote:        Fetching nokogiri 1.12.4        
remote:        Installing nokogiri 1.12.4 with native extensions        
remote:        Fetching activesupport 7.0.0.alpha2        
remote:        Installing activesupport 7.0.0.alpha2        
remote:        Fetching puma 5.4.0        
remote:        Installing puma 5.4.0 with native extensions        
remote:        Fetching globalid 0.5.2        
remote:        Installing globalid 0.5.2        
remote:        Fetching activemodel 7.0.0.alpha2        
remote:        Installing activemodel 7.0.0.alpha2        
remote:        Fetching jbuilder 2.11.2        
remote:        Installing jbuilder 2.11.2        
remote:        Fetching bootsnap 1.9.0        
remote:        Installing bootsnap 1.9.0 with native extensions        
remote:        Fetching activejob 7.0.0.alpha2        
remote:        Installing activejob 7.0.0.alpha2        
remote:        Fetching activerecord 7.0.0.alpha2        
remote:        Installing activerecord 7.0.0.alpha2        
remote:        Fetching rails-dom-testing 2.0.3        
remote:        Fetching loofah 2.12.0        
remote:        Installing rails-dom-testing 2.0.3        
remote:        Installing loofah 2.12.0        
remote:        Fetching rails-html-sanitizer 1.4.2        
remote:        Installing rails-html-sanitizer 1.4.2        
remote:        Fetching actionview 7.0.0.alpha2        
remote:        Installing actionview 7.0.0.alpha2        
remote:        Fetching actionpack 7.0.0.alpha2        
remote:        Installing actionpack 7.0.0.alpha2        
remote:        Fetching actioncable 7.0.0.alpha2        
remote:        Fetching activestorage 7.0.0.alpha2        
remote:        Fetching railties 7.0.0.alpha2        
remote:        Fetching actionmailer 7.0.0.alpha2        
remote:        Installing actionmailer 7.0.0.alpha2        
remote:        Installing actioncable 7.0.0.alpha2        
remote:        Installing activestorage 7.0.0.alpha2        
remote:        Installing railties 7.0.0.alpha2        
remote:        Fetching sprockets-rails 3.2.2        
remote:        Installing sprockets-rails 3.2.2        
remote:        Fetching actionmailbox 7.0.0.alpha2        
remote:        Fetching actiontext 7.0.0.alpha2        
remote:        Installing actiontext 7.0.0.alpha2        
remote:        Installing actionmailbox 7.0.0.alpha2        
remote:        Fetching rails 7.0.0.alpha2        
remote:        Installing rails 7.0.0.alpha2        
remote:        Fetching importmap-rails 0.5.3        
remote:        Fetching stimulus-rails 0.5.3        
remote:        Fetching turbo-rails 0.7.13        
remote:        Installing importmap-rails 0.5.3        
remote:        Installing turbo-rails 0.7.13        
remote:        Installing stimulus-rails 0.5.3        
remote:        Bundle complete! 15 Gemfile dependencies, 51 gems now installed.        
remote:        Gems in the groups 'development' and 'test' were not installed.        
remote:        Bundled gems are installed into `./vendor/bundle`        
remote:        Bundle completed (44.08s)        
remote:        Cleaning up the bundler cache.        
remote:        Removing bundler (2.2.21)        
remote: -----> Detecting rake tasks        
remote: -----> Preparing app for Rails asset pipeline        
remote:        Running: rake assets:precompile        
remote:        I, [2021-09-16T14:47:26.137737 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/manifest-dad05bf766af0fe3d79dd746db3c1361c0583026cdf35d6a2921bccaea835331.js        
remote:        I, [2021-09-16T14:47:26.138018 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/manifest-dad05bf766af0fe3d79dd746db3c1361c0583026cdf35d6a2921bccaea835331.js.gz        
remote:        I, [2021-09-16T14:47:26.138728 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/application-e0cf9d8fcb18bf7f909d8d91a5e78499f82ac29523d475bf3a9ab265d5e2b451.css        
remote:        I, [2021-09-16T14:47:26.139096 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/application-e0cf9d8fcb18bf7f909d8d91a5e78499f82ac29523d475bf3a9ab265d5e2b451.css.gz        
remote:        I, [2021-09-16T14:47:26.139625 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/application-37f365cbecf1fa2810a8303f4b6571676fa1f9c56c248528bc14ddb857531b95.js        
remote:        I, [2021-09-16T14:47:26.139897 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/application-37f365cbecf1fa2810a8303f4b6571676fa1f9c56c248528bc14ddb857531b95.js.gz        
remote:        I, [2021-09-16T14:47:26.140121 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/controllers/application-0a88d7da94dddbd4b5db3a7e58aba83c761c0de29f578c197e4e41a3a79d014f.js        
remote:        I, [2021-09-16T14:47:26.140263 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/controllers/application-0a88d7da94dddbd4b5db3a7e58aba83c761c0de29f578c197e4e41a3a79d014f.js.gz        
remote:        I, [2021-09-16T14:47:26.140438 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/controllers/hello_controller-549135e8e7c683a538c3d6d517339ba470fcfb79d62f738a0a089ba41851a554.js        
remote:        I, [2021-09-16T14:47:26.140596 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/controllers/hello_controller-549135e8e7c683a538c3d6d517339ba470fcfb79d62f738a0a089ba41851a554.js.gz        
remote:        I, [2021-09-16T14:47:26.140769 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/controllers/index-f6aa019eef4d0e13975a27efdf6d17ba2c6446417bfcb3139efd889f948a5dfc.js        
remote:        I, [2021-09-16T14:47:26.140905 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/controllers/index-f6aa019eef4d0e13975a27efdf6d17ba2c6446417bfcb3139efd889f948a5dfc.js.gz        
remote:        I, [2021-09-16T14:47:26.141088 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/turbo-94dc4be4d3faa69556020cfe3ec669a335da7afc2d6dd761ffff2b6b32ce5507.js        
remote:        I, [2021-09-16T14:47:26.141253 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/turbo-94dc4be4d3faa69556020cfe3ec669a335da7afc2d6dd761ffff2b6b32ce5507.js.gz        
remote:        I, [2021-09-16T14:47:26.141428 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/activestorage-3ab61e47dd4ee2d79db525ade1dca2ede0ea2b7371fe587e408ee37b7ade265d.js        
remote:        I, [2021-09-16T14:47:26.141564 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/activestorage-3ab61e47dd4ee2d79db525ade1dca2ede0ea2b7371fe587e408ee37b7ade265d.js.gz        
remote:        I, [2021-09-16T14:47:26.141741 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/activestorage.esm-01f58a45d77495cdfbdfcc872902a430426c4391634ec9c3da5f69fbf8418492.js        
remote:        I, [2021-09-16T14:47:26.141879 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/activestorage.esm-01f58a45d77495cdfbdfcc872902a430426c4391634ec9c3da5f69fbf8418492.js.gz        
remote:        I, [2021-09-16T14:47:26.142054 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/actiontext-28c61f5197c204db043317a8f8826a87ab31495b741f854d307ca36122deefce.js        
remote:        I, [2021-09-16T14:47:26.142191 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/actiontext-28c61f5197c204db043317a8f8826a87ab31495b741f854d307ca36122deefce.js.gz        
remote:        I, [2021-09-16T14:47:26.142362 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/trix-ac629f94e04ee467ab73298a3496a4dfa33ca26a132f624dd5475381bc27bdc8.css        
remote:        I, [2021-09-16T14:47:26.142527 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/trix-ac629f94e04ee467ab73298a3496a4dfa33ca26a132f624dd5475381bc27bdc8.css.gz        
remote:        I, [2021-09-16T14:47:26.142698 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/trix-1563ff9c10f74e143b3ded40a8458497eaf2f87a648a5cbbfebdb7dec3447a5e.js        
remote:        I, [2021-09-16T14:47:26.142836 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/trix-1563ff9c10f74e143b3ded40a8458497eaf2f87a648a5cbbfebdb7dec3447a5e.js.gz        
remote:        I, [2021-09-16T14:47:26.143006 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/es-module-shims-a1fc5fe40f250a03bc2e7a9b90c68adb1ee2c1b40d1a87f2554941d0c5232a11.js        
remote:        I, [2021-09-16T14:47:26.143142 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/es-module-shims-a1fc5fe40f250a03bc2e7a9b90c68adb1ee2c1b40d1a87f2554941d0c5232a11.js.gz        
remote:        I, [2021-09-16T14:47:26.143310 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/stimulus-b7b939e6650ecc728406742f2db713cda2d3a88976e2621cf818c8699a413e5d.js        
remote:        I, [2021-09-16T14:47:26.143568 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/stimulus-b7b939e6650ecc728406742f2db713cda2d3a88976e2621cf818c8699a413e5d.js.gz        
remote:        I, [2021-09-16T14:47:26.143743 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/stimulus@3.0.0-beta.2-f8ddc8f99943b6a6341f3992164a46f4d545bebaff65a4fd6b7d7d8bbe83b052.js        
remote:        I, [2021-09-16T14:47:26.147818 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/stimulus@3.0.0-beta.2-f8ddc8f99943b6a6341f3992164a46f4d545bebaff65a4fd6b7d7d8bbe83b052.js.gz        
remote:        I, [2021-09-16T14:47:26.148518 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/stimulus-autoloader-9dc757bdeb3b009fe86891deaf96bf334e4b276c01cb0ee1eb025c3d45a43d51.js        
remote:        I, [2021-09-16T14:47:26.148871 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/stimulus-autoloader-9dc757bdeb3b009fe86891deaf96bf334e4b276c01cb0ee1eb025c3d45a43d51.js.gz        
remote:        I, [2021-09-16T14:47:26.149341 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/stimulus-importmap-autoloader-7366d931317007a1e7e62c8dd8198dbc6d6b438207ff8d8539d06019597bf2f7.js        
remote:        I, [2021-09-16T14:47:26.149779 #1223]  INFO -- : Writing /tmp/build_b38fd75f/public/assets/stimulus-importmap-autoloader-7366d931317007a1e7e62c8dd8198dbc6d6b438207ff8d8539d06019597bf2f7.js.gz        
remote:        Asset precompilation completed (1.48s)        
remote:        Cleaning assets        
remote:        Running: rake assets:clean        
remote: -----> Detecting rails configuration        
remote: 
remote: ###### WARNING:        
remote: 
remote:        No Procfile detected, using the default web server.        
remote:        We recommend explicitly declaring how to boot your server process via a Procfile.        
remote:        https://devcenter.heroku.com/articles/ruby-default-web-server        
remote: 
remote: 
remote: -----> Discovering process types        
remote:        Procfile declares types     -> (none)        
remote:        Default types for buildpack -> console, rake, web        
remote: 
remote: -----> Compressing...        
remote:        Done: 36.6M        
remote: -----> Launching...        
remote:        Released v6        
remote:        https://protected-caverns-64520.herokuapp.com/ deployed to Heroku        
remote: 
remote: Verifying deploy... done.        
To https://git.heroku.com/protected-caverns-64520.git
 * [new branch]      main -> main
```

The output may display warnings or error messages. Check the output for these and make adjustments as necessary. 

If the deployment is successful, the application may need a few additional adjustments:

* Migration of the database
* Ensure proper dyno scaling
* Reference the app’s logs if any issues arise 

## Migrate The Database

If the application uses a database, trigger a migration by using the Heroku CLI to start a one-off  [dyno](dynos), which is a lightweight container that is the basic unit of composition on Heroku, and run `db:migrate`:

```term
$ heroku run rake db:migrate
```

Obtain an interactive shell session, instead, with `heroku run bash`.

## Access the Application

The application is successfully deployed to Heroku. Heroku runs application code using defined processes and [process types](procfile). New applications will not have a process type active by default. Scale the `web` process type using the Heroku CLI’s `ps:scale` command:

```term
$ heroku ps:scale web=1
```

Use the Heroku CLI’s `ps` command to display the state of all of an app’s dynos in the terminal:

```term
$ heroku ps
Free dyno hours quota remaining this month: 996h 54m (99%)
Free dyno usage for this app: 0h 0m (0%)
For more information on dyno sleeping and how to upgrade, see:
https://devcenter.heroku.com/articles/dyno-sleeping

=== web (Free): bin/rails server -p ${PORT:-5000} -e $RAILS_ENV (1)
web.1: up 2021/09/16 09:47:43 -0500 (~ 1s ago)
```

In the previous example, a single `web` process is running.

Use `heroku open` to launch the app in the browser.

```term
$ heroku open
```

The browser should display the “Hello World” text defined previously. If it does not, or an error is present, [review and confirm the welcome page contents](#create-a-welcome-page). 

Heroku provides a default web URL for every application during development. When the application is ready to scale up for production, add a [custom domain](https://devcenter.heroku.com/articles/custom-domains).

## View Application Logs

The app logs are a valuable tool if the app is not performing correctly or generating errors.

View information about a running app using the Heroku CLI [logging command](logging), `heroku logs`. Here is example output:

```term
$ heroku logs
2021-09-16T14:46:32.344477+00:00 app[api]: Initial release by user developer@example.com2021-09-16T14:46:32.344477+00:00 app[api]: Release v1 created by user developer@example.com2021-09-16T14:46:32.488023+00:00 app[api]: Release v2 created by user developer@example.com2021-09-16T14:46:32.488023+00:00 app[api]: Enable Logplex by user developer@example.com2021-09-16T14:46:34.000000+00:00 app[api]: Build started by user developer@example.com2021-09-16T14:47:33.719594+00:00 app[api]: Release v3 created by user developer@example.com2021-09-16T14:47:33.719594+00:00 app[api]: Set LANG, RACK_ENV, RAILS_ENV, RAILS_LOG_TO_STDOUT, RAILS_SERVE_STATIC_FILES, SECRET_KEY_BASE config vars by user developer@example.com2021-09-16T14:47:34.976639+00:00 app[api]: Running release v4 commands by user developer@example.com2021-09-16T14:47:34.976639+00:00 app[api]: Attach DATABASE (@ref:postgresql-graceful-69657) by user developer@example.com2021-09-16T14:47:34.996598+00:00 app[api]: Release v5 created by user developer@example.com2021-09-16T14:47:34.996598+00:00 app[api]: @ref:postgresql-graceful-69657 completed provisioning, setting DATABASE_URL. by user developer@example.com2021-09-16T14:47:35.384307+00:00 app[api]: Deploy 409cdef5 by user developer@example.com2021-09-16T14:47:35.384307+00:00 app[api]: Release v6 created by user developer@example.com2021-09-16T14:47:35.406052+00:00 app[api]: Scaled to console@0:Free rake@0:Free web@1:Free by user developer@example.com2021-09-16T14:47:38.289544+00:00 heroku[web.1]: Starting process with command `bin/rails server -p ${PORT:-5000} -e production`
2021-09-16T14:47:39.000000+00:00 app[api]: Build succeeded
2021-09-16T14:47:41.593531+00:00 app[web.1]: => Booting Puma
2021-09-16T14:47:41.593548+00:00 app[web.1]: => Rails 7.0.0.alpha2 application starting in production
2021-09-16T14:47:41.593548+00:00 app[web.1]: => Run `bin/rails server --help` for more startup options
2021-09-16T14:47:42.701412+00:00 app[web.1]: Puma starting in single mode...
2021-09-16T14:47:42.701441+00:00 app[web.1]: * Puma version: 5.4.0 (ruby 3.0.2-p107) ("Super Flight")
2021-09-16T14:47:42.701442+00:00 app[web.1]: *  Min threads: 5
2021-09-16T14:47:42.701442+00:00 app[web.1]: *  Max threads: 5
2021-09-16T14:47:42.701442+00:00 app[web.1]: *  Environment: production
2021-09-16T14:47:42.701443+00:00 app[web.1]: *          PID: 4
2021-09-16T14:47:42.701650+00:00 app[web.1]: * Listening on http://0.0.0.0:32822
2021-09-16T14:47:42.706924+00:00 app[web.1]: Use Ctrl-C to stop
2021-09-16T14:47:43.226867+00:00 heroku[web.1]: State changed from starting to up
```

Append `-t`/`--tail` to the command to see a full, live stream of the app’s logs:

```term
$ heroku logs --tail
```

## Dyno Sleeping and Scaling

New applications are deployed to a free dyno by default. After a period of inactivity, free apps will "sleep" to conserve resources. For more on Heroku’s free dyno behavior, see [Free Dyno Hours](free-dyno-hours).

Upgrade to a hobby or professional dyno type as described in the [Dyno Types](dyno-types) article to avoid dyno sleeping. For example, migrating an app to a production dyno allows for easy scaling by using the Heroku CLI `ps:scale` command to instruct the Heroku platform to start or stop additional dynos that run the same `web` process type.

## The Rails Console

Use the Heroku CLI `run` command to trigger [one-off dynos](one-off-dynos) to run scripts and applications only when necessary. Use the command to launch a Rails console process attached to the local terminal for experimenting in the app's environment:

```term
$ heroku run rails console
irb(main):001:0> puts 1+1
2
```

The `run bash` Heroku CLI command is also helpful for debugging. The command starts a new one-off dyno with an interactive bash session.

## Rake Commands

Run `rake` commands (`db:migrate`, for example) using the `run` command exactly like the Rails console:

```term
$ heroku run rake db:migrate
```

## Configure The Web Server

By default, a Rails app's web process runs `rails server`, which uses Puma in Rails 7. Apps upgraded to Rails 7 need the `puma` gem added to the app’s `Gemfile`:

```ruby
gem 'puma'
```

After adding the `puma` gem, install it:

```term
$ bundle install
```

Rails 7 uses `config/puma.rb` to define Puma’s configuration and functionality with Puma installed. Heroku recommends reviewing [additional Puma configuration options](https://devcenter.heroku.com/articles/deploying-rails-applications-with-the-puma-web-server) to maximize the app’s performance. 

If `config/puma.rb` doesn’t exist, create one using [Heroku’s Puma documentation](https://devcenter.heroku.com/articles/deploying-rails-applications-with-the-puma-web-server) for maximum performance.

With Puma installed, use the `Procfile` to instruct Heroku on how to launch the Rails app on a dyno. 

### Create a Procfile

Change the command used to launch your web process by creating a file called [Procfile](procfile) inside the app’s root directory. Add the following line:

```
web: bundle exec puma -t 5:5 -p ${PORT:-3000} -e ${RACK_ENV:-development}
```

>note
>This file must be named `Procfile` exactly with a capital `P`, lowercase `rocfile`, and no file extension.

To use the Procfile locally, use the `local` Heroku CLI command.

In addition to running commands in the `Procfile`, `heroku local` can also manage environment variables locally through a `.env` file. Set `RACK_ENV` to `development` for the local environment and the `PORT` for Puma. Test with the `RACK_ENV` set to `production` before pushing to Heroku; `production` is the environment in which the Heroku app will run.

```term
$ echo "RACK_ENV=development" >>.env
$ echo "PORT=3000" >> .env
```

>note
>Another alternative to using environment variables locally with a `.env` file is the [dotenv](https://github.com/bkeepers/dotenv) gem.

Add `.env` to `.gitignore` since this is for local environment setup only.

```term
$ echo ".env" >> .gitignore
$ git add .gitignore
$ git commit -m "add .env to .gitignore"
```

Test the Procfile locally using [Foreman](run-your-app-locally-using-foreman)​​. Start the web server with `local`:

```term
$ heroku local
[OKAY] Loaded ENV .env File as KEY=VALUE Format
9:47:51 AM web.1 |  Puma starting in single mode...
9:47:51 AM web.1 |  * Puma version: 5.4.0 (ruby 3.0.2-p107) ("Super Flight")
9:47:51 AM web.1 |  *  Min threads: 5
9:47:51 AM web.1 |  *  Max threads: 5
9:47:51 AM web.1 |  *  Environment: development
9:47:51 AM web.1 |  *          PID: 61670
9:47:52 AM web.1 |  * Listening on http://0.0.0.0:3000
9:47:52 AM web.1 |  Use Ctrl-C to stop
```

A successful test will look similar to the previous example. Press `Ctrl+C` or `CMD+C` to exit and deploy the changes to Heroku:

```term
$ git add .
$ git commit -m "use puma via procfile"
$ git push heroku main || git push heroku master
```

Check `ps`. The `web` process is now using the new command specifying Puma as the web server:

```term
$ heroku ps
Free dyno hours quota remaining this month: 996h 54m (99%)
Free dyno usage for this app: 0h 0m (0%)
For more information on dyno sleeping and how to upgrade, see:
https://devcenter.heroku.com/articles/dyno-sleeping

=== web (Free): bundle exec puma -t 5:5 -p ${PORT:-3000} -e ${RACK_ENV:-development} (1)
web.1: starting 2021/09/16 09:48:14 -0500 (~ 5s ago)
```

The logs also reflect that Puma is in use.

```term
$ heroku logs
```

## Rails asset pipeline

When deploying to Heroku, there are several options for invoking the [Rails asset pipeline](http://guides.rubyonrails.org/asset_pipeline.html). Please review the [Rails 3.1+ Asset Pipeline on Heroku Cedar](rails-asset-pipeline) article for general information on the asset pipeline.

Rails 7 removed the `config.assets.initialize_on_precompile` option because it is no longer needed. Additionally, any failure in asset compilation will now cause the push to fail. For Rails 7 asset pipeline support, see the [Ruby Support](ruby-support#rails-5-x-applications) page.

## Troubleshooting

If an app deployed to Heroku crashes (`heroku ps` shows state `crashed`), review the app’s logs to determine what went wrong. The following section covers common causes of app crashes.

### Runtime Dependencies on Development or Test Gems

If a gem is missing during deployment, check the Bundler groups. Heroku builds apps without the `development` or `test` groups, and if the app depends on a gem from one of these groups to run, move it out of the group.

A common example is using the RSpec tasks in the `Rakefile`. The error often looks like this:

```term
$ heroku run rake -T
Running `bundle exec rake -T` attached to terminal... up, ps.3
rake aborted!
no such file to load -- rspec/core/rake_task
```
First, duplicate the problem locally by running `bundle install` without the development or test gem groups:

```term
$ bundle install --without development:test
…
$ bundle exec rake -T
rake aborted!
no such file to load -- rspec/core/rake_task
```

>note
>The `--without` option on bundler is sticky. To get rid of this option, run `bundle config --delete without`.

Fix the error by making these Rake tasks conditional during gem load. For example:

```ruby
begin
  require "rspec/core/rake_task"

  desc "Run all examples"

  RSpec::Core::RakeTask.new(:spec) do |t|
    t.rspec_opts = %w[--color]
    t.pattern = 'spec/**/*_spec.rb'
  end
rescue LoadError
end
```

Confirm it works locally, then push to Heroku.

## Next Steps

Congratulations! You deployed your first Rails 7 application to Heroku. Review the following articles next:

* Visit the [Ruby support category](/categories/ruby-support) to learn more about using Ruby and Rails on Heroku.
* The [Deployment category](/categories/deployment) provides a variety of powerful integrations and features to help streamline and simplify deployments.
