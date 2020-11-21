# Postgres DBA with Vagrant

Demo how to configure postgres cluster, integrate backup tools from scratch using Vagrant VM

```
Primary's IP address: 192.168.34.10 

Replica's IP address: 192.168.34.20

Repository's IP address: 192.168.34.30
```

# Table of content

- [Postgres DBA with Vagrant](#postgres-dba-with-vagrant)
- [Table of content](#table-of-content)
- [Setup via script](#setup-via-script)
- [Manual Setup](#manual-setup)
  - [Setup environment](#setup-environment)
  - [General configuration](#general-configuration)
  - [Setup Passwordless SSH](#setup-passwordless-ssh)
    - [Generate RSA key pair](#generate-rsa-key-pair)
    - [Share public key between servers](#share-public-key-between-servers)
    - [Test connections](#test-connections)
  - [Configure Pgbackrest (Dedicated host - repository server)](#configure-pgbackrest-dedicated-host---repository-server)
    - [Repository server](#repository-server)
    - [Primary server](#primary-server)
    - [Create test stanza](#create-test-stanza)
  - [Perform a Backup](#perform-a-backup)

# Setup via script

[TODO]

# Manual Setup

## Setup environment

```
vagrant up
```

## General configuration

Installation postgres (version 11) & pgbackrest on both servers (primary, replica)

- Add repository


    ```
        # add the repository
        sudo tee /etc/apt/sources.list.d/pgdg.list <<END
        deb http://apt.postgresql.org/pub/repos/apt/ bionic-pgdg main
        END

        # get the signing key and import it
        wget https://www.postgresql.org/media/keys/ACCC4CF8.asc
        sudo apt-key add ACCC4CF8.asc

        # fetch the metadata from the new repo
        sudo apt-get update
    ```

- Install from repository

    ```
    sudo apt-get -y install postgresql-11 postgresql-contrib-11 pgbackrest
    ```

## Setup Passwordless SSH

### Generate RSA key pair

- On primary server
    ```
    sudo -u postgres mkdir -m 750 -p /var/lib/postgresql/.ssh

    sudo -u postgres ssh-keygen -f /var/lib/postgresql/.ssh/id_rsa \
        -t rsa -b 4096 -N ""
    ```

- On repository server

    ```
    sudo adduser --disabled-password --gecos "" pgbackrest

    sudo -u pgbackrest mkdir -m 750 /home/pgbackrest/.ssh

    sudo -u pgbackrest ssh-keygen -f /home/pgbackrest/.ssh/id_rsa \
        -t rsa -b 4096 -N ""
    ```

### Share public key between servers

- On primary server

    Create file repository.pub & paste public generated key on repository server into it

    ```
    sudo -iu postgres vim repository.pub
    ```

    Save as authorized key
    ```
    sudo -iu postgres cat repository.pub >> /var/lib/postgresql/.ssh/authorized_keys
    ```

- On repository server

    Create file primary.pub & paste public generated key on primary server into it

    ```
    sudo -iu pgbackrest vim primary.pub
    ```

    Save as authorized key
    ```
    sudo -iu pgbackrest cat primary.pub >> /home/pgbackrest/.ssh/authorized_keys
    ```    

### Test connections

- On repository server

    ```
    sudo -u pgbackrest ssh postgres@192.168.34.10
    ```

- On primary server

    ```
    sudo -u postgres ssh pgbackrest@192.168.34.30
    ```

## Configure Pgbackrest (Dedicated host - repository server)

### Repository server

```
sudo mkdir -p -m 770 /var/log/pgbackrest
sudo chown pgbackrest:pgbackrest /var/log/pgbackrest
sudo chmod 640 /etc/pgbackrest.conf
sudo chown pgbackrest:pgbackrest /etc/pgbackrest.conf
sudo mkdir -p /var/lib/pgbackrest
sudo chmod 750 /var/lib/pgbackrest
sudo chown pgbackrest:pgbackrest /var/lib/pgbackrest
```

Edit pgbackrest.conf

```
sudo -iu pgbackrest vim /etc/pgbackrest.conf
```

Add following lines
```
[demo]
pg1-host=192.168.34.10
pg1-path=/var/lib/postgresql/11/main
pg1-host-user=postgres

[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
start-fast=y
log-level-console=info
log-level-file=debug
```

Restart postgres

```
sudo service postgresql restart
```

### Primary server

```
sudo mkdir -p /var/lib/pgbackrest
sudo chmod 0750 /var/lib/pgbackrest
sudo chown -R postgres:postgres /var/lib/pgbackrest
sudo vim /etc/postgresql/11/main/postgresql.conf
```

Edit pgbackrest.conf
```
sudo -iu postgres vim  /etc/pgbackrest.conf
```

Add following lines
```
[global]
repo1-host=192.168.34.30
repo1-host-user=pgbackrest
repo1-retention-full=2
log-level-console=info
log-level-file=debug
start-fast=y

[demo]
pg1-path=/var/lib/postgresql/11/main
```

Edit postgres config

```
sudo vim /etc/postgresql/11/main/postgresql.conf
```

Add following lines
```
listen_addresses = '*'
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
```

Restart postgres
```
sudo service postgresql restart
```

### Create test stanza

On repository server

```
sudo -u pgbackrest pgbackrest --stanza=demo stanza-create

-------Result should be---------------
2020-11-21 16:31:05.206 P00   INFO: stanza-create command begin 2.30: --log-level-console=info --log-level-file=debug --pg1-host=192.168.34.10 --pg1-host-user=postgres --pg1-path=/var/lib/postgresql/11/main --repo1-path=/var/lib/pgbackrest --stanza=demo
2020-11-21 16:31:06.830 P00   INFO: stanza-create command end: completed successfully (1625ms)
```

Check it on primary server

```
sudo -u postgres pgbackrest --stanza=demo check

-------Result should be---------------
2020-11-21 16:31:16.341 P00   INFO: check command begin 2.30: --log-level-console=info --log-level-file=debug --pg1-path=/var/lib/postgresql/11/main --repo1-host=192.168.34.30 --repo1-host-user=pgbackrest --stanza=demo
2020-11-21 16:31:19.751 P00   INFO: WAL segment 000000010000000000000001 successfully archived to '/var/lib/pgbackrest/archive/demo/11-1/0000000100000000/000000010000000000000001-fe6781b52809ee9f6b2bdd515a00075c9fc77b3e.gz'
2020-11-21 16:31:19.950 P00   INFO: check command end: completed successfully (3610ms)
```

Check again on repository server

```
sudo -u pgbackrest pgbackrest --stanza=demo check
-------Result should be---------------
2020-11-21 16:31:33.227 P00   INFO: check command begin 2.30: --log-level-console=info --log-level-file=debug --pg1-host=192.168.34.10 --pg1-host-user=postgres --pg1-path=/var/lib/postgresql/11/main --repo1-path=/var/lib/pgbackrest --stanza=demo
2020-11-21 16:31:35.694 P00   INFO: WAL segment 000000010000000000000002 successfully archived to '/var/lib/pgbackrest/archive/demo/11-1/0000000100000000/000000010000000000000002-61ad7b6c1a724a54b18f227e7e3cebe719e4f58c.gz'
2020-11-21 16:31:35.796 P00   INFO: check command end: completed successfully (2570ms)
```

## Perform a Backup

On repository server

```
sudo -u pgbackrest pgbackrest --stanza=demo backup
-------Result should be---------------
...
2020-11-21 17:14:52.895 P00   INFO: expire command begin 2.30: --log-level-console=info --log-level-file=debug --pg1-host=192.168.34.10 --pg1-host-user=postgres --repo1-path=/var/lib/pgbackrest --repo1-retention-full=2 --stanza=demo
2020-11-21 17:14:52.913 P00   INFO: expire command end: completed successfully (19ms)
```