# myplaceonline

## Summary

myplaceonline is a virtual life assistant: [https://myplaceonline.com/](https://myplaceonline.com/)

## Source Code

The myplaceonline website is written in
[Ruby on Rails](http://rubyonrails.org/): https://github.com/myplaceonline/myplaceonline_rails.
The phone application
is written in [jQuery Mobile](http://jquerymobile.com/) and
[PhoneGap](http://phonegap.com/), and built using
[PhoneGap Build](https://build.phonegap.com/): https://github.com/myplaceonline/myplaceonline_phonegap. Latest Android test APK: https://build.phonegap.com/apps/1133885/download/android/

## License and Contributions

myplaceonline is licensed with the open source [AGPL (Affero GPL) v3 license](LICENSE). The AGPL basically means that you may do whatever you want with the code, but if you distribute the code or serve it through a website, share the code either by forking this repository or contributing to it. Pull requests are welcome. Guidelines for contributing: [CONTRIBUTE.md](CONTRIBUTE.md)

## Current Features

* Points system tracks life across three categories related to [happiness](#theory): order, joy, and meaning.
* Order
  * Manage online passwords (user name, password, URL, notes, recovery questions & answers, etc.)
    * Optionally encrypt passwords for storage in the database using [AES-256-CBC](http://en.wikipedia.org/wiki/Advanced_Encryption_Standard).
    * Import passwords from an [OpenOffice](https://www.openoffice.org/)/[LibreOffice](https://www.libreoffice.org/) [OpenDocument](http://en.wikipedia.org/wiki/OpenDocument) .ods spreadsheet (with support for encrypted spreadsheets) using [roo](https://github.com/roo-rb/roo).
    * Generate random passwords.
    * Use [ZeroClipboard](https://github.com/zeroclipboard/zeroclipboard) to quickly copy passwords to the clipboard.
    * Password notes support [Markdown](http://daringfireball.net/projects/markdown/syntax) syntax using [kramdown](https://github.com/gettalong/kramdown).
  * Manage To Dos.
* Joy
  * Manage movies you want to watch or have watched.
* Meaning
  * Manage notes of wisdom gathered in life.
* General Features
  * Export all data to a [JSON](https://en.wikipedia.org/wiki/JSON) file with the option of encrypting it in a standardized, [OpenPGP](https://tools.ietf.org/html/rfc4880) format (using AES-256/SHA512) which can be decrypted with [GnuPG](https://www.gnupg.org/) or [PGP](http://www.symantec.com/encryption/).
  * Save exported data in offline browser storage and display some of it even without internet using [Ember.js](http://emberjs.com/).
    * Supports decrypting AES-256-CBC passwords using [forge.js](https://github.com/digitalbazaar/forge).

## Screenshots

![Screenshot1](https://raw.githubusercontent.com/myplaceonline/myplaceonline_rails/master/app/assets/images/screenshot1.png)

## Planned Features

* Manage contacts, including tracking gift ideas and reminders for interactions.
* Manage notes, shopping lists, TODO lists, and life ideas.
* Track health, including vitals, diet changes, drugs, etc.
* Scan and manage receipts and documents.
* Manage website feeds and bookmarks.
* Synchronize files.
* Manage music.
* Find events.
* Manage travel.
* Manage finance and insurance.
* Manage exercise.
* Manage friendships and relationships.
* Manage hobbies.
* Manage addresses.
* Manage rebates.

## Running Locally

1. [Prepare your operating system](#prepos)
2. [Prepare your database](#prepdb)
3. [Download and prepare the source code](#prepsrc)
4. [Run the Rails server](#run)

### <a name="prepos"></a>Prepare Operating System

#### Fedora, CentOS, RHEL

```
$ sudo yum install git ruby ruby-devel rubygem-bundler zlib-devel patch nodejs
$ sudo yum groupinstall "C Development Tools and Libraries"
```

### <a name="prepdb"></a>Prepare Database (e.g. PostgreSQL)

#### Fedora, CentOS, RHEL

```
$ sudo yum install postgresql postgresql-server postgresql-devel postgresql-libs
$ sudo postgresql-setup initdb
$ sudo systemctl enable postgresql
$ sudo sed -ri 's/(host    all.*)ident/\1password/' /var/lib/pgsql/data/pg_hba.conf
$ sudo systemctl start postgresql
$ sudo gem install pg
$ sudo -u postgres psql postgres
postgres=# CREATE ROLE myplaceonline WITH LOGIN ENCRYPTED PASSWORD 'letmein' CREATEDB;
postgres=# \q
```

### <a name="prepsrc"></a>Download and prepare the source code

```
$ git clone --recursive https://github.com/myplaceonline/myplaceonline.git
$ cd myplaceonline/src/myplaceonline_rails/
$ cp config/database.yml.example config/database.yml
# Replace 'letmein' with your database user password:
$ sed -i 's/password: DBPASSWORD/password: letmein/g' config/database.yml
$ bin/bundle install
$ bin/rake db:setup
```

### <a name="run"></a>Run the Rails server

```
$ bin/rails server
```

Open [http://localhost:3000/](http://localhost:3000/)

### <a name="prepsrccommitter"></a>Prepare source if you'll be committing

```
$ cd ${MYPLACEONLINESRCDIR}
$ export NAME="Name"
$ export EMAIL="name@example.com"
$ git config --replace-all user.name "${NAME}"
$ git config --replace-all user.email "${EMAIL}"
$ git submodule foreach "git config --replace-all user.name \"${NAME}\""
$ git submodule foreach "git config --replace-all user.email \"${EMAIL}\""
$ git submodule foreach git checkout master
# Optional if using SSH Keys
$ git config --global url.ssh://git@github.com/.insteadOf https://github.com/
```

## <a name="theory"></a>Theory

> ‘Happiness’ is too worn and too weary a term to be of much scientific use, and the discipline of Positive Psychology divides it into three very different realms, each of which is measurable and, most importantly, each of which is skill-based and can be taught (Seligman, 2002). The first is hedonic: positive emotion (joy, love, contentment, pleasure etc.). A life led around having as much of this good stuff as possible, is the ‘Pleasant Life’. The second, much closer to what Thomas Jefferson and Aristotle sought, is the state of flow, and a life led around it is the ‘Engaged Life’. Flow, a major part of the Engaged Life, consists in a loss of self-consciousness, time stopping for you, being ‘one with the music’ (Csikszentmihalyi, 1990). Importantly engagement seems to be the opposite of positive emotion: when one is totally absorbed, no thoughts or feelings are present—even though one says afterwards ‘that was fun’ (Delle Fave & Massimini, 2005). And while there are shortcuts to positive emotion—you can take drugs, masturbate, watch television, or go shopping—there are no shortcuts to flow. Flow only occurs when you deploy your highest strengths and talents to meet the challenges that come your way, and it is clear that flow facilitates learning. The third realm in the framework of Positive Psychology is the one with the best intellectual provenance, the Meaningful Life. Flow and positive emotion can be found in solipsistic pursuits, but not meaning or purpose. Meaning is increased through our connections to others, future generations, or causes that transcend the self (Durkheim, 1951/1897; Erikson, 1963). From a Positive Psychology perspective, meaning consists in knowing what your highest strengths are, and then using them to belong to and serve something you believe is larger than the self (Seligman, 2002).
> 
> Positive education: positive psychology and classroom interventions, Seligman et al, Oxford Review of Education, 2009, http://www.ppc.sas.upenn.edu/positiveeducationarticle2009.pdf
