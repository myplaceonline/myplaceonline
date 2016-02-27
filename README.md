# myplaceonline

## Summary

myplaceonline is a virtual life assistant: [https://myplaceonline.com/](https://myplaceonline.com/)

## Source Code

The myplaceonline website is written in Ruby on Rails: https://github.com/myplaceonline/myplaceonline_rails.
The phone application is written in jQuery Mobile and PhoneGap, and built using PhoneGap Build: https://github.com/myplaceonline/myplaceonline_phonegap.

Latest Android App in the Google Play Store: https://play.google.com/store/apps/details?id=com.myplaceonline

## License and Contributions

myplaceonline is licensed with the open source [AGPL (Affero GPL) v3 license](LICENSE). The AGPL basically means that you may do whatever you want with the code, but if you distribute the code or serve it through a website, share the code either by forking this repository or contributing to it. Pull requests are welcome. Guidelines for contributing: [CONTRIBUTE.md](CONTRIBUTE.md)

## Screenshots

![Screenshot1](https://raw.githubusercontent.com/myplaceonline/myplaceonline_rails/master/app/assets/images/screenshot2.png)
![Screenshot6](https://raw.githubusercontent.com/myplaceonline/myplaceonline_rails/master/app/assets/images/screenshot6.png)
![Screenshot7](https://raw.githubusercontent.com/myplaceonline/myplaceonline_rails/master/app/assets/images/screenshot7.png)
![Screenshot8](https://raw.githubusercontent.com/myplaceonline/myplaceonline_rails/master/app/assets/images/screenshot8.png)

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
# http://stackoverflow.com/a/28515064/5657303
postgres=# ALTER ROLE myplaceonline WITH SUPERUSER;
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
$ bin/rake db:create db:migrate
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
> Positive education: positive psychology and classroom interventions, Seligman et al, Oxford Review of Education, 2009, http://www.sas.upenn.edu/~duckwort/images/upperdarbypd/10082012_PDReading.pdf
