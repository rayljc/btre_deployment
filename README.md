Deployed site: http://40.74.65.18/

# Django Deployment to Ubuntu 18.04
By Ray Jui-Chien Lin, July 2019.

## Get Azure student subscription and 100 dollars

Use your .edu email to register and check the remaining dollars before deployment.
![image](https://github.com/rayljc/btre_deployment/blob/master/image/Azure_usage.png)

## Create An Unbuntu Virtual Machine 

You can set ssh login and inbound ports at this time.
![image](https://github.com/rayljc/btre_deployment/blob/master/image/Create_VM.png)

# Security & Access

### Login To Your Server

If you setup SSH keys correctly the command below will let you right in. 

```
$ ssh USER_NAME@YOUR_SERVER_IP
```

e.g.

```
ssh djangoadmin@40.74.65.18 
```

![image](https://github.com/rayljc/btre_deployment/blob/master/image/ssh_vm.png)

# Software

## Update packages

```
# sudo apt update
# sudo apt upgrade
```

## Install Python 3, Postgres & NGINX

```
# sudo apt install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx curl
```

# Postgres Database & User Setup

```
# sudo -u postgres psql
```

You should now be logged into the pg shell

### Create a database

```
CREATE DATABASE btre_prod;
```

### Create user

```
CREATE USER dbadmin WITH PASSWORD 'abc123!';
```

### Set default encoding, tansaction isolation scheme (Recommended from Django)

```
ALTER ROLE dbadmin SET client_encoding TO 'utf8';
ALTER ROLE dbadmin SET default_transaction_isolation TO 'read committed';
ALTER ROLE dbadmin SET timezone TO 'UTC';
```

### Give User access to database

```
GRANT ALL PRIVILEGES ON DATABASE btre_prod TO dbadmin;
```

### Quit out of Postgres

```
\q
```

# Vitrual Environment

You need to install the python3-venv package

```
# sudo apt install python3-venv
```

### Create project directory

```
# mkdir pyapps
# cd pyapps
```

### Create venv

```
# python3 -m venv ./venv
```

### Activate the environment

```
# source venv/bin/activate
```

# Git & GitHub

### Clone the project into the app folder on your server (Either HTTPS or setup SSH keys)

```
# git clone https://github.com/yourgithubname/btre_project.git
```

## Install pip modules from requirements

You could manually install each one as well

```
# pip install -r requirements.txt
```

# Local Settings Setup

Add code to your settings.py file and push to server

```
try:
    from .local_settings import *
except ImportError:
    pass
```

Create a file called **local_settings.py** on your server along side of settings.py and add the following

- SECRET_KEY
- DEBUG
- ALLOWED_HOSTS
- DATABASES
- EMAIL\_\*

Delete information in settings.py that was added into local_settings.py

## Run Migrations
```
# python manage.py makemigrations
# python manage.py migrate
```

## Create super user

```
# python manage.py createsuperuser
```

## Create static files
```
python manage.py collectstatic
```

### Create exception for port 8000

```
# sudo ufw allow 8000
```

## Run Server

```
# python manage.py runserver 0.0.0.0:8000
```

![image](https://github.com/rayljc/btre_deployment/blob/master/image/0000_8000.png)

### Test the site at YOUR_SERVER_IP:8000

Add some data in the admin area

# Gunicorn Setup

Install gunicorn

```
# pip install gunicorn
```

Add to requirements.txt

```
# pip freeze > requirements.txt
```

### Test Gunicorn serve

```
# gunicorn --bind 0.0.0.0:8000 btre.wsgi
```

Your images, etc will be gone

### Stop server & deactivate virtual env

```
ctrl-c
# deactivate
```

### Open gunicorn.socket file

```
# sudo nano /etc/systemd/system/gunicorn.socket
```

### Copy this code, paste it in and save

```
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

### Open gunicorn.service file

```
# sudo nano /etc/systemd/system/gunicorn.service
```

### Copy this code, paste it in and save

```
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=djangoadmin
Group=www-data
WorkingDirectory=/home/djangoadmin/pyapps/btre_project
ExecStart=/home/djangoadmin/pyapps/venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          btre.wsgi:application

[Install]
WantedBy=multi-user.target
```

### Start and enable Gunicorn socket

```
# sudo systemctl start gunicorn.socket
# sudo systemctl enable gunicorn.socket
```

### Check status of guinicorn

```
# sudo systemctl status gunicorn.socket
```

### Check the existence of gunicorn.sock

```
# file /run/gunicorn.sock
```

# NGINX Setup

### Create project folder

```
# sudo nano /etc/nginx/sites-available/btre_project
```

### Copy this code and paste into the file

```
server {
    listen 80;
    server_name YOUR_IP_ADDRESS;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/djangoadmin/pyapps/btre_project;
    }
    
    location /media/ {
        root /home/djangoadmin/pyapps/btre_project;    
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

### Enable the file by linking to the sites-enabled dir

```
# sudo ln -s /etc/nginx/sites-available/btre_project /etc/nginx/sites-enabled
```

### Test NGINX config

```
# sudo nginx -t
```

### Restart NGINX

```
# sudo systemctl restart nginx
```

### Remove port 8000 from firewall and open up our firewall to allow normal traffic on port 80

```
# sudo ufw delete allow 8000
# sudo ufw allow 'Nginx Full'
```

### You will probably need to up the max upload size to be able to create listings with images

Open up the nginx conf file

```
# sudo nano /etc/nginx/nginx.conf
```

### Add this to the http{} area

```
client_max_body_size 20M;
```

### Reload NGINX

```
# sudo systemctl restart nginx
```

### Media File Issue
You may have some issues with images not showing up. That's normal and you can re-upload them.
