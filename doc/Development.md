# Development

## General

* Site tests:
  * https://www.google.com/webmasters/tools/mobile-friendly/

## Architecture

                                               web1
                                            +--------+            +---------+
                                            + nginx  +            +         |
                                         XXXX  RoR   XXXXXXXXXXXXXX   db1   |
                       frontend2       XXX  +        +        XX  +         |
    +--------+       +-----------+   XXX    +--------+       XX   +----+----+
    |        |       |           + XXX                     XXX         | Streaming
    |  user  +------->  haproxy  XXXX                     XX           | Replication
    |        |       |           +  XX         web2      XX            |
    +--------+       +-----------+   XXX    +--------+  XX        +----v----+
                                       XXX  + nginx  +  X         |         |
                                         XXXX  RoR   XXXX         |   db2   |
                                            +        +            |         |
                                            +--------+            +---------+
    
    
                                     admin             sendgrid
                                  +----------+        +---------+
                                  |          |        |         |
                                  |   chef   |        |  email  |
                                  |          |        |         |
                                  +----------+        +---------+

* Made with ASCIIFlow Infinity: http://asciiflow.com/

### High Availability

* The database is backed up asynchronously using PostgreSQL streaming
  replication; however, it is not currently a hot-standy.
* Some user-uploaded files are stored on the filesystem instead of in the
  database. These are stored on the primary database server and the web servers
  read/write over NFS. This NFS share is rsync'ed nightly to the backup
  database server.
* If the frontend server is unavailable, create another frontend server
  and then change the DigitalOcean Floating IP address to point to it.

### Administration

* HAProxy statistics (admin/{passwords/haproxy/stats}): https://myplaceonline.com:9443/
  * /var/log/haproxy.log
  * "8.2.3. HTTP log format" in http://www.haproxy.org/download/1.7/doc/configuration.txt
* Chef administration: https://admin.myplaceonline.com/
* ElasticSearch:
  * curl http://db2-internal.myplaceonline.com:9200/_stats?pretty=1
* Grafana: https://admin.myplaceonline.com:3000/

### Logs

* Central syslog server: db2.myplaceonline.com
** tail -f /var/log/messages | grep -v -e audit: -e telegraf: -e STATS

* Linux
** `atop -r` and use `t` and `T` to move forward/backward, and `b` to jump to time
** Crashes in /var/crash/
* Rails
** Most things rsyslog'd to db2 (/var/log/messages); however, Rails app logging doesn't support syslog, so it goes to /var/log/messages
** /var/www/html/myplaceonline/log/passenger.log
** tail -f /var/log/messages | grep rails
* ElasticSearch
** curl http://db2-internal.myplaceonline.com:9200/_cluster/stats?pretty
** /var/log/elasticsearch/

## Design Goals

* Mobile/tablet first design. Single Page Application (SPA): http://docs.phonegap.com/en/3.5.0/guide_next_index.md.html

## HTML

* Use input placeholder and a matching label with class ui-hidden-accessible: http://view.jquerymobile.com/master/demos/forms-label-hidden-accessible/

### Offline Usage

* We can't use a simple [cache manifest](https://developer.mozilla.org/en-US/docs/Web/HTML/Using_the_application_cache)
  that caches HTML pages because that might cache sensitive information
  such as decrypted passwords.

## Autocomplete

* "Standard": https://html.spec.whatwg.org/multipage/forms.html#autofilling-form-controls:-the-autocomplete-attribute
* Chrome: https://bugs.chromium.org/p/chromium/issues/detail?id=468153#c164
* Firefox: https://bugzilla.mozilla.org/show_bug.cgi?id=956906
* IE: https://blogs.msdn.microsoft.com/ieinternals/2013/09/24/ie11-changes/

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

### Encryption Algorithms

There are many parts to encryption: key exchange, authentication, symmetric
ciphers, message authentication codes, hashing, etc. The information on which
ones to use in which situations is confusing, contradictory, and constantly
changing. These are some findings after reading many articles, but they are
amateur findings and need more investigation and citation:

In general:
* Avoid MD5, SHA1, elliptic curve+NIST, bit sizes less than 2048, block sizes
  less than 128 bits, DES, RC4, ECB cipher mode, CBC cipher mode with
  non-random IV, CTR mode with repeating IV, MAC-then-encrypt, encrypt-and-MAC.
