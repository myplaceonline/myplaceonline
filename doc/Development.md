# Development

## General Guidelines

* Please follow these style guides: https://github.com/styleguide

## Design Goals

* Mobile/tablet first design. Single Page Application (SPA): http://docs.phonegap.com/en/3.5.0/guide_next_index.md.html
* Choose what information is stored offline in local storage and synchronize when internet available.

## HTML Guidelines

* Use input placeholder and a matching label with class ui-hidden-accessible: http://view.jquerymobile.com/master/demos/forms-label-hidden-accessible/

### Offline Usage

* We can't use a simple [cache manifest](http://dev.w3.org/html5/pf-summary/offline.html#manifests) that caches all HTML pages because
  that might cache sensitive information such as decrypted passwords.

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

## TODO

* Internationalize devise views
* Single re-login per session to modify settings
* Fix protect_from_forgery (in application controller) interaction with JQueryMobile (http://guides.rubyonrails.org/security.html#cross-site-request-forgery-csrf)

## Miscellaneous Notes

### Create a Rails App

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

### Rails Tips

```
puts object.inspect
```

### Ruby on Rails Tips

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

#### Create New Category Example

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

#### Create New Page Example

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

### PostgreSQL Tips

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

### Git Tips

#### Add submodule

Example:

```
$ git submodule add git@github.com:myplaceonline/roo.git src/roo
$ cd src/roo
$ irb -rubygems -I lib -r roo.rb
```
