# Linux Server Configuration
 The Linux server configuration is the sixth project to Udacity Fullstack
 Nanodegree and take a baseline installation of a Linux server to host a web
 app securely.

### Server Details
- IP address: 18.222.184.111
- Accessible SSH port: 2200
- Application URL: http://18.222.184.111

### Pre-requisites

- Apache
- PostgreSQL
- Python
- Flask


## Configuration
### 1. Connect using your own SSH client
Go to Amazon Lighsail panel, click on `networking` and configure te firewall

    # Add 2200 and 123 ports to the firewall on Amazon Lightsail panel.

Click on `connect` to get the private key

    # Download the private key from Amazon Lightsail panel.

Move the file to the `~/.ssh/` folder

    $ mv Downloads/LightsailDefaultPrivateKey-us-east-2.pem .ssh/udacity_private_key.pem

Open the terminal and type to configure permissions

    $ chmod 600 ~/.ssh/udacity_private_key.pem

Now you can connect the server using:

    $ ssh -i ~/.ssh/udacity_private_key.pem ubuntu@18.222.184.111


### 2. Set up new User
##### 2.a Create a new user named `grader`

    $ sudo adduser grader  

##### 2.b Give `grader` permission to `sudo`
create file `sudo nano /etc/sudoers.d/grader` with the content

    grader ALL=(ALL) NOPASSWD:ALL

##### 2.c Create an SSH key pair for `grader` using the `ssh-keygen` tool.

Generate the keypair on the local machine and save the private key in `~/.ssh`.

    $ ssh-keygen

Istalling the public key on server

    $ su - grader
    $ mkdir .ssh
    $ touch .ssh/authorized_keys
    $ nano .ssh/authorized_keys

Copy the public key saved in your local machine and paste to the
`.ssh/authorized_keys`.

    $ chmod 700 .ssh
    $ chmod 644 .ssh/authorized_keys


### 3. Secure the Server

##### 3.a Update all currently installed packages.

    $ sudo apt-get update
    $ sudo apgt-get upgrade
    $ sudo apt-get dist-upgrade

To allow automatic security updates, install and enable the
`unattended-upgrades` package, if not installed:

    $ sudo apt-get install unattended-upgrades
    $ sudo dpkg-reconfigure --priority=low unattended-upgrades

##### 3.b Change the SSH port from `22 to 2200`

Change line from Port 22 to Port 2200 in `sshd_config`

    $ sudo nano /etc/ssh/sshd_config
      # port 2200

Restart ssh service

    $ sudo service ssh restart

Now you can use ssh to login with the
`ssh grader@18.222.184.111 -p 2200 -i ~/.ssh/no_profit_key`

##### 3.c Configure the Uncomplicated Firewall (UFW)
Only allow incoming connections for `SSH (port 2200)`, `HTTP (port 80)`,
and `NTP (port 123)`.

```
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 2200/tcp
$ sudo ufw allow www
$ sudo ufw allow ntp
$ sudo ufw enable
```

##### 3.d Disable root login
Edit the `set PermitRootLogin` to `no`

```
$ sudo nano /etc/ssh/sshd_config
  # PermitRootLogin no
```

Restart ssh service

    $ sudo service ssh restart


### 4. Prepare to Deploy
##### 4.a Configure the local timezone UTC.

    $ sudo timedatectl set-timezone UTC

##### 4.b Install and configure Apache to serve a Python mod_wsgi application.

```
$ sudo apt-get install apache2
$ sudo apt-get install libapache2-mod-wsgi-py3
$ sudo apache2ctl restart
```

##### 4.c Install and configure PostgreSQL.
1. Install PostgreSQL `sudo apt-get install postgresql postgresql-contrib`
2. Check if no remote connections are allowed `sudo nano /etc/postgresql/9.5/main/pg_hba.conf`
3. Login to the psql as `postgres` default user.

       $ sudo su - postgres
       $ psql


4. Create a User and Database named catalog and associated them

  - Create a new User named *catalog*:

        postgres=# CREATE USER catalog WITH PASSWORD 'password';

   - Create a new DB named *catalog*:

         postgres=# CREATE DATABASE catalog WITH OWNER catalog;

    - Connect to the database *catalog* :

          postgres=# \c catalog

    - Revoke all rights:

          postgres=# REVOKE ALL ON SCHEMA public FROM public;

    - Lock down the permissions only to user *catalog*:

          postgres=# GRANT ALL ON SCHEMA public TO catalog;

    - Log out from PostgreSQL:

          postgres=# \q.

    - Then return to the *grader* user:

          $ exit


### 5. Install git and deploy
##### 5.a Install git and App Dependencies

```    
$ sudo apt-get install git
$ sudo apt-get install python3-pip
$ sudo pip3 install flask packaging oauth2client redis passlib flask-httpauth
$ sudo pip3 install sqlalchemy flask-sqlalchemy psycopg2-binary bleach requests
```

### 5.b Clone the udacity catalog project repository
 - Create a NoProfitApp diretory:

        $ sudo mkdir /var/www/NoProfitApp

- Change the owner of the NoProfitApp

        $ sudo chown -R grader:grader NoProfitApp

- Clone the No-Profit-Catalog deployment branch from github:

        $ sudo git clone -b deployment https://github.com/Pedroknoll/No-Profit-Catalog.git

- Make a noProfit.wsgi file to serve the application over the mod_wsgi.

      $ touch noProfit.wsgi && nano noProfit.wsgi

- Paste the content:

      #!/usr/bin/python
      import sys
      import logging
      logging.basicConfig(stream=sys.stderr)
      sys.path.insert(0, "/var/www/NoProfitApp/")

      from NoProfitCatalog import app as application
      application.secret_key = 'super_secret_key'

- Run the database setup

      $ sudo apt-get -qqy install postgresql python-psycopg2
      $ sudo python models.py
      $ sudo python populates_database.py

### 5.d Configure and Enable a New Virtual Host
Create a noProfitApp.conf file

    sudo nano /etc/apache2/sites-available/noProfitApp.conf

Add to the file the following code:

```sh
<VirtualHost *:80>
	ServerName 18.222.184.111
	ServerAdmin p.giacometo@gmail.com
	WSGIScriptAlias / /var/www/NoProfitApp/noProfit.wsgi
	<Directory /var/www/NoProfitApp/NoProfitCatalog/>
		Order allow,deny
		Allow from all
	</Directory>
	Alias /static /var/www/NoProfitApp/NoProfitCatalog/static
	<Directory /var/www/NoProfitApp/NoProfitCatalog/static/>
		Order allow,deny
		Allow from all
	</Directory>
	ErrorLog ${APACHE_LOG_DIR}/error.log
	LogLevel warn
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Enable the virtual host with:

    sudo a2ensite noProfitApp.conf

Disable the default apache site

    sudo a2dissite 000-default.conf

Restart Apache with the following command to apply the changes:

    sudo service apache2 restart


### 6. Third-Part
- [postgres on ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-18-04)
- [flask deploying](http://flask.pocoo.org/docs/1.0/deploying/mod_wsgi/)
- [flask deploying on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
- [configure linux webservers](https://br.udacity.com/course/configuring-linux-web-servers--ud299)
