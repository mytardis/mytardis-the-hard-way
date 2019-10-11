# MyTardis 4.1 on Ubuntu 18.04 LTS
This guide assumes that you already have an Ubuntu 18.04 LTS server or virtual
machine (VM) running. It's also assumes that you have some familiarity setting
up a local development enviroment for MyTardis following
[this guide](https://mytardis.readthedocs.io/en/develop/admin/install.html).

## Getting started

Let's set the timezone first:

```
ubuntu@mytardis-ubuntu18:~$ sudo timedatectl set-timezone Australia/Melbourne
```

We need to install some dependencies. From
[mytardis.readthedocs.io](https://mytardis.readthedocs.io/en/develop/admin/install.html):

```
ubuntu@mytardis-ubuntu18:~$ sudo apt-get update
ubuntu@mytardis-ubuntu18:~$ sudo apt-get install \
   git libldap2-dev libmagickwand-dev libsasl2-dev \
   libssl-dev libxml2-dev libxslt1-dev libmagic-dev curl gnupg \
   python3-dev python3-pip python3-venv zlib1g-dev libfreetype6-dev libjpeg-dev
ubuntu@mytardis-ubuntu18:~$ sudo pip3 install virtualenvwrapper

ubuntu@mytardis-ubuntu18:~$ curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
ubuntu@mytardis-ubuntu18:~$ sudo apt-get install -y nodejs
```

## Installing MyTardis
Let’s create a group and user for our web app:

```
ubuntu@mytardis-ubuntu18:~$ sudo groupadd -r mytardis
ubuntu@mytardis-ubuntu18:~$ sudo useradd -m -r -g mytardis -s /bin/bash mytardis
```

Let’s switch to the new user:

```
ubuntu@mytardis-ubuntu18:~$ sudo -i -u mytardis
```

Configure virtualenvwrapper:

```
mytardis@mytardis-ubuntu18:~$ cat << EOF >> ~/.bashrc
> export VIRTUALENV_PYTHON=/usr/bin/python3
> export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3
> export WORKON_HOME=$HOME/.virtualenvs
> export PROJECT_HOME=$HOME/mytardis
> source /usr/local/bin/virtualenvwrapper.sh
> EOF
mytardis@mytardis-ubuntu18:~$ source ~/.bashrc 
```

Create and activate a virtualenv:

```
mytardis@mytardis-ubuntu18:~$ mkvirtualenv mytardis
```

Clone mytardis and checkout the v4.1.x branch:

```
(mytardis) ~$ git clone https://github.com/mytardis/mytardis.git
(mytardis) ~$ cd mytardis/
(mytardis) ~/mytardis$ git checkout -b series-4.1 origin/series-4.1
```

Create and activate a virtualenv:

Install pip dependencies:

```
(mytardis) ~/mytardis$ pip install -r requirements.txt
```

Install pip dependencies for Postgresql database engine:

```
(mytardis) ~/mytardis$ pip install -r requirements-postgres.txt
```

Install Javascript dependencies:

```
(mytardis) ~/mytardis$ npm install
```

Make a directory to store the uploaded files:

```
(mytardis) ~/mytardis$ mkdir -p var/store
```
Note: In production, we would most likely mount a CIFS or NFS volume to host the data.

Let’s create a secret key:
```
(mytardis) ~/mytardis$ python -c "import os; from random import choice; key_line = '%sSECRET_KEY=\"%s\"  # generated from build.sh\n' % ('from tardis.default_settings import * \n\n' if not os.path.isfile('tardis/settings.py') else '', ''.join([choice('abcdefghijklmnopqrstuvwxyz0123456789@#%^&(-=+)') for i in range(50)])); f=open('tardis/settings.py', 'a+'); f.write(key_line); f.close()"
```
and run the mytardis unit tests to see if every seems okay with the environment:

```
(mytardis) ~/mytardis$ python test.py
nosetests --verbosity=1
Creating test database for alias 'default'...

...........S.S.S.S.S.......................................................................................................S.S.S.S.S.S.S....................................................
----------------------------------------------------------------------
Ran 176 tests in 24.138s

OK (SKIP=11)
Destroying test database for alias 'default'...
```

Now let's run mytardis's Javascript unit tests.  This requires running the full "npm install" as above, rather than just running "npm install --production" to install the bare minimum Javascript dependencies.

To ensure that Chromium can run the Javascript tests, you might need to install some additional packages:

```
(mytardis) mytardis@era2019-base:~/mytardis$ exit
logout
ubuntu@mytardis-ubuntu18:~$ sudo apt install -y gconf-service libasound2 libatk1.0-0 libc6 \
  libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 libgcc1 \
  libgconf-2-4 libgdk-pixbuf2.0-0 libglib2.0-0 libgtk-3-0 libnspr4 \
  libpango-1.0-0 libpangocairo-1.0-0 libstdc++6 libx11-6 \
  libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 \
  libxext6 libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 \
  libxtst6  ca-certificates fonts-liberation libappindicator1 \
  libnss3 lsb-release xdg-utils wget
```

```
ubuntu@mytardis-ubuntu18:~$ sudo -i -u mytardis
mytardis@mytardis-ubuntu18:~$ workon mytardis
(mytardis) mytardis@mytardis-ubuntu18:~$ cd mytardis/
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ npm test

...
>> 5 tests completed with 0 failed, 0 skipped, and 0 todo.
>> 15 assertions (in 84ms), passed: 15, failed: 0

...

 PASS  assets/js/apps/search/__tests__/search.spec.jsx
  Search Component
    ✓ Render Search Component (10ms)
  Results Component
    ✓ Render Results Component (4ms)
  Result Component
    ✓ Test Render Result component (3ms)
    ✓ Test Render Simple Search Form (2ms)
    ✓ Test Render Advance Search Form (2ms)
...
Ran all test suites.
```

Okay for now we are just going to use the `sqlite` database for testing. In fact, this is already configured in MyTardis’s  default settings [here](https://github.com/mytardis/mytardis/blob/develop/tardis/default_settings/database.py). Note we’ll need to do this again when we switch to a production database.

Let's run the migrations to create the database tables in our SQLite database, as described in `~/mytardis/build.sh`:

```
~/mytardis$ python manage.py migrate
~/mytardis$ python manage.py createcachetable default_cache
~/mytardis$ python manage.py createcachetable celery_lock_cache
~/mytardis$ ls -lh db.sqlite3
-rw-r--r-- 1 mytardis mytardis 604K Sep 25 03:10 db.sqlite3
```

Finally, we build our webpack bundles:

```
(mytardis) ~/mytardis$ npm run-script build
```

and collect our static assets:

```
(mytardis) ~/mytardis$ python manage.py collectstatic

258 static files copied to '/home/mytardis/mytardis/static', 300 post-processed.
```

While we’re here, let’s also create a superuser for our admin:
```
(mytardis) ~/mytardis$ python manage.py createsuperuser
```

Let's check if we can run MyTardis using Django's built-in web server (not recommended for production):
```
(mytardis) ~/mytardis$ ./manage.py runserver
```

And from another Terminal:

```
ubuntu@mytardis-ubuntu18:~$ wget http://127.0.0.1:8000
--2018-09-25 03:21:23--  http://127.0.0.1:8000/
Connecting to 127.0.0.1:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 10605 (10K) [text/html]
Saving to: 'index.html’

index.html          100%[===================>]  10.36K  --.-KB/s    in 0s

2018-09-25 03:21:23 (53.2 MB/s) - 'index.html’ saved [10605/10605]
```
Okay looking good.

At this point we could have a play with MyTardis on a local machine, but there are limitations in SQLite (i.e., concurrent DB access) that prevent it from being able to power a full MyTardis. We’ll come back to this later.

## Switching to Postgres DB
To run in production, we really need to switch databases.  Let’s install postgres:

The command below installs the default PostgreSQL version for Ubuntu 18.  If you want to install a specific version, refer the subsequent command.

```
ubuntu@mytardis-ubuntu18:~$ sudo apt-get install postgresql postgresql-contrib
```
Note: we have switched back to the `ubuntu` user.

If you want to install a specific version of PostgreSQL, you can use the PGDG repository.  First download and install the repository's key and apt list file:

```
ubuntu@mytardis-ubuntu18:~$ curl https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
ubuntu@mytardis-ubuntu18:~$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```
Then install your required version of PostgreSQL:
```
ubuntu@mytardis-ubuntu18:~$ sudo apt-get update
ubuntu@mytardis-ubuntu18:~$ sudo apt-get install postgresql-[VERSION]
```

Let’s create a database and credentials for `mytardis`
```
ubuntu@mytardis-ubuntu18:~$ sudo -u postgres psql
psql (10.5 (Ubuntu 10.5-0ubuntu0.18.04))
Type "help" for help.

postgres=# CREATE DATABASE mytardis_db;
postgres=# CREATE USER mytardis WITH PASSWORD 'makeitstronger';
postgres=# ALTER ROLE mytardis SET client_encoding TO 'utf8';
postgres=# ALTER ROLE mytardis SET default_transaction_isolation TO 'read committed';
postgres=# ALTER ROLE mytardis SET timezone TO 'UTC';
postgres=# \q
```
The following guide from DigitalOcean: [How To Set Up Django with Postgres, Nginx, and Gunicorn on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-18-04) provides more detailed information about installing and configuring Postgresql on Ubuntu 18.04.

Also note Django's recommendations for setting the timezone to `UTC` for the database user when `USE_TZ` is True in Django's settings: https://docs.djangoproject.com/en/1.11/ref/databases/#optimizing-postgresql-s-configuration

Let’s switch back to our `mytardis` user and configure the database in Django:

```
ubuntu@mytardis-ubuntu18:~$ sudo -i -u mytardis
mytardis@mytardis-ubuntu18:~$ workon mytardis
(mytardis) mytardis@mytardis-ubuntu18:~$ cd mytardis
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ vi tardis/settings.py
```

Add the following to  `tardis/settings.py` :

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

We now need to re-run the migrations first and then create the `default_cache` and `celery_lock_cache` tables:

```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ python manage.py migrate
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ python manage.py createcachetable default_cache
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ python manage.py createcachetable celery_lock_cache
```

## Serving through Nginx

Django provides a demo web server (which can be run with `python manage.py runserver`), however it should not be used in production, as advised by the Django docs at https://docs.djangoproject.com/en/1.11/ref/django-admin/:

> DO NOT USE THIS SERVER IN A PRODUCTION SETTING. It has not gone through security audits
> or performance tests. (And that’s how it’s gonna stay. We’re in the business of making
> Web frameworks, not Web servers, so improving this server to be able to handle a
> production environment is outside the scope of Django.)

As the `ubuntu` user, let's install nginx:

```
ubuntu@mytardis-ubuntu18:~$ sudo apt-get install nginx
```

Let’s create a simple Nginx config that proxies requests from port 80 (HTTP) to http://localhost:8000/ (i.e. MyTardis running with Django's `manage.py runserver`).

```
ubuntu@mytardis-ubuntu18:~$ sudo rm /etc/nginx/sites-enabled/default
ubuntu@mytardis-ubuntu18:~$ sudo vi /etc/nginx/sites-available/mytardis
```

Add the following to `/etc/nginx/sites-available/mytardis`:

```
server {
  listen 80;
  server_name 118.138.234.127;  # Replace this with your server's IP address

  location = /favicon.ico { access_log off; log_not_found off; }
  location /static/ {
    root /home/mytardis/mytardis;
  }

  location / {
    include proxy_params;

    # We'll use gunicorn in production:
    # proxy_pass http://unix:/tmp/gunicorn.socket;

    # But first we'll test proxying nginx to Django's manage.py runserver:
    proxy_pass http://127.0.0.1:8000;
  }
}
```

Now link this config into `/etc/nginx/site-enabled`:

```
ubuntu@mytardis-ubuntu18:~$ sudo ln -s /etc/nginx/sites-available/mytardis /etc/nginx/sites-enabled/
```

Test nginx config:

```
ubuntu@mytardis-ubuntu18:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If you get no errors, restart NGINX.

```
ubuntu@mytardis-ubuntu18:~$ sudo systemctl restart nginx
```

To be able to access the web server from outside of the Ubuntu 18 machine,
you need to open up a port (80 for HTTP and/or 443 for HTTPS).  The best way
to do that depends on your infrastructure environment.  If using the NeCTAR
research cloud, you should be familiar with security groups - https://support.ehelp.edu.au/support/solutions/articles/6000151135-how-to-create-security-groups-secgroups- and you should be familiar with tools for
managing the Linux firewall, particularly `iptables` and/or `ufw`:
https://help.ubuntu.com/lts/serverguide/firewall.html

If we run our MyTardis server after opening port 80, it should be accessible by visiting the IP address of your server in the browser:

```
ubuntu@mytardis-ubuntu18:~$ sudo -i -u mytardis
mytardis@mytardis-ubuntu18:~$ workon mytardis
(mytardis) mytardis@mytardis-ubuntu18:~$ cd mytardis
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ ./manage.py runserver
```

Navigate to http://serverip/

An important thing to note is that this is only connecting over HTTP, so the connection is not secure.

## Setting up a gunicorn service

As Django's `manage.py runserver` is not suitable for production, Django applications are typically deployed with WSGI as described at https://docs.djangoproject.com/en/1.11/howto/deployment/wsgi/

MyTardis deployments typically use the Gunicorn WSGI server.

Instead of `./mytardis.py runserver` we can run via gunicorn using the gunicorn settings in the mytardis repo:

```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ gunicorn -c gunicorn_settings.py \
  -b unix:/tmp/gunicorn.socket -b 127.0.0.1:8000 --log-syslog  wsgi:application
```
You can press Control-C to stop the Gunicorn processes - normally they would be launched from a Systemd service, not from an interactive Terminal session.

Make a note of where the `gunicorn` bin is in you virtualenv:

```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ which gunicorn
/home/mytardis/.virtualenvs/mytardis/bin/gunicorn
```

Let's create a Systemd service to manage running gunicorn. Add the following to `/etc/systemd/system/gunicorn.service`:

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
ubuntu@mytardis-ubuntu18:~$ sudo systemctl start gunicorn
ubuntu@mytardis-ubuntu18:~$ sudo systemctl status gunicorn
● gunicorn.service - gunicorn daemon
   Loaded: loaded (/etc/systemd/system/gunicorn.service; disabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-09-25 04:02:41 UTC; 4s ago
```

We should now update our nginx config to proxy HTTP requests to Gunicorn's socket (/tmp/gunicorn.socket)
instead of to the Django manage.py runserver at http://127.0.0.1:8000

Our `/etc/nginx/sites-available/mytardis` should be updated to look like this:

```
server {
  listen 80;
  server_name 118.138.234.127;  # Replace this with your server's IP address

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

We can reload the configuration with:

```
ubuntu@mytardis-ubuntu18:~$ sudo service nginx reload
```

Now let's check again that we can access MyTardis through the browser via http://serverip/

You can also check that your requests are being logged in MyTardis's request.log:

```
ubuntu@mytardis-ubuntu18:~$ tail /home/mytardis/mytardis/request.log
```

MyTardis's default log levels can be found in `tardis/default_settings/logging.py`

## Scheduled and asynchronous tasks using Celery
MyTardis has several tasks that operate asynchronously or on a schedule. This is done using [Celery](http://www.celeryproject.org). Celery uses a messaging queue and while there are some default configs in for MyTardis to use the database as the broker, this is not very efficient. A much better method is to use RabbitMQ as the broker. Let’s install RabbitMQ and configure it:

As the `ubuntu` user:

```
ubuntu@mytardis-ubuntu18:~$ sudo apt-get install rabbitmq-server

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

Now we need to add the following to `/home/mytardis/mytardis/tardis/settings.py`:

```
CELERY_RESULT_BACKEND = 'amqp'
BROKER_URL = 'amqp://mytardis:makeitreallystrong@localhost:5672/mytardisvhost'
```

Let’s test whether this works from within our mytardis virtualenv:

```
ubuntu@mytardis-ubuntu18:~$ sudo -i -u mytardis
mytardis@mytardis-ubuntu18:~$ workon mytardis
(mytardis) mytardis@mytardis-ubuntu18:~$ cd mytardis
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ DJANGO_SETTINGS_MODULE=tardis.settings celery worker -A tardis.celery.tardis_app
```

You can press Control-C to terminate your temporary celery worker instance.

Like gunicorn, the celery worker processes should be run as a Systemd service.

Add the following to `/etc/systemd/system/celeryworker.service`:

```
[Unit]
Description=celeryworker daemon
After=network.target

[Service]
User=mytardis
Group=mytardis
WorkingDirectory=/home/mytardis/mytardis
Environment=DJANGO_SETTINGS_MODULE=tardis.settings
ExecStart=/home/mytardis/.virtualenvs/mytardis/bin/celery worker \
  -A tardis.celery.tardis_app \
  -c 2 -Q celery,default -n "allqueues.%%h"

[Install]
WantedBy=multi-user.target
```

We also need a celery beat Systemd service for scheduling tasks.

Add the following to `/etc/systemd/system/celerybeat.service`:

```
[Unit]
Description=celerybeat daemon
After=network.target

[Service]
User=mytardis
Group=mytardis
WorkingDirectory=/home/mytardis/mytardis
Environment=DJANGO_SETTINGS_MODULE=tardis.settings
ExecStart=/home/mytardis/.virtualenvs/mytardis/bin/celery beat \
  -A tardis.celery.tardis_app --loglevel INFO

[Install]
WantedBy=multi-user.target
```

MyTardis's `tardis/default_settings/celery_settings.py` contains a Celery beat
task which makes sure any files that missed verification get reverified.  We
can overwrite the `CELERYBEAT_SCHEDULE` setting in our `tardis/settings.py` and
make the "verify-files" task run more often to confirm that scheduled tasks are
working.  For now, let's set it to run every 30 seconds instead of every 300
seconds.

We can add this to `tardis/settings.py`:

```
from datetime import timedelta
CELERYBEAT_SCHEDULE = {
    "verify-files": {
        "task": "tardis_portal.verify_dfos",
        "schedule": timedelta(seconds=30)
    }
}
```

We can now start the celeryworker and celerybeat services:

```
ubuntu@mytardis-ubuntu18:~$ sudo systemctl start celeryworker
ubuntu@mytardis-ubuntu18:~$ sudo systemctl status celeryworker
● celeryworker.service - celeryworker daemon
   Loaded: loaded (/etc/systemd/system/celeryworker.service; disabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-09-25 04:51:39 UTC; 12s ago
```

```
ubuntu@mytardis-ubuntu18:~$ sudo systemctl start celerybeat
ubuntu@mytardis-ubuntu18:~$ sudo systemctl status celerybeat
● celerybeat.service - celerybeat daemon
   Loaded: loaded (/etc/systemd/system/celerybeat.service; disabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-09-25 05:44:19 UTC; 3s ago
```

If we wait a minute a check the celerybeat service status again, we should see the tasks we have scheduled to run
every 30 seconds:

```
ubuntu@mytardis-ubuntu18:~$ sudo systemctl status celerybeat
● celerybeat.service - celerybeat daemon
   Loaded: loaded (/etc/systemd/system/celerybeat.service; disabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-10-02 09:51:26 AEST; 1min 45s ago
 Main PID: 5570 (celery)
    Tasks: 1 (limit: 4707)
   CGroup: /system.slice/celerybeat.service
           └─5570 /home/mytardis/virtualenvs/mytardis/bin/python2 /home/mytardis/.virtualenvs/mytardis/bin/celery beat -A tardis.celery.tardis_ap

Oct 02 09:51:26 mytardis-ubuntu18 systemd[1]: Started celerybeat daemon.
Oct 02 09:51:28 mytardis-ubuntu18 celery[5570]: [2018-10-02 09:51:28,515: INFO/MainProcess] beat: Starting...
Oct 02 09:51:33 mytardis-ubuntu18 celery[5570]: [2018-10-02 09:51:33,916: INFO/MainProcess] Scheduler: Sending due task verify-files (tardis_porta
Oct 02 09:52:03 mytardis-ubuntu18 celery[5570]: [2018-10-02 09:52:03,926: INFO/MainProcess] Scheduler: Sending due task verify-files (tardis_porta
```

You can check `/var/log/syslog` (or wherever you have chosen to redirect the output of your celerybeat service) for
more information.

Once you have confirmed that the `tardis_portal.verify_dfos` task is being run every 30 seconds, you can remove the
`CELERYBEAT_SCHEDULE` from your `tardis/settings.py` to revert to the default interval of 300 seconds in
`tardis/default_settings/celery_settings.py` and restart the Celery beat service:

```
ubuntu@mytardis-ubuntu18:~$ sudo systemctl restart celerybeat
```

If you want these services to start automatically on boot, you can enable them
as follows:

```
ubuntu@mytardis-ubuntu18:~$ sudo systemctl enable gunicorn
Created symlink /etc/systemd/system/multi-user.target.wants/gunicorn.service → /etc/systemd/system/gunicorn.service.

ubuntu@mytardis-ubuntu18:~$ sudo systemctl enable celeryworker
Created symlink /etc/systemd/system/multi-user.target.wants/celeryworker.service → /etc/systemd/system/celeryworker.service.

ubuntu@mytardis-ubuntu18:~$ sudo systemctl enable celerybeat
Created symlink /etc/systemd/system/multi-user.target.wants/celerybeat.service → /etc/systemd/system/celerybeat.service.
```

## Configuring Search

MyTardis provides a search interface and the ability to index data for fast
lookup.

To enable search in MyTardis v4.1, the first thing to do is to install
Elasticsearch v6.x:

```
sudo apt install apt-transport-https
sudo apt install openjdk-8-jdk
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo sh -c 'echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" > /etc/apt/sources.list.d/elastic-6.x.list'
sudo apt update
sudo apt install elasticsearch
```

Now we can start the Elasticsearch service:

```
sudo systemctl enable elasticsearch.service
sudo systemctl start elasticsearch.service
```

Wait for the service to start - it could take 20 seconds or more.
Then you can check that it is listening on port 9200:

```
curl -X GET "localhost:9200/"
```

Now we can configure our MyTardis instance ot use Elasticsearch by adding
the following to our `tardis/settings.py`:

```
SINGLE_SEARCH_ENABLED = True
INSTALLED_APPS += ('django_elasticsearch_dsl','tardis.apps.search',)
ELASTICSEARCH_DSL = {
    'default': {
        'hosts': 'http://localhost:9200'
    },
}
ELASTICSEARCH_DSL_INDEX_SETTINGS = {
    'number_of_shards': 1
}
```

Now we can use the `search_index` Django command, added by the
`django_elasticsearch_dsl` app:

```
sudo -i -u mytardis
cd ~/mytardis/
workon mytardis
./manage.py search_index --help
```

To build the index for the first time (or rebuild it), you can run:

```
./manage.py search_index --rebuild
```

Now to see the search box in the web interface, you need to restart or reload
the gunicorn service:

```
sudo systemctl restart gunicorn
```

Now should see the Search box in the web interface.  Newly created experiments,
datasets and datafiles will automatically be added to the index.


## That's it
We should now have a fully functional basic version of MyTardis.

### TODO

- SFTP configuration
- Working with external storage
- Adding filters
