# Project 5 of the Udacity Full Stack Nanodegree programme
_Configuring a Linux server to host a web app securely._

# Server details
IP address: `18.219.251.159`

SSH port: `2200`

URL: `http://ec2-18-219-251-159.us-east-2.compute.amazonaws.com`


# Configuration changes
## Add user
Add user `grader` with command: `useradd -m -s /bin/bash grader`

## Add user grader to sudo group
Assuming your Linux distro has a `sudo` group (like Ubuntu 16.04), simply add the user to
this admin group:
```
usermod -aG sudo grader
```

## Update all currently installed packages

`apt-get update` - to update the package indexes

`apt-get upgrade` - to actually upgrade the installed packages

If at login the message `*** System restart required ***` is display, run the following
command to reboot the machine:

`reboot`

## Set-up SSH keys for user grader
As root user do:
```
mkdir /home/grader/.ssh
chown grader:grader /home/grader/.ssh
chmod 700 /home/grader/.ssh
cp /root/.ssh/authorized_keys /home/grader/.ssh/
chown grader:grader /home/grader/.ssh/authorized_keys
chmod 644 /home/grader/.ssh/authorized_keys
```

Can now login as the `grader` user using the command:
`ssh -i ~/.ssh/udacity_key.rsa grader@18.219.251.159 -p 22`

## Disable root login
Change the following line in the file `/etc/ssh/sshd_config`:

From `PermitRootLogin without-password` to `PermitRootLogin no`.

Also, uncomment the following line so it reads:
```
PasswordAuthentication no
```

Do `service ssh restart` for the changes to take effect.

Will now do all commands using the `grader` user, using `sudo` when required.

## Change timezone to UTC
Check the timezone with the `date` command. This will display the current timezone after the time.
If it's not UTC change it like this:

`sudo timedatectl set-timezone UTC`

## Change SSH port from 22 to 2200
Edit the file `/etc/ssh/sshd_config` and change the line `Port 22` to:

`Port 2200`

Then restart the SSH service:

`sudo service ssh restart`

Will now need to use the following command to login to the server:

`ssh -i ~/.ssh/udacity_key.rsa grader@18.219.251.159 -p 2200`

## Configuration Uncomplicated Firewall (UFW)
By default, block all incoming connections on all ports:

`sudo ufw default deny incoming`

Allow outgoing connection on all ports:

`sudo ufw default allow outgoing`

Allow incoming connection for SSH on port 2200:

`sudo ufw allow 2200/tcp`

Allow incoming connections for HTTP on port 80:

`sudo ufw allow www`

Allow incoming connection for NTP on port 123:

`sudo ufw allow ntp`

To check the rules that have been added before enabling the firewall use:

`sudo ufw show added`

To enable the firewall, use:

`sudo ufw enable`

To check the status of the firewall, use:

`sudo ufw status`

Install Apache
`sudo apt-get install apache2`

Install mod_wsgi
  - Run `sudo apt-get install libapache2-mod-wsgi python-dev`
  - Enable mod_wsgi with `sudo a2enmod wsgi`
  - Start the web server with `sudo service apache2 start`

  
Clone the Catalog app from Github
  - Install git using: `sudo apt-get install git`
  - `cd /var/www`
  - Clone your project from github `sudo git clone https://github.com/vasasharath/Item_Catalog.git`
  - `sudo mv Item_Catalog catalog`
  - Change owner of the newly created catalog folder `sudo chown -R grader:grader catalog`
  - `cd /catalog`
  - Create a catalog.wsgi file, then add this inside:
  ```
  #!/usr/bin/python
  activate_this = '/var/www/catalog/venv/bin/activate_this.py'
  execfile(activate_this, dict(__file__=activate_this))
  import sys
  import logging
  logging.basicConfig(stream=sys.stderr)
  sys.path.insert(0, "/var/www/catalog/")

  from __init__ import app as application
  application.secret_key = 'supersecretkey'

  ```
  - Rename application.py to __init__.py `mv application.py __init__.py`
  
Install virtual environment
  - Install the virtual environment `sudo pip install virtualenv`
  - Create a new virtual environment with `sudo virtualenv venv`
  - Activate the virutal environment `source venv/bin/activate`
  - Change permissions `sudo chmod -R 777 venv`

Install Flask and other dependencies
  - Install pip with `apt-get install python-pip`
  - Install Flask `pip install Flask`
  - Install other project dependencies `pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils`

Update path of client_secrets.json file
  - `nano __init__.py`
  - Change client_secrets.json path to `/var/www/catalog/client_secrets.json`
Configure and enable a new virtual host
  - Run this: `sudo nano /etc/apache2/sites-available/catalog.conf`
  - Paste this code: 
  ```
  <VirtualHost *:80>
    ServerName mywebsite.com
    ServerAdmin admin@mywebsite.com
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/static
    ServerAlias ec2-18-219-251-159.us-east-2.compute.amazonaws.com
    <Directory /var/www/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>
  ```
  - Enable the virtual host `sudo a2ensite catalog`

Install and configure PostgreSQL
  - `sudo apt-get install libpq-dev python-dev`
  - `sudo apt-get install postgresql postgresql-contrib`
  - `sudo su - postgres`
  - `psql`
  - `CREATE USER catalog WITH PASSWORD 'password';`
  - `ALTER USER catalog CREATEDB;`
  - `CREATE DATABASE catalog WITH OWNER catalog;`
  - `\c catalog`
  - `REVOKE ALL ON SCHEMA public FROM public;`
  - `GRANT ALL ON SCHEMA public TO catalog;`
  - `\q`
  - `exit`
  - Change create engine line in your `__init__.py` and `database_setup.py` to: 
  `engine = create_engine('postgresql://catalog:password@localhost/catalog')`
  - `python /var/www/catalog/database_setup.py`
  - Make sure no remote connections to the database are allowed. Check if the contents of this file `sudo nano /etc/postgresql/9.3/main/pg_hba.conf` looks like this:
  ```
  local   all             postgres                                peer
  local   all             all                                     peer
  host    all             all             127.0.0.1/32            md5
  host    all             all             ::1/128                 md5
  ```
  
16. Restart Apache 
  - `sudo service apache2 restart`
  
17. Visit site at [http://18.219.251.159](http://18.219.251.159)
