# Development

## Ruby

1.  Follow the [Ruby on Rails Coding Conventions](http://guides.rubyonrails.org/contributing_to_ruby_on_rails.html#follow-the-coding-conventions).

    > * Two spaces, no tabs (for indentation).
    > * No trailing whitespace. Blank lines should not have any spaces.
    > * Indent after private/protected.
    > * Use Ruby >= 1.9 syntax for hashes. Prefer { a: :b } over { :a => :b }.
    > * Prefer &&/|| over and/or.
    > * Prefer class << self over self.method for class methods.
    > * Use my_method(my_arg) not my_method( my_arg ) or my_method my_arg.
    > * Use a = b and not a=b.
    > * Use assert_not methods instead of refute.
    > * Prefer method { do_stuff } instead of method{do_stuff} for single-line blocks.
    > * Follow the conventions in the source you see used already.

1.  Prefer double quotes (") over single quotes (').

1.  Except for self-descriptive arguments (by function name or usage), pass
    keyword arguments to methods. This is more verbose and lengthens
    refactoring, but it makes function calls self-descriptive. Ideally, there
    would be a way to combine
    [arbitrary keyword arguments](https://github.com/ruby/ruby/blob/trunk/doc/syntax/methods.rdoc#keyword-arguments)
    with default values in the function definition, thus allowing transparent
    passing of hashes across functions and avoiding needing to go into the
    definition of the function to find the default; however, Ruby doesn't
    support this. Instead, follow a hybrid approach:
    1.  If a method does not expect to pass its options as a hash
        to another method, then define explicit keyword arguments (with
        required and optional arguments as needed).
        This should be the most common case. Example:
            
            def foo(required_param1:, optional_param1: "default")
              # ... definition ...
            end
            
            foo(required_param1: 1, optional_param1: "test")
            
    1.  If a method expects to pass its options as a hash to another
        method (or might do so in the future), then define an
        arbitrary keyword arguments options hash and set default values
        at the top of the function (if needed). If a parameter is required,
        throw a MissingArgumentError using the check_hash method. Example:
            
            def foo(**options)
              options = { optional_param1: "foo default" }.merge(options)
              bar(options)
            end
            
            def bar(**options)
              MissingArgumentError.check_hash(name: :required_param1, hash: options)
              options = { optional_param1: "bar default" }.merge(options)
              # ... definition ...
            end
            
            foo(required_param1: 1, optional_param1: "test")
            
1.  Try to keep lines less than 120 characters. Line splitting example:
        
        foo(
          required_param1: 1,
          required_param2: 2,
          required_param3: 3,
          optional_param1: "test"
        )

## General

* Site tests:
  * https://www.google.com/webmasters/tools/mobile-friendly/
* Passwords: https://www.schneier.com/blog/archives/2014/03/choosing_secure_1.html
* Performance/Security:
  * https://developers.google.com/web/tools/lighthouse/
  * https://developers.google.com/speed/pagespeed/insights/?url=https%3A%2F%2Fmyplaceonline.com%2F
  * https://securityheaders.io/?q=myplaceonline.com&followRedirects=on
  * https://www.ssllabs.com/ssltest/analyze.html?d=myplaceonline.com&hideResults=on
  * https://observatory.mozilla.org/analyze.html?host=myplaceonline.com
  * https://monkeytest.it/

"Note: Out-of-band authentication using the PSTN (SMS or voice) is discouraged and is being considered for removal in future editions of this guideline." (https://pages.nist.gov/800-63-3/sp800-63b.html)

https://github.com/berzerk0/Probable-Wordlists

## Architecture

                                               web4
                                            +--------+            +---------+
                                            + nginx  +            +         |
                                         XXXX  RoR   XXXXXXXXXXXXXX   db5   |
                       frontend1       XXX  +        +        XX  +         |
    +--------+       +-----------+   XXX    +--------+       XX   +----+----+
    |        |       |           + XXX                     XXX         | Streaming
    |  user  +------->  haproxy  XXXX                     XX           | Replication
    |        |       |           +  XX         web5      XX            |
    +--------+       +-----------+   XXX    +--------+  XX        +----v----+
                                       XXX  + nginx  +  X         |         |
                                         XXXX  RoR   XXXX         |   db6   |
                                            +        +            |         |
                                            +--------+            +---------+
    
    
                                                       sendgrid
                                                      +---------+
                                                      |         |
                                                      |  email  |
                                                      |         |
                                                      +---------+

* Made with ASCIIFlow Infinity: http://asciiflow.com/

### High Availability

* The database is backed up asynchronously using PostgreSQL streaming
  replication. It's a read-only standby that may be promoted in case of primary
  database failure.
* Some user-uploaded files are stored on the filesystem instead of in the
  database. These are stored on the primary database server and the web servers
  read/write over NFS. This NFS share is rsync'ed nightly to the backup
  database server.

### Get Logs

```
for i in frontend1; do scp root@${i}.myplaceonline.com:/var/log/haproxy.log ${i}_haproxy.log; done
for i in web24 web32 db5 db6; do scp root@${i}.myplaceonline.com:/var/log/messages ${i}_messages.log; done
for i in web24 web32; do scp root@${i}.myplaceonline.com:/var/www/html/myplaceonline/log/passenger.log ${i}_passenger.log; done
```

### Administration

* Central syslog server: db6.myplaceonline.com

        ssh root@db6.myplaceonline.com tail -50f /var/log/messages

* Rails logs

        multitail -l "ssh root@web4.myplaceonline.com tail -50f /var/log/messages | grep rails" -l "ssh root@web4.myplaceonline.com tail -50f /var/www/html/myplaceonline/log/passenger.log" -l "ssh root@web12.myplaceonline.com tail -50f /var/log/messages | grep rails" -l "ssh root@web12.myplaceonline.com tail -50f /var/www/html/myplaceonline/log/passenger.log"

* Passenger thread dumps

        kill -3 $PID
        less /var/www/html/myplaceonline/log/passenger.log

* HAProxy statistics (admin/cubevar_app_passwords_haproxy_stats)
  * posixcube.sh show | grep cubevar_app_passwords_haproxy_stats
  * https://myplaceonline.com:9443/
  * To put the backends into maintenance mode, check all web* servers and apply "Set state to MAINT"
  * To remove maintenance mode, check all web* servers and apply "Set state to READY"

* Grafana: https://grafana.myplaceonline.com/
  * Use admin and cubevar_app_passwords_grafana_admin

* Frontend HTTP requests:

        ssh root@frontend1.myplaceonline.com "date; tail -50f /var/log/haproxy.log" | grep -v -e STATS
        
        Log Format (http://cbonte.github.io/haproxy-dconv/1.7/configuration.html#8.2.4):
        
        log-format             %ci:%cp\ [%t]\ %ft\ %b/%s\ %Th/%Ti/%TR/%Tw/%Tc/%Tr/%Tt\ %ST\ %B\ %U\ %ac/%fc/%bc/%sc/%rc\ %sq/%bq\ %{+Q}r\ %hr %hs
        
                      first request               2nd request
            |<-------------------------------->|<-------------- ...
            t         tr                       t    tr ...
            |----|----|----|----|----|----|----|----|--
            : Th   Ti   TR   Tw   Tc   Tr   Td : Ti   ...
            :<---- Tq ---->:                   :
            :<-------------- Tt -------------->:
                      :<--------- Ta --------->:
        
        Times in milliseconds:
        
        - Th: total time to accept tcp connection and execute handshakes for low level
          protocols. Currently, these protocoles are proxy-protocol and SSL. This may
          only happen once during the whole connection's lifetime. A large time here
          may indicate that the client only pre-established the connection without
          speaking, that it is experiencing network issues preventing it from
          completing a handshake in a reasonable time (eg: MTU issues), or that an
          SSL handshake was very expensive to compute.

        - Ti: is the idle time before the HTTP request (HTTP mode only). This timer
          counts between the end of the handshakes and the first byte of the HTTP
          request. When dealing with a second request in keep-alive mode, it starts
          to count after the end of the transmission the previous response. Some
          browsers pre-establish connections to a server in order to reduce the
          latency of a future request, and keep them pending until they need it. This
          delay will be reported as the idle time. A value of -1 indicates that
          nothing was received on the connection.

        - TR: total time to get the client request (HTTP mode only). It's the time
          elapsed between the first bytes received and the moment the proxy received
          the empty line marking the end of the HTTP headers. The value "-1"
          indicates that the end of headers has never been seen. This happens when
          the client closes prematurely or times out. This time is usually very short
          since most requests fit in a single packet. A large time may indicate a
          request typed by hand during a test.

        - Tq: total time to get the client request from the accept date or since the
          emission of the last byte of the previous response (HTTP mode only). It's
          exactly equalt to Th + Ti + TR unless any of them is -1, in which case it
          returns -1 as well. This timer used to be very useful before the arrival of
          HTTP keep-alive and browsers' pre-connect feature. It's recommended to drop
          it in favor of TR nowadays, as the idle time adds a lot of noise to the
          reports.

        - Tw: total time spent in the queues waiting for a connection slot. It
          accounts for backend queue as well as the server queues, and depends on the
          queue size, and the time needed for the server to complete previous
          requests. The value "-1" means that the request was killed before reaching
          the queue, which is generally what happens with invalid or denied requests.

        - Tc: total time to establish the TCP connection to the server. It's the time
          elapsed between the moment the proxy sent the connection request, and the
          moment it was acknowledged by the server, or between the TCP SYN packet and
          the matching SYN/ACK packet in return. The value "-1" means that the
          connection never established.

        - Tr: server response time (HTTP mode only). It's the time elapsed between
          the moment the TCP connection was established to the server and the moment
          the server sent its complete response headers. It purely shows its request
          processing time, without the network overhead due to the data transmission.
          It is worth noting that when the client has data to send to the server, for
          instance during a POST request, the time already runs, and this can distort
          apparent response time. For this reason, it's generally wise not to trust
          too much this field for POST requests initiated from clients behind an
          untrusted network. A value of "-1" here means that the last the response
          header (empty line) was never seen, most likely because the server timeout
          stroke before the server managed to process the request.

        - Ta: total active time for the HTTP request, between the moment the proxy
          received the first byte of the request header and the emission of the last
          byte of the response body. The exception is when the "logasap" option is
          specified. In this case, it only equals (TR+Tw+Tc+Tr), and is prefixed with
          a '+' sign. From this field, we can deduce "Td", the data transmission time,
          by subtracting other timers when valid :

              Td = Ta - (TR + Tw + Tc + Tr)

          Timers with "-1" values have to be excluded from this equation. Note that
          "Ta" can never be negative.

        - Tt: total session duration time, between the moment the proxy accepted it
          and the moment both ends were closed. The exception is when the "logasap"
          option is specified. In this case, it only equals (Th+Ti+TR+Tw+Tc+Tr), and
          is prefixed with a '+' sign. From this field, we can deduce "Td", the data
          transmission time, by subtracting other timers when valid :

              Td = Tt - (Th + Ti + TR + Tw + Tc + Tr)

          Timers with "-1" values have to be excluded from this equation. In TCP
          mode, "Ti", "Tq" and "Tr" have to be excluded too. Note that "Tt" can never
          be negative and that for HTTP, Tt is simply equal to (Th+Ti+Ta).
        
        Example output:
        
        [...] servers/web4 324/34/0/0/1/16/376 200 7596 81 2/1/0/0/0 0/0 "GET / HTTP/1.1" {curl/7.53.1|} {ed038329-634f-4d6b-87ab-62047c653b2a}
        
* Rails:

        ssh root@db6.myplaceonline.com "tail -f /var/log/messages" | grep rails
        
        To check the response time of a request or tracked sub-executions, check for "response time in milliseconds":
        
        Sep 19 04:40:53 web12 rails[18477]: Started GET "/" for [...] at 2017-09-19 04:40:53 +0000
        Sep 19 04:40:53 web12 rails[18477]: Processing by WelcomeController#index as */*
        Sep 19 04:40:53 web12 rails[18477]: Completed 200 OK in 79ms (Views: 28.3ms | ActiveRecord: 2.4ms)
        Sep 19 04:40:53 web12 rails[18477]: MyplaceonlineRack.call response time in milliseconds = 85.72 context: {:uri=>"/", :request_id=>"c8757166-0f21-4ebe-8d7c-14e80f77c907", :user_id=>-1}
        
        PASSENGER_INSTANCE_REGISTRY_DIR=/var/run/ /usr/local/bin/passenger-status
        PASSENGER_INSTANCE_REGISTRY_DIR=/var/run/ /usr/local/bin/passenger-memory-stats

* Database:

        log_min_duration_statement controls which SQLs (with response time) are printed. Use 0 to print all statements;
        otherwise, a millisecond threshold.
        
        Messages go to /var/log/messages:
        
        Sep 19 04:55:27 db5 postgres[19587]: [2412-1] 2017-09-19 04:55:27 UTC myplaceonline@10.134.19.183(37608):myplaceonline_production [19587] LOG:  duration: 0.072 ms  execute a19: SELECT  "users".* FROM "users" WHERE "users"."id" = $1 LIMIT $2
        Sep 19 04:55:27 db5 postgres[19587]: [2412-2] 2017-09-19 04:55:27 UTC myplaceonline@10.134.19.183(37608):myplaceonline_production [19587] DETAIL:  parameters: $1 = '1', $2 = '1'

* ElasticSearch:

        curl http://db6-internal.myplaceonline.com:9200/
        curl http://db6-internal.myplaceonline.com:9200/_stats?pretty=1
        
        # Print field mapping:
        ssh root@db6.myplaceonline.com curl -s http://db6-internal.myplaceonline.com:9200/user/wisdom/_mapping | jq .

* Cleanup elasticsearch

        curl http://db6-internal.myplaceonline.com:9200/_aliases?pretty=1
        curl -XDELETE 'http://db6-internal.myplaceonline.com:9200/logstash-2016.08*/'

* Take a web server down for maintenance:

        systemctl stop nginx

* Put web server back into rotation:

        systemctl stop nginx

* Rails
  * Most things rsyslog'd to db6 (/var/log/messages); however, Rails app logging doesn't support syslog, so it goes to /var/log/messages

          grep rails /var/log/messages
          cat /var/www/html/myplaceonline/log/passenger.log

  * multitail -l "ssh root@web4.myplaceonline.com tail -f /var/www/html/myplaceonline/log/passenger.log" -l "ssh root@web12.myplaceonline.com tail -f /var/www/html/myplaceonline/log/passenger.log"

Common issues:
* grep "rails.*processed.*[^0] failed" /var/log/messages
* grep "rails.*Performing" /var/log/messages
* journalctl -p warning

* Linux
  * `atop -r` and use `t` and `T` to move forward/backward, and `b` to jump to time
    * ls -l /var/log/atop/atop_*
* Crashes in /var/crash/
* Sometimes stuff in
  * /var/www/html/myplaceonline/log/passenger.log
* ElasticSearch
  * curl http://db2-internal.myplaceonline.com:9200/_cluster/stats?pretty
  * /var/log/elasticsearch/

* Journal disk usage: journalctl --disk-usage
* Restart journald: sudo systemctl restart systemd-journald.service
* Clear: journalctl --vacuum-size=1M

* PostgreSQL archive files in 
** pg_archivecleanup /var/lib/pgsql/data/pg_xlog/ ${MOST_RECENT_FILE_IN_pg_xlog}

* InfluxDB/Telegraf

* influx -host db6-internal.myplaceonline.com -database telegraf
  * show diagnostics
  * show stats
  * show measurements
  * show series
    * drop series from "system" where "host" = 'web11'
  * drop old data: DELETE WHERE time < '2019-01-01'

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

## Learn

* https://github.com/ssllabs/research/wiki/SSL-and-TLS-Deployment-Best-Practices

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

TODO: [Password requirements](https://github.com/usnistgov/800-63-3/blob/nist-pages/sp800-63b/sec5_authenticators.md#5112-memorized-secret-verifiers)

## Rails

1.  The basic flow of views is app/views/${CATEGORY}/_form.html.erb includes
    app/views/shared/_model_form.html.erb passing in obj: @obj as a local.
    Normally the view would just reference the @obj member variable of the 
    controller directly, but we use the obj:@obj model so that one form
    can include another form (usually a belongs_to relationship).
2.  Use semantic names. For example, if you have the date of a weight
    measurement, use a field named measurement_start instead of just start. The
    reason for this is that, by default, we use form field names matching the
    model field name, so the autocomplete will be more specific if the field
    name is more specific.
3.  belongs_to: In the class that has the foreign key.
    has_one: If the other class has the foreign key.
4.  MyplaceonlineController supports an "insecure" mode where items can be
    added without needing to re-enter a password (just a remember me cookie is
    needed). Add in the protected section.
        def insecure
          true
        end

5.  Order has_many example:
    has_many :job_salaries, -> { order('started DESC') }, :dependent => :destroy
6.  Is not null example:
    IdentityDriversLicense.where("identity_id = ? and expires is not null and expires < ?", user.primary_identity, threshold)
7.  Enumeration:
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
8.  List of Files/Pictures
    $ BUNDLE_GEMFILE=Gemfile_engines bin/rails generate model test_object_file test_object:references:index identity_file:references:index identity:references:index position:integer is_public:boolean
    $ BUNDLE_GEMFILE=Gemfile_engines bin/rails db:migrate
    $ cp app/models/test_object_file.rb app/models/${X}
      And then update the parent

    controller:

      # public

      def may_upload
        true
      end

      quest_files_attributes: FilesController.multi_param_names
    
    model:
      child_files
    _form
      <%=
        render partial: "myplaceonline/pictures_form", locals: {
          f: f,
          obj: obj,
          position_field: :position,
          pictures_field: :test_object_files,
          item_placeholder: "myplaceonline.identity_files.file",
          heading: "myplaceonline.identity_files.files",
          addbutton: "myplaceonline.identity_files.add_file",
          addbutton_multi: "myplaceonline.identity_files.add_files",
          deletebutton: "myplaceonline.identity_files.delete_file"
        }
      %>
    show
      <%= data_row(heading: t("myplaceonline.identity_files.file"), content: obj.test_object_files) %>
9.  Add model initialization code
        def self.build(params = nil)
          result = self.dobuild(params)
          # initialize result here
          result
        end
10. Add category filter text
        $ BUNDLE_GEMFILE=Gemfile_engines bin/rails generate migration AddCategoryFiltertextTODO
        Myp.migration_add_filtertext("$CATEGORY", "$SPACE_DELIMITED_ADDITIONS")
        $ BUNDLE_GEMFILE=Gemfile_engines bin/rails db:migrate
11. Add column

        $ BUNDLE_GEMFILE=Gemfile_engines bin/rails generate migration AddColumnsTODOToModel (Model Plural) newcol:text
        
        Example:
        $ BUNDLE_GEMFILE=Gemfile_engines bin/rails generate migration AddColumnsFavoriteFoodsToIdentities favorite_foods:text
        
        $ BUNDLE_GEMFILE=Gemfile_engines bin/rails db:migrate

12. Transaction

        # See http://api.rubyonrails.org/classes/ActiveRecord/Transactions/ClassMethods.html
        
        ActiveRecord::Base.transaction do
        end
        
        or
        
        ActiveRecord::Base.transaction(requires_new: true) do
        end

13. Logging
        Rails.logger.debug{"test"}
        Rails.logger.info{"test"}
        Rails.logger.warn{"test"}
        Rails.logger.error{"test"}
14. Rebuild index
        $ bin/rails generate migration RebuildIndex005
        UserIndex.reset!(timeout: "3600000ms")
15. Error: ArgumentError: Index name '...' on table '...' is too long; the limit is 63 characters
    Modify migration to set index: false, and:
    add_index :table_name, :column_id, name: "table_shortname_on_column_shortname"

### Jobs

Run with delay:

BUNDLE_GEMFILE=Gemfile_engines PERMDIR=/var/lib/remotenfs/ FILES_PREFIX=/work/myplaceonline/backups/ bin/rails jobs:work

Run once:

BUNDLE_GEMFILE=Gemfile_engines PERMDIR=/var/lib/remotenfs/ FILES_PREFIX=/work/myplaceonline/backups/ bin/rails jobs:workoff

(Also called delayed_jobs)

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
$ bin/rails generate migration AddPointsToIdentities points:integer
$ RAILS_ENV=test bin/rake db:reset test
```

Update gems:

```
$ BUNDLE_PATH=vendor/bundle/ bin/bundle update
# Gemfile_engines.lock must be symlinked, e.g. ln -s engines_config/drom-production/Gemfile_engines.lock
$ BUNDLE_PATH=vendor/bundle/ BUNDLE_GEMFILE=Gemfile_engines bin/bundle update
$ # Test app
$ git commit; git push
$ cd engines_config/*/
$ git commit; git push
```

Running:

```
alias ms='sudo systemctl start postgresql; sudo systemctl start elasticsearch; cd ...; BUNDLE_PATH=vendor/bundle/ PERMDIR=... FILES_PREFIX=... GOOGLE_MAPS_API_SERVER_KEY=... GOOGLE_MAPS_API_KEY=... SKIP_NOTIFICATIONS=true GOOGLE_PLACES_KEY=... BUNDLE_GEMFILE=Gemfile_engines MINCACHE=true bin/rails server -b 127.0.0.1'
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
$ RAILS_ENV=test bin/rake db:drop db:create db:schema:load db:seed myp:reinitialize test
# Start rails server
```

#### Create New Leaf Category Example

```
# ${X} is usually plural here in capital camel case:
# Create a migration:
$ BUNDLE_GEMFILE=Gemfile_engines bin/rails generate migration AddCategory${X}
# Edit the new migration (${X} is all lowercase here and usually plural and underscores):
  def change
    Category.create(name: "${X}", link: "${X}", position: 0, parent: Category.find_by_name("${Y}"), icon: "FatCow_Icons16x16/check_box_uncheck.png")
  end
# Add to config/locales/en.yml (first ${X} is lowercase, second one is usually capitalized):
  myplaceonline:
    category:
      ${X}: "${X}"
# ${X} is non-plural, lower-case and underscores instead of camel case:
$ BUNDLE_GEMFILE=Gemfile_engines bin/rails generate scaffold ${X} ${COLUMNS} notes:text visit_count:integer archived:datetime rating:integer is_public:boolean identity:references:index
# x:string x:text 'x:decimal{10,2}' x:integer x:decimal x:float x:boolean x:binary x:date x:time x:datetime
$ BUNDLE_GEMFILE=Gemfile_engines bin/rails db:migrate
# Edit app/models/identity.rb
  has_many :${X}, :dependent => :destroy
      :${X} => ${X}.to_a.sort{ |a,b| a.name.downcase <=> b.name.downcase }.map{|x| x.as_json},
$ X=...
# Change after: cp app/controllers/test_objects_controller.rb app/controllers/${X}_controller.rb
# Create a myplaceonline.${X} section config/locales/en.yml based on myplaceonline.wisdom
# rm app/views/${X}/*
# cp app/views/test_objects/* app/views/${X} and replace all instances of wisdom with ${X}
# Edit config/routes.rb and remove the resources line that was auto-generated
# Replace ${X} with singular version: cp app/models/test_object.rb app/models/${X}.rb
# Edit tests/fixtures/${X}.yml and create a fixture with a name of ${X} (see test_objects.yml)
# cp test/controllers/test_objects_controller_test.rb test/controllers/${X}_controller_test.rb
$ RAILS_ENV=development BUNDLE_GEMFILE=Gemfile_engines bin/rake myp:dump
# If changing the following, update .travis.yml
$ RAILS_ENV=test SKIP_LARGE_UNNEEDED_IMPORTS=true SKIP_ZIP_CODE_IMPORTS=true BUNDLE_GEMFILE=Gemfile_engines bin/rake db:drop db:test:prepare test
# To run a particular test, add to the end: TEST=test/controllers/[...]
```

#### Add encrypted column(s)

```
$ BUNDLE_GEMFILE=Gemfile_engines bin/rails generate migration AddEncryptionTo${MODEL} ${COLUMN}_encrypted:references:index
    add_reference :ssh_keys, :ssh_private_key_encrypted, index: true, foreign_key: false
    add_foreign_key :ssh_keys, :encrypted_values, column: :ssh_private_key_encrypted_id
$ BUNDLE_GEMFILE=Gemfile_engines bin/rails db:migrate
# Add to model:
  include EncryptedConcern
  belongs_to :${COLUMN}_encrypted, class_name: "EncryptedValue", dependent: :destroy, :autosave => true
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

#### Add Nested Model

```
$ cp app/controllers/vehicle_services_controller.rb app/controllers/${MODEL}_controller.rb
# Add category to en.yml
$ cp -r app/views/vehicle_services/ app/views/${MODEL}/
# Add to parent controller
  def footer_items_show
    super + [
      {
        title: I18n.t("myplaceonline.vehicles.add_vehicle_service"),
        link: new_vehicle_vehicle_service_path(@obj),
        icon: "plus"
      },
      {
        title: I18n.t("myplaceonline.vehicles.vehicle_services"),
        link: vehicle_vehicle_services_path(@obj),
        icon: "bars"
      },
    ]
  end
# Add to routes
    vehicles: [
      {
        subresources: true,
        name: :vehicle_services
      }
    ],
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
$ dropdb -U myplaceonline -h localhost  myplaceonline_development; createdb -U myplaceonline -h localhost myplaceonline_development; pg_restore -U myplaceonline -h localhost -d myplaceonline_development -n public *.sql
$ BUNDLE_GEMFILE=Gemfile_engines MINCACHE=true bin/rails db:migrate:status 2>&1 | grep down | while read line; do pending="$(echo "${line}" | awk '{print $2}')"; echo "INSERT INTO schema_migrations (version) values ('${pending}');"; done | psql -U myplaceonline -h localhost -d myplaceonline_development
$ sudo reindexdb -U myplaceonline -h localhost -d myplaceonline_development
$ BUNDLE_GEMFILE=Gemfile_engines MINCACHE=true bin/rails c
# UserIndex.reset!(timeout: "3600000ms")
```

* Log in as root: sudo -u postgres psql postgres
* List advisory locks: SELECT * FROM pg_locks where locktype = 'advisory';

### Full Backup

PGPASSWORD=password pg_dumpall -U myplaceonline -h localhost --clean --if-exists > dumpall.sql

### Full Restore

PGPASSWORD=password psql -U myplaceonline -h localhost -f dumpall.sql --single-transaction postgres

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

* https://guides.rubyonrails.org/testing.html
* Test data in test/fixtures/*.yml
* Devise user logged in for every test in test/test_helper.rb setup

## Running Tests

```
RAILS_ENV=test SKIP_LARGE_UNNEEDED_IMPORTS=true SKIP_ZIP_CODE_IMPORTS=true BUNDLE_GEMFILE=Gemfile_engines time bin/rails test --fail-fast --verbose
```

To run particular tests, add files or directories to the end, e.g.:

```
RAILS_ENV=test SKIP_LARGE_UNNEEDED_IMPORTS=true SKIP_ZIP_CODE_IMPORTS=true BUNDLE_GEMFILE=Gemfile_engines time bin/rails test --fail-fast --verbose test/controllers/test_objects_controller_test.rb
```

# Server Administration

## Reboot

### Web Server

NODE=web1; ssh root@${NODE}.myplaceonline.com "systemctl stop nginx; reboot"

## Analyze Crash

http://averageradical.github.io/Linux_Core_Dumps.pdf

    $ /usr/local/src/crash*/crash /usr/lib/debug/lib/modules/4*/vmlinux /var/crash/*/vmcore
    # ps
    # http://averageradical.github.io/Linux_Core_Dumps.pdf
    
* strings /var/crash/*/vmcore | grep "Linux version"
* dnf list installed | grep kernel-debuginfo
* Emulate a crash: echo c > /proc/sysrq-trigger

## Kibana

* http://localhost:5601/
* Search: _id:1004 AND _type:museum

## rsyslog

* tcpdump -Xni eth1 port 514

# Add domain

* DNS
  * Set A records to main floating IP
  * Create MX record @ with db5.myplaceonline.com
  * Add sender domain to Postmark
  * No SPF for Postmark needed: https://postmarkapp.com/support/article/1092-how-do-i-set-up-spf-for-postmark
* Email hosting:
  * Add to cubevar_app_email_domains and cubevar_app_email_dkim_domains
  * Follow instructions in email_server.sh to add domain & users
* Create WebsiteDomain with all of the hosting details filled out and update verified = true on it
* Update cubevar_app_letsencrypt_tls_domains and cubevar_app_tls_domains in envars_production.sh and run posixcube.sh -z frontend
* Set homepage to public if it's a particular object (e.g. /model/N)
* Log into frontend and run the commands in /etc/cron.d/letsencrypt
* Update local/maintenance*.sh

# PhoneGap

## Building New Version

1. Bump version and versionCode in config.xml
1. Android:
    1. Set widget id="com.myplaceonline" in config.xml
    1. Set suppressPushState: false in index.js
1. iOS
    1. Set widget id="com.myplaceonline.main" in config.xml
    1. Set suppressPushState: true in index.js
1. Commit any changes in src/myplaceonline_phonegap
1. Go to https://build.phonegap.com/
1. Login with Adobe ID
1. Click Update Code > Pull Latest
1. Android:
    1. Set Key=Android
    1. Download APK to lib/android/builds
    1. Test with adb install -r myplaceonline*.apk
    1. Update APK at https://play.google.com/apps/publish/
    1. Click on the app
    1. Click Release Management
    1. Click App releases
    1. Click Production: Manage
    1. Click Create Release
1. iOS:
    1. Set Key=Production
    1. Download IPA to lib/ios/builds
    1. Test with XCode

## Debugging Using Chrome Developer ToolsController

1. Go to https://build.phonegap.com/apps/1133885/settings
1. Check "Enable debugging"
1. Go to https://build.phonegap.com/apps/1133885/builds
1. Click Update code > Pull latest
1. Select "No key selected" to build the APK
1. Uninstall the app on the phone
1. Install from https://build.phonegap.com/apps/1133885/share
1. Open the app
1. Open Chrome to chrome://inspect/
1. Click inspect on the WebView

## iOS Certificate

1.  Connect iPhone to Mac
2.  Mac: XCode > Windows > Devices
    1.  Select "Identifier" and Copy
    2.  Ctrl+Click the UDID and click Copy
3.  Log in to [https://developer.apple.com/account/](https://developer.apple.com/account/)
    1.  Click "Certificates, Identifiers & Profiles"
4.  Create development and distribution certificates
    1.  Follow the instructions, download the .cer file and double click it to install into Keychain Access into the "login" keychain
    2.  Although these show up on the main screen, do not export from here. On the left, click My Certificates > Expand iPhone Developer > Right click on the private key and export
    3.  Export the certificate (cer)
    4.  Repeat for iOS distribution certificate (uses the same p12/pem keys)
5.  Add device UDID to provisioning profile
    1.  Click Devices > All
    2.  Click Plus icon in the top right and follow the steps
    3.  Provisioning Profiles > All
    4.  Click the Plus icon
    5.  Create one for iOS App Development
    6.  Select wildcard App ID
    7.  Download .mobileprovision file
6.  Phonegap Build
    1.  Edit account > signing keys
    2.  Add p12 and .mobileprovision file and use the password created when exporting the p12 file
    3.  Build iOS app and download the IPA file
7.  Mac: XCode > Window > Devices > Select Device > Click Plus under "Installed Apps" and select .ipa
8.  iPhone: Settings > Safari > Advanced > Web Inspector = On
9.  Mac: Safari > Preferences... > Advanced > Check "Show Develop menu in menu bar"
10.  iPhone: Launch app
11.  Mac: Safari > Develop > iPhone Name > Select app

# OAuth/OpenID Connect

Test:

```
client = OAuth2::Client.new('Application UID', 'Secret', site: 'http://localhost:3000/')
access_token = client.password.get_token('user@example.com', 'password')
Myp.http_get(url: "http://localhost:3000/test_api/hello_world", try_https: false, bearer_token: access_token.token)
```

```
RestClient::Request.execute(method: :post, url: "http://localhost:3000/api/login_or_register", headers: {"Content-Type" => "application/json"}, payload: { email: "testapi@myplaceonline.com", password: "letmein" }.to_json).body

RestClient::Request.execute(method: :post, url: "http://localhost:3000/api/refresh_token", headers: {"Content-Type" => "application/json"}, payload: { refresh_token: access_token.refresh_token }.to_json).body
```

Create client:
```
Doorkeeper::Application.create!(
  name: "Internal",
  uid: SecureRandom.hex(22),
  secret: SecureRandom.hex(22),
  redirect_uri: "",
  confidential: true,
)
```

Applications at /oauth/applications

## Gemfiles

```
sudo BUNDLE_GEMFILE=Gemfile_engines bin/bundle install
BUNDLE_GEMFILE=Gemfile_engines bin/rails server
BUNDLE_GEMFILE=Gemfile_engines bin/rails console
```

## Engines

```
BUNDLE_GEMFILE=Gemfile_engines bin/rails engine:install:migrations
BUNDLE_GEMFILE=Gemfile_engines bin/rails db:migrate
```
