# Notes

Various development notes.

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
$ bin/rake routes
$ bin/rails generate controller ${CONTROLLER} ${ACTION}
$ bin/rails generate model ${MODEL}
$ bin/rake db:migrate
$ bin/rake db:reset
$ bin/bundle show # Show gem versions
$ bin/bundle update # Update gems
$ bin/rails generate migration AddPointsToIdentities points:integer
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
