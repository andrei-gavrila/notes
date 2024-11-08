# Install the AGACHE server

This guide is designed for a Raspberry Pi 5 with an additional WiFi adapter running Debian 64bit bookworm.

## Use the built in WiFi adapter for the WiFi client connection

```bash
sudo nmcli connection modify preconfigured ifname wlan0
```

Reboot the Raspberry.

## Configure the additional WiFi adapter as Access Point

```bash
sudo nmcli connection add con-name ap ifname wlan1 type wifi ssid "Raspberry Pi AP 01"
sudo nmcli connection modify ap wifi-sec.key-mgmt wpa-psk
sudo nmcli connection modify ap wifi-sec.psk "<PASSWORD>"
sudo nmcli connection modify ap 802-11-wireless.mode ap 802-11-wireless.band bg ipv4.method shared
sudo nmcli connection modify ap wifi-sec.pmf disable
```

Reboot the Raspberry.

## Install MariaDB, Apache 2 and PHP

```bash
# Update the packages list
sudo apt update

# Install MariaDB server
sudo apt install mariadb-server
# Secure the installation (simply follow the default values, create a new root password)
sudo mysql_secure_installation

# Test the connection
mysql -u root -p

# Install Apache2
sudo apt install apache2

# Install PHP with MySQL support
sudo apt install php php-mysql
```

## Create the MariaDB database and user

```bash
mysql -u root -p
```

and execute

```sql
CREATE DATABASE agache; CREATE USER 'agache'@'localhost' IDENTIFIED BY '<PASSWORD>'; GRANT ALL PRIVILEGES ON agache.* TO 'agache'@'localhost'; FLUSH PRIVILEGES;
```

## Deploy the application MariaDB tables

```bash
mysql -u agache -p
```

and execute

```sql
USE agache; SOURCE agache.sql;
```

Test the installation by creating a **index.php** file in **/var/www/html/**

```php
<?php

phpinfo();

$link = mysqli_connect('localhost', 'agache', '<PASSWORD>', 'agache');
$result = mysqli_query($link, "SELECT * FROM rfid_keys");

while ($line = mysqli_fetch_array($result, MYSQLI_ASSOC)) {
    var_dump($line);
}

mysqli_free_result($result);

mysqli_close($link);
```

and point the browser to http://agache.local/index.php

## References

Use a Raspberry Pi as a WiFi Hotspot: https://trainedmonkey.com/2024/05/20/using_a_raspberry_pi_as_a_wifi_hotspot

Disable SAE and force the use of WPA2 Personal/SHA-256/AES: https://superuser.com/questions/1771752/m1-macbook-pro-cant-join-linux-hotspot-when-other-devices-can
