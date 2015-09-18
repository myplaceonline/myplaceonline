# Development

## General

* Please follow these style guides: https://github.com/styleguide
* Site tests:
** https://www.google.com/webmasters/tools/mobile-friendly/

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
* http://stackoverflow.com/a/15746205/4135310

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
   needed).
5. Order has_many example:
   has_many :job_salaries, -> { order('started DESC') }, :dependent => :destroy
6. Is not null example:
   IdentityDriversLicense.where("owner_id = ? and expires is not null and expires < ?", user.primary_identity, threshold)
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
              owner_id: current_user.primary_identity.id
            )
          else
            model.where(
              owner_id: current_user.primary_identity.id,
              program_type: @program_type
            )
          end
        end
8. List of Pictures
  $ bin/rails generate model vehicle_picture vehicle:references:index identity_file:references:index owner:references:index
  $ bin/rake db:migrate
  $ cp app/models/vehicle_picture.rb app/models/${X}
  controller:
    def may_upload
      true
    end

    vehicle_pictures_attributes: [
      :id,
      :_destroy,
      identity_file_attributes: [
        :id,
        :file,
        :notes
      ]
    ]
    
    def presave
      @obj.vehicle_pictures.each do |pic|
        if pic.identity_file.folder.nil?
          pic.identity_file.folder = IdentityFileFolder.find_or_create([I18n.t("myplaceonline.category.recreational_vehicles"), @obj.display])
        end
      end
    end
  config/locales
    vehicles:
      pictures: "Pictures"
      picture: "Picture"
      add_picture: "Add Picture"
      delete_picture: "Delete Picture"
  _form
    <%=
      render layout: 'myplaceonline/childproperties', locals: {
        f: f,
        heading: t("myplaceonline.vehicles.pictures"),
        childpropertiesname: :vehicle_pictures,
        childproperties: obj.vehicle_pictures,
        deletebutton: t("myplaceonline.vehicles.delete_picture"),
        addbutton: t("myplaceonline.vehicles.add_picture"),
        expanded: obj.vehicle_pictures.length > 0,
        formjson: [
          {
            type: 'file',
            name: 'identity_file_attributes.file',
            placeholder: t("myplaceonline.vehicles.picture")
          }
        ]
      } do |child_fields, childproperty|
    %>
      <%= child_fields.fields_for :identity_file, childproperty.identity_file do |file_fields| %>
        <%= myp_file_field file_fields, :file, "myplaceonline.vehicles.picture", childproperty.identity_file.file %>
      <% end %>
    <% end %>
  show
    <% @obj.vehicle_pictures.each do |pic| %>
      <%= table_row_heading(t("myplaceonline.vehicles.picture")) %>
      <% if !pic.identity_file.nil? && !pic.identity_file.file_content_type.nil? && (pic.identity_file.file_content_type.start_with?("image")) %>
        <%= attribute_table_row_image(t("myplaceonline.vehicles.picture"), file_view_path(pic.identity_file)) %>
      <% end %>
    <% end %>
  model
    has_many :vehicle_pictures, :dependent => :destroy
    accepts_nested_attributes_for :vehicle_pictures, allow_destroy: true, reject_if: :all_blank
9. Add model initialization code
  def self.build(params = nil)
    result = super(params)
    # initialize result here
    result
  end


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

#### Create New Sub-Category Example

