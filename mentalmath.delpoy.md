# Index
- [Deployment guide](#deployment-guide)
	- [Disclaimer](#disclaimer)
	- [Assumptions](#assumptions)
	- [Configuring PostgreSQL](#Configuring-PostgreSQL)
	- [Installing Python virtual environment](#Installing-Python-virtual-environment)
	- [Configuring Django](#Configuring-Django)
		- [settings.py](#settings.py)
		- [Migrations](#Migrations)
		- [Test run](#Test-run)
	- [Configuring a production-ready setup](#Configuring-a-production-ready-setup)
		- [nginx](#nginx)
		- [gunicorn](#gunicorn)
	- [Setting up SSL](#Setting-up-SSL)
	- [Production settings for Django](#Production-settings-for-Django)
- [Creating new users](#Creating-new-users)
	- [Creating students](#Creating-students)
	- [Creating a superuser](#Creating-a-superuser)
- [Copying the production database](#Copying-the-production-database)

# Delpoyment guide
## Disclaimer

Here is a guide on how to delpoy the mental math training website.
I am **not** a Django professional at all, so please take everything here with a hint of doubt. 
This guide was created after exploring the current production server, as well as reading a thousand other guides on Django. Nevertheless, this guide should give you a starting point in case you need it.

*Reverse-engineered from the production server by p. tessman. 14/08/2024.*
## Assumptions

- We are using a virtual private server with Ubuntu 24.04.
- The server has been set up with the [VPS Quick Start Guide](https://github.com/itgentech/configs-and-guides/blob/main/VPS%20Quick%20Start.md).
- Default firewall rules for the input chain are DROP, with all ports closed (besides the minimal working firewall configuration described in the  VPS Quick Start Guide).

## Configuring PostgreSQL

We are going to start with installing and configuring `postgresql`. Install it:

```bash
sudo apt install postgresql -y
```

Now perform some configuration. First, go to `psql` shell:

```bash
cd
sudo -u postgres psql
```

Create a new user for our django application (more on this [here](https://www.postgresql.org/docs/8.0/sql-createuser.html)):

```sql
create user main_user with password '<a-strong-pg-password>' superuser;
```

_Note: Actually, assigning a superuser authority here is **not a good idea**. I leave it like this on simplification purpose. So if you know what you're doing - change it._

Check that the user has been created, then exit from psql shell:

```
postgres=# \du
postgres=# \q
```

Create a database (more info [here](https://www.postgresql.org/docs/current/app-createdb.html)):

```bash
sudo -u postgres createdb -T template0 mental_itgenio
```

Ensure that the database has been created. The `mental_itgenio` database should appear in the output list:

```bash
sudo -u postgres psql --list
```

## Installing Python virtual environment

Create a directory for the project and go to the directory:

```bash
cd
mkdir mentalmath
cd mentalmath
```

Clone the repo:

```bash
git clone git@github.com:itgenio/mentalITGENIOsite.git 
```

Install additional python packages for working with virtual environment:

```bash
sudo apt install python3-dev python3-venv -y
```

Create a virtual environment:

```bash
python3 -m venv .venv
```

The `.venv` directory should appear. Activate the environment:

```bash
chmod +x .venv/bin/activate
source .venv/bin/activate
```

Install some packages:

```shell
pip install Django
	pip install djangorestframework
pip install xlsxwriter
pip install python-dateutil
pip install jsonschema
pip install gunicorn
```

Install psycopg2:

```bash
pip install psycopg2
```

In case you face an error while compiling psycopg2, install psycopg2-binary instead:

```bash
pip install psycopg2-binary
```

Some additional utilities:

```bash
sudo apt install npm -y
sudo npm install -g uglify-js
sudo npm install -g uglifycss
```

## Configuring Django

Go to the project directory:

```bash
cd ~/mentalmath/mentalITGENIOsite
```

### settings.py

Let's make some changes to the Django settings. Open `settings.py` with your favourite text editor:

```bash
nvim mentalITGENIO/settings.py
```

We will make the first run without SSL. So set this variables to false:

```python
CSRF_COOKIE_SECURE = False
SESSION_COOKIE_SECURE = False
SECURE_SSL_REDIRECT = False
```

Fill the `ALLOWED_HOSTS` with proper credentials of your server:

```python
ALLOWED_HOSTS = [
   'mentalmath.itgen.io',
   'www.mentalmath.itgen.io',
   '167.71.33.7'
]
```

Fill the `DATABASES` with the proper credentials (you've set it on the step 1 of this guide, while configuring postgres).

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'mental_itgenio',
        'USER': 'main_user',
        'PASSWORD': '1111',
        'HOST': 'localhost',
        'PORT': ''
    }
}
```

**Note: It is safer to invoke the db credentials from the environment variables.**

Also, for the beauty while testing, we can set `DEDUG`:

```python
DEBUG = True
```

Save `settings.py` and exit.

Also, collect static files:

```bash
python3 manage.py collectstatic
```

### Migrations

Let Django to mark up the database. Try running the folowing command:

```zsh
python3 manage.py makemigrations
```

You probably will get this error:
```
ImportError: cannot import name 'ugettext_lazy' from 'django.utils.translation' (/home/itgen-tech/mentalmath/.venv/lib/python3.12/site-packages/django/utils/translation/__init__.py). Did you mean: 'gettext_lazy'?
```

In that case, open again `settings.py` and replace the import (on line 14 or somewhere near) with the following:

```python
from django.utils.translation import gettext_lazy as _
```

Basically, just delete `u`, so it is `gettext_lazy` instead of `ugettext_lazy`.
And run the makemigrations again. 

*Note: Django may ask you to provide a default value for `points`. As I understood, it should be some integer value. In that case select `1`, then type `0`. I'm not sure about this step, but with `0` it seems to be working fine.*

Now, commit changes:

```bash
python3 manage.py migrate
```

### Test run

First, open some port in iptables (you've configured iptables in [VPS Quick Start](https://github.com/itgentech/configs-and-guides/blob/main/VPS%20Quick%20Start.md), right?):

```bash
sudo iptables -A INPUT -p tcp --dport=42673 -j ACCEPT
```

Then run the django server on this port:

```bash
python3 manage.py runserver 0.0.0.0:42673
```

And open <server_ip>:42673 in your browser (like a regular website). You should see the login page.

## Configuring a production-ready setup

More info [here](https://docs.djangoproject.com/en/5.0/howto/deployment/) and [here](https://developer.mozilla.org/en-US/docs/Learn/Server-side/Django/Deployment).

Basically, our django app is ready, but using the built-in django server on production is a bad idea. So we will configure the following setup:

```
web-clients (browsers) <-> nginx <-> gunicorn <-> our django app
```

Nginx will be collecting all the requests coming from the web to our server. It will filter them and forward the legal ones to gunicorn via a unix socket.
Gunicorn will be listening the unix socket and feeding the requests to our django app.

### nginx

More info [here](https://uwsgi-docs.readthedocs.io/en/latest/tutorials/Django_and_nginx.html#configure-nginx-for-your-site).

Fisrt, install nginx to  the OS:

```bash
sudo apt install nginx -y
```

Create a configuration file:

```bash
sudo touch /etc/nginx/sites-available/mental-math.conf
```

Open it with your favourite text editor:

```bash
sudo nvim /etc/nginx/sites-available/mental-math.conf
```

Paste this config (do not forget to modify the domain name and other stuff):

```nginx
server {
# set the proper urls and ip
    server_name www.mentalmath.itgen.io mentalmath.itgen.io 167.71.33.7;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {

	# set the proper path
        root /home/main/mental_itgenio/mentalITGENIOsite;
    }

    location / {
        include proxy_params;

	# Set the proper path for the unix-socket file.
	# You do not need to create the file itself -
	# it will be created by gunicorn later.
	# Just remember the path.
        proxy_pass http://unix:/home/main/mental_itgenio/mentalITGENIOsite/mental_itgenio.sock;
    }
}
```

Save the file and exit. Then create a symlink:

```bash
sudo ln -s /etc/nginx/sites-available/mental-math.conf /etc/nginx/sites-enabled/
```

Restart nginx:

```bash
sudo systemctl restart nginx
```

Open ports 80 and 443 in your iptables, and do not forget to remove the rule for port 42673 (we opened it earlier for testing):

```bash
sudo iptables -A INPUT -p tcp --dport=80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport=443 -j ACCEPT
sudo iptables -D INPUT -p tcp --dport=42673 -j ACCEPT
```

Save iptables configuration:

```zsh
sudo netfilter-persistent save
```

Now open the website url in your browser. You should see the error 502, as we have not configured gunicorn yet.

**A side note**

If you store the socket file in your project directory, you may want to add the `www-data` user to your user group. Otherwise, nginx may not have enough permissions to work with the `.sock` file. 

Although, it doesn't seem like a good practice, and you'd better store the socket in the `/tmp` directory or, even better, somewhere in the `$XDG_RUNTIME_DIR`.  

More info [here](https://stackoverflow.com/questions/43124765/is-tmp-the-right-place-for-unix-domain-socket-files).

But in case you do store the socket file in the project directory, here's how you would add `www-data` to your user group.

```zsh
sudo gpasswd -a www-data <your-username>
sudo nginx -s reload
```

**End of the side note.**

### gunicorn

We have already installed gunicorn in our `.venv`. So, let's try rinning it. You should run the following command from the django project directory (where `manage.py` is located) with the `.venv` activated. 
Do not forget to specify a proper path to the .sock file:

```zsh
# to go to the project dir. replace with the proper path
cd ~/mentalmath/mentallITGENIOsite

# to run gunicorn. replace with the proper path to the .sock file
gunicorn --access-logfile - --workers 3 --bind unix:/home/itgen-tech/mentalmath/mentalITGENIOsite/mental_itgenio.sock mentalITGENIO.wsgi:application
```

With that, our gunicorn server should be up and running, as well as nginx. 
So, open the website url in your browser. You should see the login page.

Now, let's setup a systemd daemon for gunicorn. This will allow to run the server automatically in the background.

Create a file and open it with your favourite text editor:

```bash
sudo touch /etc/systemd/system/mmath-gunicorn.service
sudo nvim /etc/systemd/system/mmath-gunicorn.service
```

Paste this config and replace paths with the porper ones:

```bash
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=root
Group=www-data

# directory of the django app (where manage.py is located)
WorkingDirectory=/home/main/mentalmath/mentalITGENIOsite

# directory of the .venv, then path to the .sock file
ExecStart=/home/main/mentalmath/.venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/main/mentalmath/mentalITGENIOsite/mental_itgenio.sock mentalITGENIO.wsgi:application

[Install]
WantedBy=multi-user.target
```

Save and exit. Then start the service.

```zsh
sudo systemctl enable mmath-gunicorn.service
sudo systemctl start mmath-gunicorn.service
```

The server should start, and you should be able to load the login page.
Let's stop the server for the last configuration part.

```zsh
sudo systemctl stop mmath-gunicorn.service
```

## Setting up SSL

We are at the finish line. Now let's set up an SSL. We will use `certbot` for that purpose. More info [here](https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal).

First, let's install it:

```zsh
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

Configure certbot for nginx:

```bash
sudo certbot --nginx
```

You will be prompted to provide some additional info. Just complete the steps, and then run the service again:

```zsh
sudo systemctl start mmath-gunicorn.service
```

After this, the website should be loading with secured connection.

## Production settings for Django

Now we are ready for our last step - set the Django settings for serving in production. Open settings.py:

```bash
cd ~/mentalmath/mentalITGENIOsite
nvim mentalITGENIO/settings.py
```

Change some variables. Set them like this:

```python
DEBUG = False
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_SECURE = True
SECURE_SSL_REDIRECT = True
```

Save, exit and restart gunicorn:

```bash
sudo systemctl restart mmath-gunicorn.service
```

Congratulations! You've finished the deployment.

I recommend you to visit [this page](https://docs.djangoproject.com/en/5.1/howto/deployment/checklist/) and doublecheck everything. 

# Creating new users 
## Creating students

As we created a new database, it seems like we don't have any users yet, neither teachers or students. Let's create some to work with.

To autocreate some student accounts, open the Django's interactive shelll (do not forget to activate `.venv`):

```bash
cd ~/mentalmath/mentalITGENIOsite/
python3 manage.py shell
```

Do some imports to work with:

```python
from accounts.models import User, Student
from accounts.tools import filler
import json
```

And call the filler's method:

```python
filler.add_data_users()
```

This will create student accounts with names specified in `accounts/tools/student_data.txt`.

The file `accounts/tools/output_data.txt` will store logins and passwords. 

Type `exit()` to exit from the shell mode.

## Creating a superuser

To create a superuser, run the following command:

```zsh
python3 manage.py createsuperuser
```

You will be prompted to specify credentials.

#  Copying the production database

The server is hosted on Digital Ocean with backups enabled, so this section is not concidered as the main way of backing up the project. But it might be useful in some cases. 

To copy the database, we will be using [SQL dump](https://www.postgresql.org/docs/current/backup-dump.html).

Ssh to the production server and create a dump:

```bash
sudo -u postgres pg_dump > /home/main/dumpfile-$(date +%F).sqldmp
```

Then, transfer this dump file to the server you're working on and restore it:

```bash
sudo -u postgres dropdb mental_itgenio
sudo -u postgres createdb -T template0 mental_itgenio
sudo -u postgres psql --set ON_ERROR_STOP=on mental_itgenio < /path/to/dump.sqldmp
```