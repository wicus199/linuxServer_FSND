# Linux Server Configuration

This README details the steps followed to host the Item Catalog project, done as part of the Full Stack Nanodegree course on Udacity, on an Amazon Lightsail Linux server.

## Server Access
* **IP Address**: 52.56.103.136
* **SSH Port**: 2200

## Wesite URL 
http://ec2-52-56-103-136.eu-west-2.compute.amazonaws.com/

## Server setup process

This section details the steps performed to set up the server to host the Item Catalog project. A summary of the steps are:
* Set up the firewall to only allow incoming connections on certain ports
* Changed the SSH port from 22 to 2200
* Created a `grader` account with `sudo` privileges
* Created an `.ssh` directory for the `grader` user with limited permission
* Generated a key pair for the `grader` user to be able to use ssh to login to the server
* Changed the local timezone to UTC
* Install global packages using the `sudo` command
* Checkout the GitHub repository of the Item Catalog project to the /var/www/itemCatalog directory
* Set up a python3 virtual environment inside the project directory
* Activate the virtual environment and install packages necessary for the project
* Change Apache server configuraion settings
* Set up a database and run the database.py file to create the tables
* Start the server

### Firewall setup
The `ufw` package was used to set up the firewall for the server. The following steps were performed to set up the firewall:
* Deny all incoming conenctions by default: `sudo ufw default deny incoming`
* Allow all outgoing connections by default: `sudo ufw default allow outgoing`
* Allow connections on port `2200` (SSH): `sudo ufw allow 2200/tcp`
* Allow connections on port `80` (HTTP): `sudo ufw allow 80/tcp`
* Allow connections on port `123` (NTP): `sudo ufw allow 123/tcp`
* Enable firewall: `sudo ufw enable`

### Create and configure grader account
* Create the `grader` user: `sudo adduser grader`
* Add `grader ALL=(ALL) NOPASSWD:ALL` to `/etc/sudoers.d/90-cloud-init-users` file to grant `sudo` access to `grader`.
* Create `.ssh` directory in `/home/grader/` to store authorized keys
* Create `authorized_keys` file inside `.ssh` directory
* Generate key pairs using `ssh_keygen` command on a local machine 
* Copy the text inside the locally generated `.pub` file into the `authorized_keys` file on the server
* The `grader` will now be able to login to the server using a public key

The timezone was changed to UTC using the `sudo dpkg-reconfigure tzdata` command.

### The following packages where installed globally (using sudo):
* finger: `sudo apt-get install finger`
* python3 mod_wsgi package: `sudo apt-get install libapache2-mod-wsgi-py3`
* postgresql: `sudo apt-get install postgresql postgresql-contrib`
* Apache: `sudo apt-get install apache2`
* pip for python 3: `sudo apt-get install ptyhon3-pip`
* virtualenv for python 3: `sudo pip3 install virtualenv`
* git: `sudo apt-get install git`

After installing git, the Item Catalog repository where checked out into the `/var/www/itemCatalog/` directory. The folder name was then changed to itemCatalog, resulting in a project directory of `/var/www/itemCatalog/itemCatalog/`.

A `wsgi` application, `itemcatalog.wsgi`, was created in the `/var/www/itemCatalog` directory with the following content:

```python
#!/var/www/itemCatalog/itemCatalog/itemcatvenv/bin/python
import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/itemCatalog/")

from itemCatalog import app as application
application.secret_key = 'super_secret_key'
```

### The virtualenv were set up as follows:
1. Change ownership of the `/var/www/itemCatalog/itemCatalog directory` to the `ubuntu` user to allow creation of the virtualenv without the use of the `sudo` command 
2. Create a virtualenv in the `/var/www/itemCatalog/itemCatalog/` directory using the command python3 -m venv itemcatvenv 
3. Activate the virtual environment using `source itemcatvenv/bin/activate`
4. Install Flask: `pip3 install flask`
5. Install sqlalchemy: `pip3 install sqlalchemy`
6. Install oauth2client: `pip3 install --upgrade oauth2client`
7. Install psycopg2: `pip3 install psycopg2`
8. Install requests: `pip3 install requests`
9. After all packages were installed in the virtualenv, deactivate the environmet using `deactivate`

### Apache server configuration

A config file for the application, `itemCatalog.conf`, was created in the `/etc/apache2/sites-available/` directory. The contents of the file are the following:

```xml
<VirtualHost *:80>
                ServerName 52.56.103.136
                ServerAdmin wicus92@gmail.com
                WSGIDaemonProcess itemCatalog python-home=/var/www/itemCatalog/itemCatalog/itemcatvenv
                WSGIProcessGroup itemCatalog
                WSGIScriptAlias / /var/www/itemCatalog/itemcatalog.wsgi
                <Directory /var/www/itemCatalog/itemCatalog/>
                        Require all granted
                </Directory>
                Alias /static /var/www/itemCatalog/itemCatalog/static
                <Directory /var/www/itemCatalog/itemCatalog/static/>
                        Require all granted
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
* The `ServerName` is the static IP of the Amazon Lightsail instance.
* The `WSQIDaemonProcess` entry points the server to the virtualenv created for the Item Catalog project. This will ensure that the correct packages are used.

It was required to configure the server to prevent access to the `.git` directory by web clients by adding the following to the `/etc/apache2/apache2.conf` file:

```xml
<Directorymatch "^/.*/\.git/">
        Require all denied
</Directorymatch>
```
After the server is configured, enable the wsgi mod using `sudo a2enmod wsgi` and restart the server using `sudo service apache2 restart`.

### Database Creation

To set up the database, the following steps were followed (Note that postgres needs to be installed before following these steps):

* Login as the postgres user using `sudo su -l postgres`
* Create the `catalog` user and force password entry using `createuser catalog -P`
* Create a database called `catalogDatabaseWithUsers` using `createdb catalogDatabaseWithUsers --owner catalog`

SQLite was used for the Item Catalog project. This had to change to use Postgresql. To do this, the 
```python
engine = create_engine('sqlite:///catalogDatabaseWithUsers')
```
command was changed to 
```python
engine = create_engine('postgresql://catalog:secret@localhost/catalogDatabaseWithUsers')
```
where `catalog` is the name of the user and `secret` is the passphrase.

After the database was created, the `database.py` file in the project directory was run using `python3 database.py`. This was to set up the tables inside the database.  

### Other small changes made
* The `application.py` file in the project directory was changed to `__init__.py`
* The path to the `client_secrets.json` file was changed in the `__init__.py` to contain an absolute path:
  ```python
  oauth_flow = flow_from_clientsecrets('/var/www/itemCatalog/itemCatalog/client_secrets.json', scope='')
  ```
* A reverse DNS lookup website were used to find the Domain name of the static IP. This domain name was then used as the Authorized Javascript origins in the Google Developers console for the client_secrets.json file. The link to the site is  https://mxtoolbox.com/

## Third party resources used
* https://mxtoolbox.com/ for reverse domain name lookup
* DigitalOcean on how to configure the a flask app for a server: https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
* Extensive use of the Udacity forums. That was really helpful
* The Flask mega tutorial for additional help on configuring the Flask apllication https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world
* The mod_wsgi user guide on how to use Python virtual environments with mod_wsgi. This helped the most with getting the server to look in the virtual environment for packages http://modwsgi.readthedocs.io/en/develop/user-guides/virtual-environments.html