```
# Stop rails server if running
# ${X} is usually plural here in capital camel case:
# Create a migration:
$ bin/rails generate migration AddCategory${X}
# Edit the new migration (${X} is all lowercase here and underscores):
  def change
    Category.create(name: "${X}", link: "${X}", position: 0, parent: Category.find_by_name("${Y}"))
  end
$ bin/rake db:migrate
# Add to config/locales/en.yml (first ${X} is lowercase, second one is usually capitalized):
  myplaceonline:
    category:
      ${X}: "${X}"
# Add to db/seeds.rb (${X} is all lowercase here and usually plural):
  ${X} = Category.create(name: "${X}", link: "${X}", position: 0, parent: ${Y})
$ mkdir app/views/${X}/
$ cp app/views/order/* app/views/${X}/ and replace the translation name
$ cp app/controllers/order_controller.rb app/controllers/${X}_controller.rb and replace what's necessary
# Add to config/routes.rb
  get '${X}/index'
  get '${X}', :to => '${X}#index'
$ RAILS_ENV=test bin/rake db:drop db:create db:migrate && bin/rake test
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
$ bin/rails generate scaffold ${X} ${COLUMNS} owner:references:index
# x:string x:text 'x:decimal{10,2}' x:integer x:decimal x:float x:boolean x:binary x:date x:time x:datetime
# Example:
# bin/rails generate scaffold wisdom name:string wisdom:text owner:references:index
# You'll get the following warning and you should answer 'Y':
  conflict    app/assets/stylesheets/scaffolds.css.scss
  Overwrite /work/myplaceonline/src/src/myplaceonline_rails/app/assets/stylesheets/scaffolds.css.scss? (enter "h" for help) [Ynaqdh] Y
$ git checkout -- app/assets/stylesheets/scaffolds.css.scss
# Run migrate
$ bin/rake db:migrate
# Edit app/models/identity.rb
  has_many :${X}, :foreign_key => 'owner_id', :dependent => :destroy
      :${X} => ${X}.to_a.sort{ |a,b| a.name.downcase <=> b.name.downcase }.map{|x| x.as_json},
$ X=...
# Change after: cp app/controllers/wisdoms_controller.rb app/controllers/${X}_controller.rb
$ rm app/views/${X}/*jbuilder
# Create a myplaceonline.${X} section config/locales/en.yml based on myplaceonline.wisdom
# cp app/views/wisdoms/* app/views/${X} and replace all instances of wisdom with ${X}
# Edit config/routes.rb and add after resources ${X}
  post '${X}/new'
# Replace ${X} with singular version: cp app/models/wisdom.rb app/models/${X}.rb
# Edit app/models/ability.rb and add a line:
  can :manage, ${X}, :owner => identity
# Edit tests/fixtures/${X}.yml and create a fixture with a name of ${X} (see wisdoms.yml)
# cp test/controllers/wisdoms_controller_test.rb test/controllers/${X}_controller_test.rb
$ RAILS_ENV=test bin/rake db:drop db:create db:migrate && bin/rake test
```

#### Add encrypted column(s)

```
$ bin/rails generate migration AddEncryptionTo${MODEL} ${COLUMN}_encrypted:references:index
$ bin/rake db:migrate
# Add to model:
  include EncryptedConcern
  belongs_to :${COLUMN}_encrypted, class_name: EncryptedValue, dependent: :destroy, :autosave => true
  belongs_to_encrypted :${COLUMN}
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
    
    def create_presave
      @obj.${COLUMN}_finalize
    end
    
    def update_presave
      @obj.${COLUMN}_finalize
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
    cat = ${OLD}.where(name: "${OLD}").take!
    cat.name = "${NEW}"
    cat.link = "${NEW}"
    cat.save
# Change config/locales/en.yml
# Change db/seeds.rb
$ git mv app/views/${OLD}/ app/views/${NEW}/
$ git mv app/controllers/${OLD}_controller.rb app/controllers/${NEW}_controller.rb
$ git mv app/models/${OLD}.rb app/models/${NEW}.rb
$ git mv app/helpers/${OLD}_helper.rb app/helpers/${NEW}_helper.rb
$ git mv test/controllers/${OLD}_controller_test.rb test/controllers/${NEW}_controller_test.rb
$ git mv test/factories/${OLD}.rb test/factories/${NEW}.rb
$ git mv test/fixtures/${OLD}.yml test/fixtures/${NEW}.yml
$ git mv test/helpers/${OLD}_helper_test.rb test/helpers/${NEW}_helper_test.rb
$ git mv test/models/${OLD}_test.rb test/models/${NEW}_test.rb
$ git mv app/assets/javascripts/${OLD}.js.coffee app/assets/javascripts/${NEW}.js.coffee
$ git mv app/assets/stylesheets/${OLD}.css.scss app/assets/stylesheets/${NEW}.css.scss
$ bin/rake db:migrate
# Change lib/myp.rb
# Change config/routes.rb
# Change app/models/ability.rb
# Change app/models/identity.rb
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
# Equivalent to MySQL \G: \x before the command
# Backup: pg_dump -U postgres -h localhost -d myplaceonline_production > backup_`date +%Y%m%d%H%M%S`.sql
# Restore:
$ gpg --output tmp.sql --decrypt file.sql.pgp
$ pg_restore -U myplaceonline -h localhost -d myplaceonline_development -c -n public tmp.sql
$ rm tmp.sql
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

1. Go to https://build.phonegap.com/
2. Login with Adobe ID
3. Click Update Code > Pull Latest
