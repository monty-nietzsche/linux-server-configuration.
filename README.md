# Linux Server Configuration (Ubunto 18.4 Server)

## Project Overview:

The purpose of this project is to deploy the application *BOOKBUDDY* ([github repository](https://github.com/monty-nietzsche/book-buddy)) to an Ubuntu server. The steps to deploy this application on the server involves creating users, securing the server from attacks (firewall, changing the TCP port for SSH), installing and configuring a database server (Postgresql) to finally upload the application and tweak it lightly so as to function in the setup environment. 

#### Technical details:
```
Platform : Amazon Web Service (LightSail)
Server : Ubuntu Server 18.04 LTS (HVM), SSD Volume Type
SSH Port: 2200
Public IP address: 18.223.166.231
Public DNS: ec2-18-223-166-231.us-east-2.compute.amazonaws.com
url: http://18.223.166.231
```


## Deployment Steps:
The deployment of the application BookBuddy on an Ubuntu is done in six steps:

[Setting up the development environment](#1-setting-up-the-development-environment)

[Securing the server from illegitimate access](#2-securing-the-server-from-illegitimate-access)

[Creating a user with sudo rights](#3-creating-a-user-grader-with-sudo-rights)

[Setting up the database server and creating a database](#4-setting-up-the-database-server-and-creating-a-database)

[Installing required packages and uploading the app to the server](#5-installing-required-packages-and-uploading-the-app-to-the-server)

[Configuring Apache and launching BookBuddy](#6-configuring-apache-and-launching-bookbuddy)


##  1. SETTING UP THE DEVELOPMENT ENVIRONMENT

#### 1.  Launch an instance on [Amazon LightSail](https://lightsail.aws.amazon.com)

* Launch a new instance with **Ubuntu Server 18.04 LTS (HVM)** as a server
* Download private key (save it e.g. as `udacity.pem`) and write down your public IP address.
* Move the private key file into the folder ~/.ssh:  
  `$ mv ~/Downloads/udacity.pem ~/.ssh/`
* Access the instance through SSH:  
  `$ ssh -i ~/.ssh/udacity.pem ubuntu@PUPLIC-IP-ADDRESS`

#### 2. Configure the security group inbound rules

* Make sure to configure the security groups of the instance you launch to allow access to different ports on the server instance. Under the menu [Networking and Security], choose [Security Groups], click on the security group corresponding to your instance. On the bottom half of the page, click on the tab [inbound] and then [Edit]. Add the following rules:

    | Type          | Protocol      | Port Range  | Source |
    | ------------- |:-------------:| -----:|---:|
    | HTTP      | TCP | 80 | 0.0.0.0/0, ::/0|
    | Custom TCP Rule      | TCP      |   2200 |0.0.0.0/0|

#### 3. Update and upgrade the server software

* Updating and upgrading the current packages
```
    $ sudo apt-get update
    $ sudo apt-get upgrade
```

* Automate unattended-upgrades process updates:
  ```
  $ sudo apt-get install unattended-upgrades
  $ sudo dpkg-reconfigure unattended-upgrades
  ```

#### 4. Set the local timezone to UTC
```
$ sudo timedatectl set-timezone UTC
```

## 2. SECURING THE SERVER FROM ILLEGITIMATE ACCESS

#### 1. Changing the default SSH port and disable login for the `root` user

* Access the SSH configuration file:
   ```
   $ sudo nano /etc/ssh/sshd_config
   ```
* Change the line `#Port 22` to `Port 2200` (! delete '#');  the line `#PermitRootLogin prohibit-password` to `PermitRootLogin no`, save and exit.

* Restart the ssh service:
   ```
   $ sudo service sshd restart
   ```

#### 2 . Configuring the Uncomplicated Firewall (UFW)

* We configure the firewall to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123) before enabling it.

    ```
     $ sudo ufw default deny incoming
     $ sudo ufw default allow outgoing
     $ sudo ufw allow 2200/tcp
     $ sudo ufw allow www
     $ sudo ufw allow ntp
    ```
    ```
     $ sudo ufw enable
    ```

## 3. CREATING A USER GRADER WITH SUDO RIGHTS

#### 1. Creating a new user *grader* and giving `sudo` access

* Create a new user:  
    ```
    $ sudo adduser grader
    ```

* Open the sudo configuration to give grader `sudo` access:  
    ```
    $ sudo visudo
    ```

* Add the following line below `root ALL...`:  
    `grader ALL=(ALL:ALL) ALL`

#### 2. Creating an SSH key for **grader** and storing the public key in the server

* **On your local machine**, create an encryotion key using:
    ```
    $ ssh-keygen -f ~/.ssh/grader_key.rsa
    ```

* On your SSH server session, login as `grader` with the following command:
    ```
    $ sudo su - grader
    ```

* Make a new directory `.ssh` and create a a file `authorized_keys` in it:
    ```
    $ sudo mkdir .ssh
    $ sudo nano .ssh/authorized_keys
    ```

* Copy the content of the public key file [`~/.ssh/grader_key.rsa.pub`]  that you have just created,  paste it into the `authorized_keys` file,  save and exit.

* Change the permisions of the `.ssh` directory and the `authorized_keys` file:
    ```
    $ chmod 700 .ssh
    $ sudo chmod 644 .ssh/authorized_keys
    ```

* Change owner of the .ssh folder (and it contents) from ubuntu to grader
    ```
    $ sudo chown -R grader:grader /home/grader/.ssh
    ```

#### 3. Logging in to server as **grader**

* From **your local machine**, run the following command:
    ```
    $ ssh -i ~/.ssh/grader_key.rsa grader@PUBLIC IP ADDRESS -p 2200
    ```

* You should now be connected to the server through SSH at port 2200.

## 4. SETTING UP THE DATABASE SERVER AND CREATING A DATABASE

#### 1. Install and configure PostgreSQL

* Install Postgresql-10:

    ```
    $ sudo apt-get install postgresql-10 postgresql-contrib
    ```
* Edit the configuration file for PostgreSQL `postgresql.conf`
    ```
    $ sudo nano/etc/postgresql/10/main/postgresql.conf
    ```
* Change the line `listen_addresses = 'localhost'` to `listen_addresses = '*'`

* Restart the database engine:
    ```
    sudo service postgresql restart
    ```
#### 2. Create the database for the BookBuddy App

* Login as *postgres* user (Default user), and get into `psql` shell:
* 
    ```
    $ sudo su - postgres
    $ psql
    ```
* Create a new user named **admin**: 

    ```
    # CREATE USER admin WITH PASSWORD 'adminpassword';
    ```
    
* Create a new database named *bookmarket*:
    ```
    # CREATE DATABASE bookmarket WITH OWNER admin;`
    ```
* Connect to the *bookmarket* database: 
    ```
    \c bookmarket
    ```
* Revoke all rights: 
    ```
    # REVOKE ALL ON SCHEMA public FROM public;
    ```
* Grant all permissions to **admin** user: 
    ```
    # GRANT ALL ON SCHEMA public TO admin;
    ```
* Leave `psql` shell and switch back to *grader* user
    ```
    # \q
    $ exit
    ```

## 5. INSTALLING REQUIRED PACKAGES AND UPLOADING THE APP TO THE SERVER   

#### 1. Install required ubuntu and python packages 

* Install required ubuntu packages by  the following command:

    ```
    $ sudo apt-get install apache2 libapache2-mod-wsgi git python-pip libpq-dev python-dev
    ```

* Configure `git`:

    ```
    $ git config --global user.name "<Your-Full-Name>"
    $ git config --global user.email "<your-email-address>"
    ```

* Install python packages a.o. flask and sqlalchemy
    ```
    $ sudo pip install flask
    $ sudo pip install httplib2 oauth2client sqlalchemy psycopg2 requests
    ```
#### 2. Setting up the bookbuddy app on  the server 

* Create a directory named *bookbuddy* in **/var/www**:

    ```
    $ sudo mkdir /var/www/bookbuddy
    ```
* Within the directory `bookbuddy` just created, create another directory with the same name:
    ```
        $ cd /var/www/bookbuddy
        $ sudo mkdir bookbuddy
    ```
* Clone the git project *book-buddy* to the `/var/www/bookbuddy/bookbuddy/`  directory:

    ```
    $ sudo git clone https://github.com/monty-nietzsche/book-buddy.git  /var/www/bookbuddy/bookbuddy
    ```

* Create a file *bookbuddy.wsgi* file to handle requests with **mod_wsgi** and edit it:

    ```
    $ cd /var/www/bookbuddy
    $ sudo nano bookbuddy.wsgi
    ```

* Paste the following content, save and exit:

    ```
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, "/var/www/bookbuddy/")
    
    from bookbuddy.app import app as application
    application.secret_key = 'super_secret_key'
    ```

* In  the  files `app.py`, `database_setup.py` and `database_fill.py`, replace the line ` engine =  ...`  with:

    ```
    engine = create_engine('postgresql://admin:adminpassword@localhost/bookmarket')
    ```

* In `app.py` file,  replace the line `app_path = ""`:

    ```
    app_path = '/var/www/bookbuddy/bookbuddy'
    ```

* Fill the database with sample data:

    ```
    $ python database_fill.py
    ```

## 6. CONFIGURING APACHE AND LAUNCHING BOOKBUDDY

*  Open the  VirtualHost file of Apache by using the following command:

    ```
    $ sudo nano /etc/apache2/sites-available/000-default.conf
    ```
* Erase all its content and paste the following content:

    ```
    <VirtualHost *:80>
                    ServerName < Your-Server-Name ending in .compute.amazonaws.com>
                    ServerAdmin <Email of the server admin>
                    WSGIScriptAlias / /var/www/bookbuddy/bookbuddy.wsgi
                    <Directory /var/www/bookbuddy/bookbuddy>
                            Order allow,deny
                            Allow from all
                    </Directory>
                    Alias /static /var/www/bookbuddy/bookbuddy/static
                    <Directory /var/www/bookbuddy/bookbuddy/static/>
                            Order allow,deny
                            Allow from all
                    </Directory>
                    ErrorLog ${APACHE_LOG_DIR}/error.log
                    LogLevel warn
                    CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
    ```

* Restart the Apache service  to launch the app

    ```
    $ sudo apache2ctl restart
    ```

* Access BookBuddy  in your default browser at this url: http://18.223.166.231


##  References

* Stack Overflow
* [JebinPhilipose Repository](https://github.com/jebinphilipose/Linux-Server-Configuration/blob/master/README.md)
* DigitalOcean Community
