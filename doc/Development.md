# Development

## General Guidelines

* Please follow these style guides: https://github.com/styleguide

## HTML Guidelines

* Use input placeholder and a matching label with class ui-hidden-accessible: http://view.jquerymobile.com/master/demos/forms-label-hidden-accessible/

## JavaScript Guidelines

There is some sharing of JavaScript between the rails and phonegap apps. The
phonegap app loads JQuery, JQueryMobile, a phonegap index.js that does
initialization and myplaceonline.js which is shared between the two apps
(these are equivalent to application.js.coffee). After phonegap loads the
homepage, it will dynamically load the rest of the javascript files
(these are equivalent to application_extra.js.coffee). The JavaScript files
myplaceonline.js should have all code that is required offline and everything
else should go into myplaceonline_final.js or page-specific JS files.
When updating myplaceonline.js, update the version at the top of the file and
update the file in both apps and do rebuilds.

## Create App

```
# Create new rails app
$ mkdir -p src/rails; cd src/rails; rails new myplaceonline --git --database=postgresql; cd myplaceonline
$ git mv config/database.yml config/database.yml.example
# Add config/database.yml to .gitignore
$ cp config/database.yml.example config/database.yml
# Edit config/database.yml and uncomment & change development/username,password,host
$ bin/rake db:create db:migrate
$ bin/rails server
# Add to Gemfile: gem 'jquery_mobile_rails'
$ bin/bundle install
# Edit app/assets/javascripts/application.js
#  Add: //= require jquery.mobile
#  Remove: //= require turbolinks
# Edit app/assets/stylesheets/application.css
#  Add after require_self: *= require jquery.mobile
$ bin/rails generate controller welcome index
# Edit config/routes.rb
#  Uncomment: root 'welcome#index'
# Edit app/views/layouts/application.html.erb
#  Remove all instances of data-turbolinks-track=true
```

## Rails

```
puts object.inspect
```

## Ruby on Rails

```
$ bin/rails server
$ bin/rake -T
$ bin/rake routes
$ bin/rails generate controller ${CONTROLLER} ${ACTION}
$ bin/rails generate model ${MODEL}
$ bin/rake db:migrate
$ bin/rake db:reset
$ bin/bundle show # Show gem versions
$ bin/bundle update # Update gems
$ bin/rails generate migration AddPointsToIdentities points:integer
$ RAILS_ENV=test bin/rake db:reset test
```

### Create New Category Example

```
# Add to config/locales/en.yml:
  myplaceonline:
    category:
      passwords: "Passwords"
# Add to db/seeds.rb:
passwords = Category.create(name: "passwords", link: "passwords", position: 0, parent: order)
# Create a migration:
$ bin/rails generate migration AddCategoryPasswords
# Edit the new migration:
  def change
    passwords = Category.create(name: "passwords", link: "passwords", position: 0, parent: Category.find_by_name("order"))
  end
# Run migrate
$ bin/rake db:migrate
```

### Create New Page Example

```
$ bin/rails generate controller joy index
# Add to config/routes.rb:
match '/joy', :to => 'joy#index', via: :get
# Add to config/locales/en.yml
    joy:
      title: "Joy"
# Edit app/views/joy/index.html.erb
<% content_for :heading do -%><%= t('myplaceonline.joy.title') %><% end -%>
<h1><%= t('myplaceonline.joy.title') %></h1>
```

## PostgreSQL

```
# Basic Usage
$ psql -U user1 -h localhost -d myplaceonline_development
# List databases: \l
# List users: \du
# Connect to database: \c ${DB}
# List tables in database: \dt
# Describe table: \d ${TABLE}
# Dump database: pg_dump ${DB} > ${FILE}.sql
# CREATE DATABASE ${DB} WITH OWNER ${USER} ENCODING 'UTF8';
# GRANT ALL PRIVILEGES ON DATABASE ${DB} TO ${USER};
# DROP DATABASE ${DB};
```

## Add submodule

Example:

```
$ git submodule add git@github.com:myplaceonline/roo.git src/roo
$ cd src/roo
$ irb -rubygems -I lib -r roo.rb
```


D, [2014-12-22T20:10:52.106603 #8450] DEBUG -- :   ESC[1mESC[36mSQL (0.2ms)ESC[0m  ESC[1mINSERT INTO "passwords" ("account_number", "created_at", "encrypted_password_id", "identity_id", "is_encrypted_password", "name", "notes", "updated_at", "url", "user") VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10) RETURNING "id"ESC[0m  [["account_number", "Access Key ID: 0RZ4CZVS3GYYSFPCFVG2"], ["created_at", "2014-12-22 20:10:52.105529"], ["encrypted_password_id", 17], ["identity_id", 1], ["is_encrypted_password", "t"], ["name", "Amazon EC2"], ["notes", "Private Key File: pk-M2L4BE76PKA332T5MYM2JT6NU6OCY6V3.pem\nX.509 Certificate File: cert-M2L4BE76PKA332T5MYM2JT6NU6OCY6V3.pem"], ["updated_at", "2014-12-22 20:10:52.105529"], ["url", ""], ["user", "schizoidboy@gmail.com"]]
D, [2014-12-22T20:10:52.531051 #8450] DEBUG -- :   ESC[1mESC[35m (1.0ms)ESC[0m  ROLLBACK
I, [2014-12-22T20:10:52.533020 #8450]  INFO -- : Completed 500 Internal Server Error in 32679ms
F, [2014-12-22T20:10:52.534751 #8450] FATAL -- : 
ArgumentError (data must not be empty):
  lib/myp.rb:147:in `encrypt_value'
  lib/myp.rb:138:in `encrypt'
  lib/myp.rb:133:in `encrypt_from_session'
  app/controllers/passwords_controller.rb:307:in `add_secret'
  app/controllers/passwords_controller.rb:249:in `block (2 levels) in importodf3'
  app/controllers/passwords_controller.rb:209:in `block in importodf3'
  app/controllers/passwords_controller.rb:207:in `importodf3'