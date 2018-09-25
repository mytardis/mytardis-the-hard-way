# MyTardis 3.x on Ubuntu 16.04 LTS
This guide assumes that you already have an Ubuntu 16.04 LTS server or virtual machine (VM) running. It's also assumes that you have some familiarity setting up a local development enviroment for MyTardis following [this (somewhat outdated) guide](https://mytardis.readthedocs.io/en/develop/admin/install.html).

## Getting started
We need to install some dependencies. From [mytardis.readthedocs.io](https://mytardis.readthedocs.io/en/develop/admin/install.html):
```
ubuntu@mytardis-manual:~$ sudo apt-get update
ubuntu@mytardis-manual:~$ sudo apt-get install git ipython libldap2-dev libsasl2-dev \
  libssl-dev libxml2-dev libxslt1-dev nodejs npm python-anyjson \
  python-billiard python-bs4 python-crypto python-dateutil \
  python-dev python-flexmock python-html5lib \
  python-pip python-psycopg2 python-pystache \
  python-virtualenv python-wand python-yaml virtualenvwrapper \
  zlib1g-dev libfreetype6-dev libjpeg-dev
```

## Installing MyTardis
Let’s create a group and user for our web app:
```
ubuntu@mytardis-manual:~$ sudo groupadd -r mytardis
ubuntu@mytardis-manual:~$ sudo useradd -m -r -g mytardis -s /bin/bash mytardis
```

Let’s switch to the new user:
```
ubuntu@mytardis-manual:~$ sudo -i -u mytardis
```

Clone mytardis:
```
mytardis@mytardis-manual:~$ git clone https://github.com/mytardis/mytardis.git
Cloning into 'mytardis'...
remote: Counting objects: 29586, done.
remote: Compressing objects: 100% (11/11), done.
remote: Total 29586 (delta 2), reused 7 (delta 2), pack-reused 29573
Receiving objects: 100% (29586/29586), 28.45 MiB | 10.74 MiB/s, done.
Resolving deltas: 100% (21913/21913), done.
Checking connectivity... done.
mytardis@mytardis-manual:~$ cd mytardis/
```

Create and activate virtualenv:
```
mytardis@mytardis-manual:~/mytardis$ source /usr/share/virtualenvwrapper/virtualenvwrapper.sh
mytardis@mytardis-manual:~/mytardis$ mkvirtualenv --system-site-packages mytardis
Running virtualenv with interpreter /usr/bin/python2
New python executable in /home/mytardis/.virtualenvs/mytardis/bin/python2
Also creating executable in /home/mytardis/.virtualenvs/mytardis/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.
(mytardis) mytardis@mytardis-manual:~/mytardis$ pip install -U pip
Requirement already up-to-date: pip in /home/mytardis/.virtualenvs/mytardis/lib/python2.7/site-packages (10.0.1)
```

Install pip dependencies:
```
(mytardis) mytardis@mytardis-manual:~/mytardis$ pip install -U -r requirements.txt
```

Install npm dependencies:
```
(mytardis) mytardis@mytardis-manual:~/mytardis$ npm install
```

Make a directory to store out data:
```
(mytardis) mytardis@mytardis-manual:~/mytardis$ mkdir -p var/store
```
Note: In production, we would most likely mount an NFS volume to host the data.

Let’s create a secret key:
```
(mytardis) mytardis@mytardis-manual:~/mytardis$ python -c "import os; from random import choice; key_line = '%sSECRET_KEY=\"%s\"  # generated from build.sh\n' % ('from tardis.settings_changeme import * \n\n' if not os.path.isfile('tardis/settings.py') else '', ''.join([choice('abcdefghijklmnopqrstuvwxyz0123456789\\!@#$%^&*(-_=+)') for i in range(50)])); f=open('tardis/settings.py', 'a+'); f.write(key_line); f.close()"
```
and run the mytardis unittests to see if every seems okay with the environment:
```
(mytardis) mytardis@mytardis-manual:~/mytardis$ python test.py
nosetests --verbosity=1
Creating test database for alias 'default'...

..................................................................................................................S.S.S.S.S.S.S.....................S..............................S.S.S.S.S...S..................
----------------------------------------------------------------------
Ran 196 tests in 49.608s

OK (SKIP=14)
Destroying test database for alias 'default'...
```

Okay for now we are just going to use the `sqlite` database for testing. In fact, this is already configured in MyTardis’s  default settings [here](https://github.com/mytardis/mytardis/blob/develop/tardis/default_settings/database.py). Note we’ll need to do this again when we switch to a production database.

Let’s run the migration:
```
(mytardis) mytardis@mytardis-manual:~/mytardis$ python mytardis.py migrate
Operations to perform:
  Synchronize unmigrated apps: mustachejs, search, publication_forms, staticfiles, admindocs, templatetags, bootstrapform, analytics, django_extensions, oaipmh, tastypie_swagger, humanize
  Apply all migrations: sessions, admin, tastypie, djcelery, sites, kombu_transport_django, tardis_portal, contenttypes, auth, registration
Synchronizing apps without migrations:
  Creating tables...
    Running deferred SQL...
  Installing custom SQL...
Running migrations:
  Rendering model states... DONE
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying djcelery.0001_initial... OK
  Applying kombu_transport_django.0001_initial... OK
  Applying registration.0001_initial... OK
  Applying registration.0002_registrationprofile_activated... OK
  Applying registration.0003_migrate_activatedstatus... OK
  Applying sessions.0001_initial... OK
  Applying sites.0001_initial... OK
  Applying tardis_portal.0001_squashed_0011_auto_20160505_1643...
 OK
  Applying tastypie.0001_initial... OK
```
Let's setup the cache and celery_lock tables:
```
(mytardis) mytardis@mytardis-manual:~/mytardis$ python mytardis.py createcachetable default_cache
(mytardis) mytardis@mytardis-manual:~/mytardis$ python mytardis.py createcachetable celery_lock_cache
```

Finally, we collect our static assets:
```
(mytardis) mytardis@mytardis-manual:~/mytardis$ python mytardis.py collectstatic
```

While we’re here, let’s also create a superuser for our admin:
```
(mytardis) mytardis@mytardis-manual:~/mytardis$ python mytardis.py createsuperuser
```

Let’s check whether we can run the server:
```
(mytardis) mytardis@mytardis-manual:~/mytardis$ ./mytardis.py runserver
```
And from another session:
```
ubuntu@mytardis-manual:~$ wget http://127.0.0.1:8000
--2018-05-15 20:49:30--  http://127.0.0.1:8000/
Connecting to 127.0.0.1:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: unspecified [text/html]
Saving to: 'index.html’

index.html              [ <=>                ]  10.16K  --.-KB/s    in 0s

2018-05-15 20:49:31 (177 MB/s) - 'index.html’ saved [10402]
```
Okay looking good.

At this point we could have a play with MyTardis on a local machine, but there are limitations in SQLite (i.e., concurrent DB access) that prevent it from being able to power a full MyTardis. We’ll come back to this later.

## Switching to Postgres DB
To run in production, we really need to switch databases.  Let’s install postgres:
```
ubuntu@mytardis-manual:~$ sudo apt-get install postgresql postgresql-contrib
```
Note: I have switched back to the `ubuntu` user.

Let’s create a database and credentials for `mytardis`
```
ubuntu@mytardis-manual:~$ sudo -u postgres psql
postgres=# CREATE DATABASE mytardis_db;
postgres=# CREATE USER mytardis WITH PASSWORD 'makeitstronger';
postgres=# ALTER ROLE mytardis SET client_encoding TO 'utf8';
postgres=# ALTER ROLE mytardis SET default_transaction_isolation TO 'read committed';
postgres=# ALTER ROLE mytardis SET timezone TO 'UTC';
postgres=# GRANT ALL PRIVILEGES ON DATABASE mytardis_db TO mytardis;
postgres=# \q
```
Note: I basically followed the guide for [How To Use PostgreSQL with your Django Application on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-use-postgresql-with-your-django-application-on-ubuntu-16-04) from Digital Ocean. There is lot more complete information there.
Note: Set timezone to your timezone. More info [here](https://www.postgresql.org/docs/7.2/static/timezones.html).

Let’s switch back to our `mytardis` user and configure the database in Django:
```
ubuntu@mytardis-manual:~$ sudo -i -u mytardis
mytardis@mytardis-manual:~$ source /usr/share/virtualenvwrapper/virtualenvwrapper.sh
mytardis@mytardis-manual:~$ workon mytardis
(mytardis) mytardis@mytardis-manual:~$ cd mytardis
(mytardis) mytardis@mytardis-manual:~/mytardis$ vi tardis/settings.py
```

Add the following to  `settings.py` :
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'mytardis_db',
        'USER': 'mytardis',
        'PASSWORD': 'makeitstronger',
        'HOST': 'localhost',
        'PORT': '',
    }
}
```

We now need to re-run the migrations first and then create the `default_cache` and `celery_lock_cache` tables.
```
(mytardis) mytardis@mytardis-manual:~/mytardis$ python mytardis.py migrate
Operations to perform:
  Synchronize unmigrated apps: mustachejs, search, publication_forms, staticfiles, admindocs, templatetags, bootstrapform, analytics, django_extensions, oaipmh, tastypie_swagger, humanize
  Apply all migrations: sessions, admin, tastypie, djcelery, sites, kombu_transport_django, tardis_portal, contenttypes, auth, registration
Synchronizing apps without migrations:
  Creating tables...
    Running deferred SQL...
  Installing custom SQL...
Running migrations:
  Rendering model states... DONE
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying djcelery.0001_initial... OK
  Applying kombu_transport_django.0001_initial... OK
  Applying registration.0001_initial... OK
  Applying registration.0002_registrationprofile_activated... OK
  Applying registration.0003_migrate_activatedstatus... OK
  Applying sessions.0001_initial... OK
  Applying sites.0001_initial... OK
  Applying tardis_portal.0001_squashed_0011_auto_20160505_1643...
 OK
  Applying tastypie.0001_initial... OK
```

```
(mytardis) mytardis@mytardis-manual:~/mytardis$ python mytardis.py createcachetable default_cache
(mytardis) mytardis@mytardis-manual:~/mytardis$ python mytardis.py createcachetable celery_lock_cache
```

## Serving through Nginx
By default MyTardis runs on port 8000 and it’s not generally advisable to open this to the world. We typically run a simple Nginx server that port forwards the MyTardis server and it’s static assets to port 80.

From the `ubuntu` user:
```
ubuntu@mytardis-manual:~$ sudo apt-get install nginx
```

Let’s create a simple Nginx config that port forwards from http://localhost:8000/ (i.e., MyTardis) to port 80.
```
ubuntu@mytardis-manual:~$ sudo rm /etc/nginx/sites-enabled/default
ubuntu@mytardis-manual:~$ sudo vi /etc/nginx/sites-available/mytardis
```

Add the following to `/etc/nginx/sites-available/mytardis`:
```
server {
  listen 80;
  server_name 118.138.241.161;

  location = /favicon.ico { access_log off; log_not_found off; }
  location /static/ {
    root /home/mytardis/mytardis;
  }

  location / {
    include proxy_params;
    proxy_pass http://unix:/tmp/gunicorn.socket;
  }
}
```

Link config into `/etc/nginx/site-enabled`:
```
ubuntu@mytardis-manual:~$ sudo ln -s /etc/nginx/sites-available/mytardis /etc/nginx/sites-enabled/
```

Test nginx config:
```
ubuntu@mytardis-manual:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If you get no errors, restart NGINX.
```
ubuntu@mytardis-manual:~$ sudo systemctl restart nginx
```

By default Ubuntu iptables rules don’t allow port 80 through the firewall. I like to use `ufw`  to control iptables and we can use the ‘Nginx Full’ profile to allow this:
```
ubuntu@mytardis-manual:~$ sudo ufw enable
ubuntu@mytardis-manual:~$ sudo ufw allow OpenSSH
ubuntu@mytardis-manual:~$ sudo ufw allow 'Nginx Full'
ubuntu@mytardis-manual:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
Nginx Full                 ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
Nginx Full (v6)            ALLOW       Anywhere (v6)
```
Note: it is **important** to allow OpenSSH too otherwise you will not be able to SSH back your VM later.
Note: you could iptables rules directly.
Note: If you are using a cloud service provider, you may also have security groups enable. Make sure you allow port 80 (i.e., HTTP) in your security groups too.

If we run our MyTardis server again now, it should be accessible by visiting the IP address of your server in the browser.
```
ubuntu@mytardis-manual:~$ sudo -i -u mytardis
mytardis@mytardis-manual:~$ source /usr/share/virtualenvwrapper/virtualenvwrapper.sh
mytardis@mytardis-manual:~$ workon mytardis
(mytardis) mytardis@mytardis-manual:~$ cd mytardis
(mytardis) mytardis@mytardis-manual:~/mytardis$ ./mytardis.py runserver
```

Navigate to http://serverip/

An important thing to note is that this is only connecting over HTTP, so the connection is not secure.

## Setting up a gunicorn service
For scalability reasons, we typically run MyTardis using gunicorn. Instead of `./mytardis.py runserver` we can run via gunicorn using the gunicorn settings in the mytardis repo:
```
(mytardis) mytardis@mytardis-manual:~/mytardis$ gunicorn -c gunicorn_settings.py \
-b unix:/tmp/gunicorn.socket \
-b 127.0.0.1:8000 \
--log-syslog  \
wsgi:application
```

Make a note of where the `gunicorn` bin is in you virtualenv:
```
(mytardis) mytardis@mytardis-manual:~$ which gunicorn
/home/mytardis/.virtualenvs/mytardis/bin/gunicorn
```

Let’s create a systemd service to manage running gunicorn. Add the following to `/etc/systemd/system/gunicorn.service`:
```
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=mytardis
Group=mytardis
WorkingDirectory=/home/mytardis/mytardis
ExecStart=/home/mytardis/.virtualenvs/mytardis/bin/gunicorn -c gunicorn_settings.py -b unix:/tmp/gunicorn.socket -b 127.0.0.1:8000 --log-syslog wsgi:application

[Install]
WantedBy=multi-user.target
```
Note that ExecStart is basically calling gunicorn from our virtualenv (see above) with the previous command we used to start it manually.

We can now launch the service and check the status:
```
ubuntu@mytardis-manual:~$ sudo systemctl start gunicorn
ubuntu@mytardis-manual:~$ sudo systemctl status gunicorn
● gunicorn.service - gunicorn daemon
   Loaded: loaded (/etc/systemd/system/gunicorn.service; disabled; vendor preset: enabled)
   Active: active (running) since Wed 2018-05-16 01:35:12 UTC; 9min ago
 Main PID: 27776 (gunicorn)
   CGroup: /system.slice/gunicorn.service
           ├─27776 /home/mytardis/.virtualenvs/mytardis/bin/python2 /home/mytardis/.virtualenvs/mytardis/bin/gunicorn -c gunicorn_settings.py -b unix:/tmp/gunicorn.soc
           ├─27781 /home/mytardis/.virtualenvs/mytardis/bin/python2 /home/mytardis/.virtualenvs/mytardis/bin/gunicorn -c gunicorn_settings.py -b unix:/tmp/gunicorn.soc
           ├─27782 /home/mytardis/.virtualenvs/mytardis/bin/python2 /home/mytardis/.virtualenvs/mytardis/bin/gunicorn -c gunicorn_settings.py -b unix:/tmp/gunicorn.soc
           ├─27783 /home/mytardis/.virtualenvs/mytardis/bin/python2 /home/mytardis/.virtualenvs/mytardis/bin/gunicorn -c gunicorn_settings.py -b unix:/tmp/gunicorn.soc
           ├─27784 /home/mytardis/.virtualenvs/mytardis/bin/python2 /home/mytardis/.virtualenvs/mytardis/bin/gunicorn -c gunicorn_settings.py -b unix:/tmp/gunicorn.soc
           └─27785 /home/mytardis/.virtualenvs/mytardis/bin/python2 /home/mytardis/.virtualenvs/mytardis/bin/gunicorn -c gunicorn_settings.py -b unix:/tmp/gunicorn.soc
```

We can also now check that MyTardis is available through the browser: http://ipaddress/

## Scheduled and asynchronous tasks using Celery
MyTardis has several tasks that operate asynchronously or on a schedule. This is done using [Celery](http://www.celeryproject.org). Celery uses a messaging queue and while there are some default configs in for MyTardis to use the database as the broker, this is not very efficient. A much better method is to use RabbitMQ as the broker. Let’s install RabbitMQ and configure it:

From `ubuntu` user:
```
ubuntu@mytardis-manual:~$ sudo apt-get install rabbitmq-server
ubuntu@mytardis-manual:~$ sudo systemctl status
```
Note: this installs from the Ubuntu repos. In general, it's better to install the most up-to-date RabbitMQ. See [this page](https://www.rabbitmq.com/install-debian.html)

We need to do a bit of setup to use RabbitMQ in Celery. This includes adding a user and a virtual host:
```
sudo rabbitmqctl add_user mytardis makeitreallystrong
sudo rabbitmqctl add_vhost mytardisvhost
sudo rabbitmqctl set_user_tags mytardis mytardistag
sudo rabbitmqctl set_permissions -p mytardisvhost mytardis ".*" ".*" ".*"
```
Note: this information was sourced from [Using RabbitMQ — Celery 4.1.0 documentation](http://docs.celeryproject.org/en/latest/getting-started/brokers/rabbitmq.html)

```
CELERY_RESULT_BACKEND = 'amqp'
BROKER_URL = 'amqp://mytardis:makeitreallystrong@localhost:5672/mytardisvhost'
```

Let’s test whether this works:
```
(mytardis) mytardis@mytardis-manual:~/mytardis$ ./mytardis.py celeryd -c 2 -Q default
[2018-05-17 10:29:52,277: WARNING/MainProcess] /home/mytardis/.virtualenvs/mytardis/local/lib/python2.7/site-packages/celery/apps/worker.py:161: CDeprecationWarning:
Starting from version 3.2 Celery will refuse to accept pickle by default.

The pickle serializer is a security concern as it may give attackers
the ability to execute any command.  It's important to secure
your broker from unauthorized access when using pickle, so we think
that enabling pickle should require a deliberate action and not be
the default choice.

If you depend on pickle then you should set a setting to disable this
warning and to be sure that everything will continue working
when you upgrade to Celery 3.2::

    CELERY_ACCEPT_CONTENT = ['pickle', 'json', 'msgpack', 'yaml']

You must only enable the serializers that you will actually use.


  warnings.warn(CDeprecationWarning(W_PICKLE_DEPRECATED))


 -------------- celery@mytardis-manual v3.1.25 (Cipater)
---- **** -----
--- * ***  * -- Linux-4.4.0-116-generic-x86_64-with-Ubuntu-16.04-xenial
-- * - **** ---
- ** ---------- [config]
- ** ---------- .> app:         default:0x7f96e228d850 (djcelery.loaders.DjangoLoader)
- ** ---------- .> transport:   amqp://mytardis:**@localhost:5672/mytardisvhost
- ** ---------- .> results:     amqp://
- *** --- * --- .> concurrency: 2 (prefork)
-- ******* ----
--- ***** ----- [queues]
 -------------- .> default          exchange=default(direct) key=default


[2018-05-17 10:29:53,753: WARNING/MainProcess] celery@mytardis-manual ready.
```

We need to setup two managed services for most MyTardis deployments:
- Celerybeat service to run scheduled MyTardis tasks
- Celeryworkers to process asynchronous tasks required by MyTardis

Add the following to `/etc/systemd/service/celerybeat.service`:
```
[Unit]
Description=celerybeat daemon
After=network.target

[Service]
User=mytardis
Group=mytardis
WorkingDirectory=/home/mytardis/mytardis
ExecStart=/home/mytardis/.virtualenvs/mytardis/bin/python mytardis.py celerybeat

[Install]
WantedBy=multi-user.target
```

Add the following to `/etc/systemd/service/celeryworker.service`:
```
[Unit]
Description=celeryworker daemon
After=network.target

[Service]
User=mytardis
Group=mytardis
WorkingDirectory=/home/mytardis/mytardis
ExecStart=/home/mytardis/.virtualenvs/mytardis/bin/python mytardis.py celeryd -c 2 -Q celery,default -n "allqueues.%%h"

[Install]
WantedBy=multi-user.target
```

It's good practice in MyTardis to have a Celery beat task that makes sure any files that missed verification get reverified. We can add this to `settings.py`:
```
CELERYBEAT_SCHEDULE = {
    'task': "tardis_portal.verify_dfos",
    'schedule': timedelta(seconds=3600)
}
```

We need to restart our `celerybeat` service to enable the new setting.
```
ubuntu@mytardis-manual:~$ sudo systemctl restart celerybeat
```

## That's it
We should now have a fully functional basic version of MyTardis.

### TODO

- Working with external storage
- Adding filters
