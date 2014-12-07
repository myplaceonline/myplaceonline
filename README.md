# myplaceonline

## Summary

myplaceonline is a virtual life coach: [https://myplaceonline.com/](https://myplaceonline.com/)

## License

myplaceonline is licensed with the [AGPL (Affero GPL) v3 license](LICENSE). Please share your contributions. Guidelines for contributing: [CONTRIBUTE.md]

## Screenshots

![Screenshot1](https://raw.githubusercontent.com/myplaceonline/myplaceonline_rails/master/app/assets/images/screenshot1.png)

## Proposed Features

* Points system helps identify strong and weak character strengths.
* Manage online passwords and export as PDF.
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

## Design Goals

* Mobile/tablet first design. Single Page Application (SPA): http://docs.phonegap.com/en/3.5.0/guide_next_index.md.html
* Choose what information is stored offline in local storage and synchronize when internet available.

## Running Locally

### First Time

```
# Install database (see below)
# Get the source code
$ git clone --recursive git@github.com:myplaceonline/myplaceonline.git
$ cd myplaceonline
$ export NAME="Name"
$ export EMAIL="name@example.com"
$ git config --replace-all user.name "${NAME}"
$ git config --replace-all user.email "${EMAIL}"
$ git submodule foreach "git config --replace-all user.name \"${NAME}\""
$ git submodule foreach "git config --replace-all user.email \"${EMAIL}\""
$ git submodule foreach git checkout master
$ cd src/myplaceonline_rails/
$ cp config/database.yml.example config/database.yml
# Put database credentials into config/database.yml. For example:
$ sed -i 's/#username: myplaceonline/username: user1/g' config/database.yml
$ sed -i 's/#password:/password: letmein/g' config/database.yml
$ sed -i 's/#host: localhost/host: localhost/g' config/database.yml
# Initialize the database
$ bin/rake db:setup
```

### Running the site

```
$ bin/rails server
```

Open [http://localhost:3000/](http://localhost:3000/)

### Install Database (e.g. PostgreSQL)

#### Fedora, CentOS, RHEL

```
$ sudo yum install postgresql postgresql-server postgresql-devel postgresql-libs
$ sudo postgresql-setup initdb
$ sudo systemctl enable postgresql
$ sudo sed -ri 's/(host    all.*)ident/\1password/' /var/lib/pgsql/data/pg_hba.conf
$ sudo systemctl start postgresql
$ sudo gem install pg
$ sudo -u postgres psql postgres
# CREATE ROLE user1 WITH LOGIN ENCRYPTED PASSWORD 'letmein' CREATEDB;
```

## TODO

* Internationalize devise views
* Single re-login per session to modify settings
* Add dependent destroy on user or identity

## Source Code Guidelines

* Use input placeholder and a matching label with class ui-hidden-accessible: http://view.jquerymobile.com/master/demos/forms-label-hidden-accessible/

## Theory

‘Happiness’ is too worn and too weary a term to be of much scientific use, and the discipline of Positive Psychology divides it into three very different realms, each of which is measurable and, most importantly, each of which is skill-based and can be taught (Seligman, 2002). The first is hedonic: positive emotion (joy, love, contentment, pleasure etc.). A life led around having as much of this good stuff as possible, is the ‘Pleasant Life’. The second, much closer to what Thomas Jefferson and Aristotle sought, is the state of flow, and a life led around it is the ‘Engaged Life’. Flow, a major part of the Engaged Life, consists in a loss of self-consciousness, time stopping for you, being ‘one with the music’ (Csikszentmihalyi, 1990). Importantly engagement seems to be the opposite of positive emotion: when one is totally absorbed, no thoughts or feelings are present—even though one says afterwards ‘that was fun’ (Delle Fave & Massimini, 2005). And while there are shortcuts to positive emotion—you can take drugs, masturbate, watch television, or go shopping—there are no shortcuts to flow. Flow only occurs when you deploy your highest strengths and talents to meet the challenges that come your way, and it is clear that flow facilitates learning. The third realm in the framework of Positive Psychology is the one with the best intellectual provenance, the Meaningful Life. Flow and positive emotion can be found in solipsistic pursuits, but not meaning or purpose. Meaning is increased through our connections to others, future generations, or causes that transcend the self (Durkheim, 1951/1897; Erikson, 1963). From a Positive Psychology perspective, meaning consists in knowing what your highest strengths are, and then using them to belong to and serve something you believe is larger than the self (Seligman, 2002).

Positive education: positive psychology and classroom interventions, Seligmana et al, Oxford Review of Education, 2009, http://www.ppc.sas.upenn.edu/positiveeducationarticle2009.pdf