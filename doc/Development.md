# Development

## General

* Please follow these style guides: https://github.com/styleguide

## Design Goals

* Mobile/tablet first design. Single Page Application (SPA): http://docs.phonegap.com/en/3.5.0/guide_next_index.md.html

## HTML

* Use input placeholder and a matching label with class ui-hidden-accessible: http://view.jquerymobile.com/master/demos/forms-label-hidden-accessible/

### Offline Usage

* We can't use a simple [cache manifest](https://developer.mozilla.org/en-US/docs/Web/HTML/Using_the_application_cache)
  that caches HTML pages because that might cache sensitive information
  such as decrypted passwords.

## JavaScript

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

## Encryption

Values are encrypted using a symmetric cipher with the user's password. When
a user changes their password, we need to update all encrypted values. To avoid
any requirement of registering each encrypted value in each model, we simply
use a foreign key value to the encrypted values table.

When a model has a potentially encrypted value, it will have two columns:
one is a plain text value and the other is the foreign key reference to the
encrypted values table. We can check if a value is encrypted or not by checking
whether or not the encrypted value reference is nil or not.

### Why not encrypt all data?

One option is to encrypt all data (or at least make that a user preference).
One potential problem with this is simply the overhead. For example, looking
at passwords, it's one thing to encrypt the password which will only be
decrypted when a particular password is shown, but if even the name of the
service the password is for is encrypted, that would need to be decrypted
on a simple password list or search.

One option to reduce the overhead of such decryption is to have a cache
from (encrypted_data,salt)->(decrypted_data), but that might not be scalable
if every piece of data was encrypted.

## TODO

* Internationalize devise views
* Make sure roo temporary files are deleted
* Make sure accepts_nested_attributes_for can't view/delete unrelated items
* Fix protect_from_forgery (in application controller) interaction with JQueryMobile (http://guides.rubyonrails.org/security.html#cross-site-request-forgery-csrf)
* Consider http://unlicense.org/
* File download:
  * http://stackoverflow.com/questions/18413595/how-to-download-a-file-from-php-page-using-phonegap-android-platform
  * http://stackoverflow.com/questions/9523736/problems-with-download-link-in-phonegap-android
  * https://github.com/apache/cordova-plugin-file-transfer/blob/master/doc/index.md
  * https://github.com/apache/cordova-plugin-file/blob/master/doc/index.md
* No results text for searches:
  * http://stackoverflow.com/questions/9685921/jquery-mobile-data-filter-in-case-of-no-result
  * http://stackoverflow.com/questions/22292250/jquery-mobile-listview-if-there-is-no-result

## Miscellaneous Notes

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
# Stop rails server if running
# ${X} is usually plural here:
# Create a migration:
$ bin/rails generate migration AddCategory${X}
# Edit the new migration (${X} is all lowercase here and usually plural):
  def change
    Category.create(name: "${X}", link: "${X}", position: 0, parent: Category.find_by_name("${Y}"))
  end
# Add to config/locales/en.yml (first ${X} is lowercase, second one is usually capitalized):
  myplaceonline:
    category:
      ${X}: "${X}"
# Add to db/seeds.rb (${X} is all lowercase here and usually plural):
  ${X} = Category.create(name: "${X}", link: "${X}", position: 0, parent: ${Y})
# ${X} is non-plural and capitalized here:
$ bin/rails generate scaffold ${X} ${COLUMNS} identity:references:index
# Example:
# bin/rails generate scaffold Wisdom name:string wisdom:text identity:references:index
# You'll get the following warning and you should answer 'Y':
  conflict    app/assets/stylesheets/scaffolds.css.scss
  Overwrite /work/myplaceonline/src/src/myplaceonline_rails/app/assets/stylesheets/scaffolds.css.scss? (enter "h" for help) [Ynaqdh] h
$ git checkout -- app/assets/stylesheets/scaffolds.css.scss
# Run migrate
$ bin/rake db:migrate
# Edit lib/myp.rb
  @@all_categories[:${X}] = Category.find_by(:name => :${X})
# Edit app/models/identity.rb
  has_many :${X}, :dependent => :destroy
      :${X} => ${X}.to_a.sort{ |a,b| a.name.downcase <=> b.name.downcase }.map{|x| x.as_json},
# Change app/controllers/${X}Controller.rb based on WisdomController.rb
$ rm app/views/${X}/*jbuilder
# Create a myplaceonline.${X} section config/locales/en.yml based on myplaceonline.wisdom
# Copy app/views/wisdom/* over to app/views/${X} and replace all instances of wisdom with ${X}
# Edit config/routes.rb and add after resources ${X}
  post '${X}/new'
# Edit app/models/${X}.rb and add any validations
# Edit app/models/ability.rb and add a line:
  can :manage, ${X}, :identity => identity
# Edit app/tests/fixtures/${X}.yml and create a fixture with a name of ${X} (see wisdoms.yml)
# Edit app/tests/controllers/${X}_controller_test.rb and base it off of wisdoms_controller_test.rb
$ RAILS_ENV=test bin/rake db:reset
$ bin/rake test
# Start rails server
```

#### Create a Rails App

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

#### Create New Page Example

```
# ${X} is all lowercase here:
$ bin/rails generate controller ${X} index
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

# Testing

## Notes

* Test data in test/fixtures/*.yml
* Devise user logged in for ever test in test/test_helper.rb setup

## Running Tests

```
$ RAILS_ENV=test bin/bundle exec rake db:setup
$ bin/rake
```
