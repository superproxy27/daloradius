#!/bin/sh

sudo apt-get update -y
sudo apt-get upgrade -y

#Install Apache
sudo apt-get install apache2

#Install PHP
sudo apt-get install php libapache2-mod-php php-gd php-common php-mail php-mail-mime php-mysql php-pear php-db php-mbstring php-xml php-curl

#Install MariaDB
sudo apt -y install mariadb-server mariadb-client


#Configure Database for FreeRADIUS
sudo mysql_secure_installation

Enter current password for root (enter for none): Just press the Enter
Set root password? [Y/n]: Y
New password: Enter password
Re-enter new password: Repeat password
Remove anonymous users? [Y/n]: Y
Disallow root login remotely? [Y/n]: Y
Remove test database and access to it? [Y/n]:  Y
Reload privilege tables now? [Y/n]:  Y

sudo mysql -u root -p
CREATE DATABASE radius CHARACTER SET UTF8 COLLATE UTF8_BIN;
CREATE USER 'radius'@'%' IDENTIFIED BY 'G@s%w&rJ';
GRANT ALL PRIVILEGES ON radius.* TO 'radius'@'%';
QUIT;


#Install FreeRADIUS
sudo apt policy freeradius
sudo apt -y install freeradius freeradius-mysql freeradius-utils

#Once installed, import the freeradius MySQL database schema with the following command:
sudo su -
sudo mysql -u root -p radius < /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql
sudo  mysql -u root -p -e "use radius;show tables;"

#Create a soft link for sql module under
sudo ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/

#Run the following commands to create the Certificate Authority (CA) keys:
cd /etc/ssl/certs/
sudo openssl genrsa 2048 > ca-key.pem
sudo openssl req -sha1 -new -x509 -nodes -days 3650 -key ca-key.pem > ca-cert.pem

#Make the following changes as per your database:
sudo nano /etc/freeradius/3.0/mods-enabled/sql

sql {
driver = "rlm_sql_mysql"
dialect = "mysql"

        mysql {
                # If any of the files below are set, TLS encryption is enabled
                tls {
                        ca_file = "/etc/ssl/certs/ca-cert.pem"
                        ca_path = "/etc/ssl/certs/"
                        #certificate_file = "/etc/ssl/certs/private/client.crt"
                        #private_key_file = "/etc/ssl/certs/private/client.key"
                        cipher = "DHE-RSA-AES256-SHA:AES128-SHA"

                        tls_required = no
                        tls_check_cert = no
                        tls_check_cert_cn = no
                }

# Connection info:
server = "localhost"
port = 3306
login = "radius"
password = "G@s%w&rJ"

# Database table configuration for everything except Oracle
radius_db = "radius"
}

read_clients = yes
client_table = "nas"

#Save and close

#Set the proper permission
sudo chgrp -h freerad /etc/freeradius/3.0/mods-available/sql
sudo chown -R freerad:freerad /etc/freeradius/3.0/mods-enabled/sql

sudo systemctl restart freeradius
sudo systemctl status freeradius

#Install daloRADIUS
sudo apt -y install wget unzip
wget https://github.com/lirantal/daloradius/archive/master.zip
unzip master.zip
mv daloradius-master daloradius
cd daloradius


#Configuring daloradius
sudo mysql -u root -p radius < contrib/db/fr2-mysql-daloradius-and-freeradius.sql 
sudo mysql -u root -p radius < contrib/db/mysql-daloradius.sql

cd ..
sudo mv daloradius /var/www/html/

sudo chown -R www-data:www-data /var/www/html/daloradius/
sudo cp /var/www/html/daloradius/library/daloradius.conf.php.sample /var/www/html/daloradius/library/daloradius.conf.php
sudo chmod 664 /var/www/html/daloradius/library/daloradius.conf.php

sudo nano /var/www/html/daloradius/library/daloradius.conf.php

#Make the following changes that match your database:
sudo nano /var/www/html/daloradius/library/daloradius.conf.php

$configValues['CONFIG_DB_HOST'] = 'localhost';
$configValues['CONFIG_DB_PORT'] = '3306';
$configValues['CONFIG_DB_USER'] = 'radius';
$configValues['CONFIG_DB_PASS'] = 'G@s%w&rJ';
$configValues['CONFIG_DB_NAME'] = 'radius';

#Import the daloRAIUS MySQL tables to the FreeRADIUS database
cd /var/www/html/daloradius/
sudo mysql -u root -p radius < /var/www/html/daloradius/contrib/db/fr2-mysql-daloradius-and-freeradius.sql
sudo mysql -u root -p radius < /var/www/html/daloradius/contrib/db/mysql-daloradius.sql

sudo chown -R www-data:www-data /var/www/html/daloradius/

sudo systemctl restart freeradius.service apache2

#Visit: 
http://localhost/daloradius/login.php

Username: administrator
Password: radius
