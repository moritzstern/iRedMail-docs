# Integrate netdata monitor (on Linux server)

[TOC]

!!! attention

    This tutorial is tested on CentOS 7, Debian 9, Ubuntu 16.04.
    For FreeBSD, please check this tutorial instead:
    [Integrate netdata on FreeBSD](./integration.netdata.freebsd.html).

## What's netdata

netdata (<http://my-netdata.io>) is a "Simple. Effective. Awesome!" monitor
which can monitor almost everyting on your Linux/FreeBSD system. You can visit
its website to check online demo.

We will show you how to install and configure netdata on iRedMail server
(Linux) to monitor mail service related softwares.

## Install packages required by netdata

netdata requires some tools to get stastics data from other softwares, let's
install it first.

* On RHEL/CentOS:

```
yum install curl libmnl libuuid lm_sensors nc PyYAML zlib iproute MySQL-python python-psycopg2
```

* On Debian/Ubuntu:

```
apt-get install zlib1g libuuid1 libmnl0 curl lm-sensors iproute netcat python-mysqldb python-psycopg2
```

## Install netdata

* Download the latest netdata from its github project page, and upload to
  iRedMail server: <https://github.com/firehol/netdata/releases>

    We use version `1.9.0` for example in this tutorial, the package we download
    is: <https://github.com/firehol/netdata/releases/download/v1.9.0/netdata-latest.gz.run>

    We assume you upload the package to `/root/netdata-latest.gz.run`.

* Install netdata:

```
cd /root/
chmod +x netdata-latest.gz.run
./netdata-latest.gz.run --accept
```

netdata installs its files under `/opt/netdata/` by default, let's create
symbol link of the configuration and log directories:

```
ln -s /opt/netdata/etc/netdata /etc/netdata
ln -s /opt/netdata/var/log/netdata /var/log/netdata
```

netdata will create required systemd script for service control, also logrotate
config file, so there's not much we need to do after the package installation.

## Configure netdata

Main config file of netdata is `/etc/netdata/netdata.conf`, it contains many
parameters with detailed comments. Here's the
[config file](https://bitbucket.org/zhb/iredmail/src/default/iRedMail/samples/netdata/netdata.conf)
used by iRedMail:

* It binds to address `127.0.0.1` and port `19999` by default. Since it doesn't
  have ACL control, we will run netdata behind Nginx to get ACL control done in
  Nginx.

```
[registry]
    enabled = no

[global]
    bind to = 127.0.0.1
    run as user = netdata
    default port = 19999
    update every = 3

[plugin:proc]
    # Disable IPVS check since iRedMail doesn't use ipvs by default
    /proc/net/ip_vs/stats = no

    # inbound packets dropped
    /proc/net/dev = no
```

netdata ships a lot modular config files to gather information of softwares
running on the server, they have very good default settings and most config
files don't need your attention at all, including:

* System resources (CPU, RAM, disk I/O, etc)
* Nginx log file monitoring
* Fail2ban jails
* Memcached
* ...

But some applications do require extra settings, we will cover them below.

### Monitor Nginx and php-fpm

We need to enable `stub_status` in Nginx to get detailed server info, also
update php-fpm config file to enable similar feature.

* Create Nginx config snippet `/etc/nginx/templates/stub_status.tmpl` with
  content below:

```
location = /stub_status {
    stub_status on;
    access_log off;
    allow 127.0.0.1;
    deny all;
}

location = /status {
    include fastcgi_params;
    fastcgi_pass php_workers;
    fastcgi_param SCRIPT_FILENAME $fastcgi_script_name;
    access_log off;
    allow 127.0.0.1;
    deny all;
}
```

* Update default virtual host config file `/etc/nginx/sites-enabled/00-default.conf`,
  include new snippet config file `stub_status.tmpl` after the
  `redirect_to_https.tmpl` line like below:

```
server {
    ...
    include /etc/nginx/templates/redirect_to_https.tmpl;
    include /etc/nginx/templates/stub_status.tmpl;      # <- add this line
    ...
}
```

* Update php-fpm pool config file `www.conf`, enable parameter `pm.status_path`
  like below:
    * On RHEL/CentOS, it's `/etc/php-fpm.d/www.conf`
    * On Debian, it's `/etc/php5/fpm/pool.d/www.conf`
    * On Ubuntu, it's `/etc/php/7.0/fpm/pool.d/www.conf` (note: php version number may be different on your server)
    * On FreeBSD, it's `/usr/local/etc/php-fpm.d/www.conf`
    * On OpenBSD, it's `/etc/php-fpm.conf`

```
pm.status_path = /status
```

* Restart both php-fpm and Nginx service.

### Monitor Dovecot

We need to enable statistics module in Dovecot.

* Please open Dovecot config file:
    * on Linux and OpenBSD, its `/etc/dovecot/dovecot.conf`.
    * on FreeBSD, it's `/usr/local/etc/dovecot/dovecot.conf`.

* Append plugin `stats` in global parameter `mail_plugins`, and `imap_stats`
  for imap protocol:

```
mail_plugins = ... stats

protocol imap {
    mail_plugins = ... imap_stats
```

* Append settings below in Dovecot config file:

```
plugin {
    # how often to session statistics (must be set)
    stats_refresh = 30 secs
    # track per-IMAP command statistics (optional)
    stats_track_cmds = yes
}

service stats {
    fifo_listener stats-mail {
        user = vmail
        mode = 0644
    }

    inet_listener {
        address = 127.0.0.1
        port = 24242
    }
}
```

* Restart Dovecot service.

### Monitor MySQL/MariaDB server

netdata requires a SQL user (we use `netdata` here) with privilege `USAGE` to
gather MySQL server information.

* Create the SQL user with a strong password (please replace `<password>` in
  command below by the real (and strong) password).

```
# mysql -u root
sql> GRANT USAGE ON *.* TO netdata@localhost IDENTIFIED BY '<password>';
sql> FLUSH PRIVILEGES;
```

* Create file `/etc/netdata/python.d/mysql.conf` with content below.

    !!! attention
    
        * This file already exists, feel free to remove all content in this file
          and copy content below as its new content.
        * Please replace `<password>` below by the real password.

```
tcp:
    name: 'local'
    host: '127.0.0.1'
    port: '3306'
    user: 'netdata'
    pass: '<password>'
```

### Monitor PostgreSQL server

netdata requires a SQL user (we use `netdata` here) to gather PostgreSQL server
information.

* Create the SQL user with a strong password (please replace `<password>` in
  command below by the real (and strong) password).

```
# su - postgres
$ psql
sql> CREATE USER netdata WITH ENCRYPTED PASSWORD '<password>' NOSUPERUSER NOCREATEDB NOCREATEROLE;
```

* Create file `/etc/netdata/python.d/mysql.conf` with content below.

    !!! attention
    
        * This file already exists, feel free to remove all content in this file
          and copy content below as its new content.
        * Please replace `<password>` below by the real password.

```
socket:
    name     : 'local'
    user     : 'netdata'
    password : '<password>'
    database : 'postgres'
```

## Configure Nginx to forward requests to netdata

## System tuning

To get better performance, netdata requires few sysctl settings. Please add
lines below in `/etc/sysctl.conf`:

```
vm.dirty_expire_centisecs=60000
vm.dirty_background_ratio=80
vm.dirty_ratio=90
```

Also increase max open files limit. 

```
mkdir -p /etc/systemd/system/netdata.service.d
```

Create file `/etc/systemd/system/netdata.service.d/limits.conf`:

```
[Service]
LimitNOFILE=30000
```

Reload systemd daemon:

```
systemctl daemon-reload
```