* Prefer forward secrecy (e.g. ephemeral), Authenticated Encryption with
  Associated Data (AEAD) Cipher (e.g. GCM).

Therefore:
* DHE+SHA2 or ECDH+Curve25519+SHA2
* RSA/4069 or Ed25519
* AES-128-GCM or AES-256-GCM or ChaCha20-poly1305
* HMAC+SHA2+ETM (unless using GCM which is already authenticated)

References:
* "This presents a problem for cipher constructions with data-dependent padding
  (such as CBC). TLS 1.3 removes the length field and relies on the AEAD cipher"
  (http://tlswg.github.io/tls13-spec/)
  
The future appears to be Dan Bernstein's Curve25519/EdDSA/Poly1305/ChaCha20

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

## Rails

1. The basic flow of views is app/views/${CATEGORY}/_form.html.erb includes
   app/views/shared/_model_form.html.erb passing in obj: @obj as a local.
   Normally the view would just reference the @obj member variable of the 
   controller directly, but we use the obj:@obj model so that one form
   can include another form (usually a belongs_to relationship).
2. Use semantic names. For example, if you have the date of a weight
   measurement, use a field named measurement_start instead of just start. The
   reason for this is that, by default, we use form field names matching the
   model field name, so the autocomplete will be more specific if the field
   name is more specific.
3. belongs_to: In the class that has the foreign key.
   has_one: If the other class has the foreign key.
4. MyplaceonlineController supports an "insecure" mode where items can be
   added without needing to re-enter a password (just a remember me cookie is
   needed). Add in the protected section.
    def insecure
      true
    end

5. Order has_many example:
   has_many :job_salaries, -> { order('started DESC') }, :dependent => :destroy
6. Is not null example:
   IdentityDriversLicense.where("identity_id = ? and expires is not null and expires < ?", user.primary_identity, threshold)
7. Enumeration:
   controller: Add permit param
   model:
      CONTACT_TYPES = [
        ["myplaceonline.contacts.best_friend", 0],
        ["myplaceonline.contacts.good_friend", 1],
        ["myplaceonline.contacts.acquiantance", 2],
        ["myplaceonline.contacts.business_contact", 3],
        ["myplaceonline.contacts.best_family", 4],
        ["myplaceonline.contacts.good_family", 5]
      ]
   _form: <%= myp_select(f, :dimensions_type, "myplaceonline.recreational_vehicles.dimensions_type", Myp.translate_options(Myp::DIMENSIONS), obj.dimensions_type) %>
   show: <%= attribute_table_row_select(t("myplaceonline.recreational_vehicles.dimensions_type"), @obj.dimensions_type, Myp::DIMENSIONS) %>
   filter:
        <div class="horizontal_center" data-role="collapsible">
          <h4><%= t("myplaceonline.general.filter") %></h4>
          <%= myp_select_tag(:program_type, "myplaceonline.reward_programs.program_type", Myp.translate_options(RewardProgram::REWARD_PROGRAM_TYPES), @program_type, false, nil, false, "refreshWithParam('program_type', $('#program_type').val())") %>
        </div>
     controller:
        def index
          @contact_type = params[:contact_type]
          if !@contact_type.blank?
            @contact_type = @contact_type.to_i
          end
          super
        end

        def all
          if @program_type.blank?
            model.where(
              identity_id: current_user.primary_identity.id
            )
          else
            model.where(
              identity_id: current_user.primary_identity.id,
              program_type: @program_type
            )
          end
        end
8. List of Files/Pictures
  $ bin/rails generate model quest_file quest:references:index identity_file:references:index identity:references:index position:integer
  $ bin/rake db:migrate
  $ cp app/models/quest_file.rb app/models/${X}
  
  controller:

    # public

    def may_upload
      true
    end

    quest_files_attributes: FilesController.multi_param_names
    
  model:
    include AllowExistingConcern

    has_many :quest_files, :dependent => :destroy
    accepts_nested_attributes_for :quest_files, allow_destroy: true, reject_if: :all_blank
    allow_existing_children :quest_files, [{:name => :identity_file}]

    before_validation :update_file_folders
    
    def update_file_folders
      put_files_in_folder(quest_files, [I18n.t("myplaceonline.category.quests"), display])
    end
  config/locales
    quests:
      files: "Files/Pictures"
      file: "File/Picture"
      add_file: "Add File/Picture"
      add_files: "Add File(s)/Picture(s)"
      delete_file: "Delete File/Picture"
  _form
    <%=
      render partial: "myplaceonline/pictures_form", locals: {
        f: f,
        obj: obj,
        position_field: :position,
        pictures_field: :quest_files,
        item_placeholder: "myplaceonline.quests.file",
        heading: "myplaceonline.quests.files",
        addbutton: "myplaceonline.quests.add_file",
        addbutton_multi: "myplaceonline.quests.add_files",
        deletebutton: "myplaceonline.quests.delete_file"
      }
    %>
  show
    <%=
      render partial: 'myplaceonline/pictures', locals: {
        pics: @obj.quest_files,
        placeholder: "myplaceonline.quests.file"
      }
    %>
9. Add model initialization code
  def self.build(params = nil)
    result = self.dobuild(params)
    # initialize result here
    result
  end
10. Add category filter text
  $ bin/rails generate migration AddCategoryFiltertext
  Myp.migration_add_filtertext("$CATEGORY", "$SPACE_DELIMITED_ADDITIONS")
11. Transaction
  ActiveRecord::Base.transaction do
  end
12. Logging
  Rails.logger.debug{"test"}
13. Rebuild index
  $ bin/rails generate migration RebuildIndex001
  UserIndex.reset!

### Jobs

bin/delayed_job run --exit-on-complete

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

* GC: https://github.com/ruby/ruby/blob/trunk/gc.c#L7373

#### Create New Sub-Category Example

```
# Stop rails server if running
# ${X} is usually plural here in capital camel case:
# Create a migration:
$ bin/rails generate migration AddCategory${X}
# Edit the new migration (${X} is all lowercase here and underscores):
  def change
    Category.create(name: "${X}", link: "${X}", position: 0, parent: Category.find_by_name("${Y}"), icon: "FatCow_Icons16x16/check_box_uncheck.png")
  end
$ bin/rake db:migrate
# Add to config/locales/en.yml (first ${X} is lowercase, second one is usually capitalized):
  myplaceonline:
    category:
      ${X}: "${X}"
$ cp -R app/views/order app/views/${$}
$ cp app/controllers/order_controller.rb app/controllers/${X}_controller.rb and replace what's necessary
# Add to config/routes.rb
  get '${X}/index'
  get '${X}', :to => '${X}#index'
$ RAILS_ENV=development bin/rake myp:dump
$ RAILS_ENV=test bin/rake db:drop db:create db:schema:load db:seed myp:reload_categories test
# Start rails server
```

#### Create New Leaf Category Example

```
# ${X} is usually plural here in capital camel case:
# Create a migration:
$ bin/rails generate migration AddCategory${X}
# Edit the new migration (${X} is all lowercase here and usually plural and underscores):
  def change
    Category.create(name: "${X}", link: "${X}", position: 0, parent: Category.find_by_name("${Y}"), icon: "FatCow_Icons16x16/check_box_uncheck.png")
  end
# Add to config/locales/en.yml (first ${X} is lowercase, second one is usually capitalized):
  myplaceonline:
    category:
      ${X}: "${X}"
# ${X} is non-plural, lower-case and underscores instead of camel case:
$ bin/rails generate scaffold ${X} ${COLUMNS} visit_count:integer identity:references:index
# x:string x:text 'x:decimal{10,2}' x:integer x:decimal x:float x:boolean x:binary x:date x:time x:datetime
# Example:
# bin/rails generate scaffold wisdom name:string wisdom:text identity:references:index
# You'll get the following warning and you should answer 'Y':
  conflict    app/assets/stylesheets/scaffolds.css.scss
  Overwrite /work/myplaceonline/src/src/myplaceonline_rails/app/assets/stylesheets/scaffolds.css.scss? (enter "h" for help) [Ynaqdh] Y
$ git checkout -- app/assets/stylesheets/scaffolds.css.scss
# Run migrate
$ bin/rake db:migrate
# Edit app/models/identity.rb
  has_many :${X}, :dependent => :destroy
      :${X} => ${X}.to_a.sort{ |a,b| a.name.downcase <=> b.name.downcase }.map{|x| x.as_json},
$ X=...
# Change after: cp app/controllers/wisdoms_controller.rb app/controllers/${X}_controller.rb
$ rm app/views/${X}/*jbuilder
# Create a myplaceonline.${X} section config/locales/en.yml based on myplaceonline.wisdom
# cp app/views/wisdoms/* app/views/${X} and replace all instances of wisdom with ${X}
# Edit config/routes.rb and add after resources ${X}
  post '${X}/new'
# Replace ${X} with singular version: cp app/models/wisdom.rb app/models/${X}.rb
# Edit tests/fixtures/${X}.yml and create a fixture with a name of ${X} (see wisdoms.yml)
# cp test/controllers/wisdoms_controller_test.rb test/controllers/${X}_controller_test.rb
$ RAILS_ENV=development bin/rake myp:dump
$ pkill -9 -f "spring.*test mode"
$ RAILS_ENV=test SKIP_LARGE_IMPORTS=true FTS_TARGET=localhost:9200 bin/rake db:drop db:create db:schema:load db:seed myp:reload_categories test
# Add migration with UserIndex.reset!
$ bin/rails generate migration ResetSearch001
  def change
    UserIndex.reset!
  end
# Run migrate
$ bin/rake db:migrate
```

#### Add encrypted column(s)

```
$ bin/rails generate migration AddEncryptionTo${MODEL} ${COLUMN}_encrypted:references:index
    add_reference :ssh_keys, :ssh_private_key_encrypted, index: true, foreign_key: false
    add_foreign_key :ssh_keys, :encrypted_values, column: :ssh_private_key_encrypted_id
$ bin/rake db:migrate
# Add to model:
  include EncryptedConcern
  belongs_to :${COLUMN}_encrypted, class_name: EncryptedValue, dependent: :destroy, :autosave => true
  belongs_to_encrypted :${COLUMN}
  before_validation :${COLUMN}_finalize
# Change any validations to check both the encrypted and unencrypted values:
  validate do
    if ${COLUMN}.blank? && ${COLUMN}_encrypted.nil?
      errors.add(:${COLUMN}, t("myplaceonline.general.non_blank"))
    end
  end
# Remove potentially unencrypted values from JSON
  def as_json(options={})
    if ${COLUMN}_encrypted?
      options[:except] ||= %w(${COLUMN})
    end
    super.as_json(options)
  end
# Add :encrypt to controller permitted params
# Add to controller:
  protected
    def sensitive
      true
    end
    
    def before_edit
      @obj.encrypt = @obj.${COLUMN}_encrypted?
    end
# Add to _form.html.erb
<%= myp_check_box_tag :encrypt, "myplaceonline.general.encrypt", @encrypt %>
```

#### Change Category Name

```
$ bin/rails generate migration Rename${OLD}To${NEW}
rename_table :${OLD}, :${NEW}
$ bin/rails generate migration ChangeCategory${OLD}
    cat = Category.where(name: "${OLD}").take!
    cat.name = "${NEW}"
    cat.link = "${NEW}"
    cat.additional_filtertext = "${OLD}"
    cat.save!
# Change config/locales/en.yml
$ git mv app/views/${OLD}/ app/views/${NEW}/
$ git mv app/controllers/${OLD}_controller.rb app/controllers/${NEW}_controller.rb
$ git mv app/models/${OLD}.rb app/models/${NEW}.rb # SINGULAR
$ git mv app/helpers/${OLD}_helper.rb app/helpers/${NEW}_helper.rb
$ git mv test/controllers/${OLD}_controller_test.rb test/controllers/${NEW}_controller_test.rb
$ git mv test/factories/${OLD}.rb test/factories/${NEW}.rb
$ git mv test/fixtures/${OLD}.yml test/fixtures/${NEW}.yml
$ git mv test/helpers/${OLD}_helper_test.rb test/helpers/${NEW}_helper_test.rb
$ git mv test/models/${OLD}_test.rb test/models/${NEW}_test.rb # SINGULAR
$ git mv app/assets/javascripts/${OLD}.js.coffee app/assets/javascripts/${NEW}.js.coffee
$ git mv app/assets/stylesheets/${OLD}.css.scss app/assets/stylesheets/${NEW}.css.scss
$ bin/rake db:migrate
# Change config/routes.rb
# Change app/models/identity.rb
# If the model has reminders, check due_item.rb
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

### Add additional filtertext to category

```
$ bin/rails generate migration UpdateFilterTextForCategory
Myp.migration_add_filtertext("vehicles", "truck")
```

## PostgreSQL Tips

```
# Basic Usage
$ psql -U myplaceonline -h localhost -d myplaceonline_development
# List databases: \l
# List users: \du
# Connect to database: \c ${DB}
# List tables in database: \dt
# Describe table: \d ${TABLE}
# Dump database: pg_dump ${DB} > ${FILE}.sql
# CREATE DATABASE ${DB} WITH OWNER ${USER} ENCODING 'UTF8';
# GRANT ALL PRIVILEGES ON DATABASE ${DB} TO ${USER};
# DROP DATABASE ${DB};
# Equivalent to MySQL \G: \x before the command
# Backup: pg_dump -U postgres -h localhost -d myplaceonline_production > backup_`date +%Y%m%d%H%M%S`.sql
# Restore:
$ gpg --output tmp.sql --decrypt file.sql.pgp
$ dropdb -U myplaceonline -h localhost  myplaceonline_development; createdb -U myplaceonline -h localhost myplaceonline_development; pg_restore -U myplaceonline -h localhost -d myplaceonline_development -n public tmp.sql; rm -f tmp.sql
$ bin/rails c
# UserIndex.reset!
```

## Git Tips

#### Add submodule

Example:

```
$ git submodule add https://github.com/myplaceonline/myplaceonline_ffclipboard.git src/myplaceonline_ffclipboard
$ cd src/myplaceonline_ffclipboard
$ export NAME="Name"
$ export EMAIL="name@example.com"
$ git config --replace-all user.name "${NAME}"
$ git config --replace-all user.email "${EMAIL}"
$ cd ../..
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

# PhoneGap

## Building New Version

1. Bump app version in config.xml
1. Commit any changes in src/myplaceonline_phonegap
1. Go to https://build.phonegap.com/
1. Login with Adobe ID
1. Click Update Code > Pull Latest
1. Download APKs to lib/android/builds
1. Update APK at https://play.google.com/apps/publish/

# CKEditor

* Remove
  * About CKEditor
* Add
  * Blockquote
  * Editor Resize
  * Format
  * Maximize
  * Paste As Plain Text
  * Remove Format
  * Source Editing Area
  * Tab Key Handling

Build new version
* cd /work/myplaceonline/ckeditor-dev
* git remote add upstream ssh://git@github.com/ckeditor/ckeditor-dev
* git fetch upstream
* git rebase upstream/master
* git push -f origin master
# Make changes
* git commit -a -m "Pull request #237: Fix additional null dereferences"
* git push
* dev/builder/build.sh --leave-js-unminified -s
* cd ../ckeditor
* git remote add upstream ssh://git@github.com/galetahub/ckeditor
* git fetch upstream
* git reset --hard 0ebc23b5c512b1ceb6de58a2616cacf25ec5c2f6
* git rebase upstream/master
* git push -f origin master
* git rm -rf ./vendor/assets/javascripts/ckeditor
* git commit -a -m "Remove galetahub/ckeditor version of ckeditor"
* mkdir -p vendor/assets/javascripts/; cd vendor/assets/javascripts/
* cp ../../../../ckeditor-dev/dev/builder/release/ckeditor_*.tar.gz .
* tar xzvf ckeditor_*.tar.gz
* rm ckeditor_*.tar.gz
* rm -rf ckeditor/samples/
* git add ../../
* git commit -a -m "Add ckeditor build from myplaceonline/ckeditor-dev"
* git push
* cd $MYPLACEONLINE_SRC
* bin/bundle update

# Server Administration

## Reboot

### Web Server

NODE=web1; ssh root@${NODE}.myplaceonline.com "systemctl stop nginx; reboot"

## Analyze Crash

http://averageradical.github.io/Linux_Core_Dumps.pdf

    $ /usr/local/src/crash-*/crash /usr/lib/debug/lib/modules/4*/vmlinux /var/crash/*/vmcore
    # ps
    # http://averageradical.github.io/Linux_Core_Dumps.pdf
    
* strings /var/crash/*/vmcore | grep "Linux version"
* dnf list installed | grep kernel-debuginfo

## Kibana

* http://localhost:5601/
* Search: _id:1004 AND _type:museum
