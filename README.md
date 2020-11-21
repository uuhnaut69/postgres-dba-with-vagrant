# Postgres DBA with Vagrant

Demo how to configure postgres cluster, integrate backup tools from scratch using Vagrant VM

```
Master's IP address: 192.168.34.10 

Slave's IP address: 192.168.34.20
```

# Table of content

- [Postgres DBA with Vagrant](#postgres-dba-with-vagrant)
- [Table of content](#table-of-content)
- [Setup via script](#setup-via-script)
- [Manual Setup](#manual-setup)
  - [Master server configuration:](#master-server-configuration)
  - [Slave server configuration:](#slave-server-configuration)
  - [Test Master-Slave Mode](#test-master-slave-mode)
  - [Pgbackrest Configuration - Master server configuration](#pgbackrest-configuration---master-server-configuration)
    - [Config Pgbackrest](#config-pgbackrest)

# Setup via script

[TODO]

# Manual Setup

## Master server configuration:

Start VM master server:

```
cd master && vagrant up
```

Connect to master server:

```
vagrant ssh
```

Add repository

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

Install postgres, pgbackrest

```
sudo apt-get -y install postgresql-11 postgresql-contrib-11
```

Switch to postgres user

```
sudo -iu postgres
```

```
# Create replication user with password
postgres=# CREATE USER replication REPLICATION LOGIN CONNECTION LIMIT 1 ENCRYPTED PASSWORD '12345678Aa';
-------------------
CREATE ROLE


# Get list users
postgres=# \du
                                    List of roles
  Role name  |                         Attributes                         | Member of
-------------+------------------------------------------------------------+-----------
 postgres    | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 replication | Replication                                                | {}

# Leaving psql
postgres=# \q
```

Adjust system settings. Modify the following parameters in /etc/postgresql/11/main/postgresql.conf :

```
# vim configuration file 
postgres@ubuntu-bionic:~$ vim /etc/postgresql/11/main/postgresql.conf

# Adjust the following settings 
listen_addresses ='*' 
wal_level = replica 
max_wal_senders = 10 
wal_keep_segments = 64 
```

Modify /etc/postgresql/11/main/pg_hba.conf the Allow the Slave using replication user connected to the master server

```
# vim configuration file 
postgres@ubuntu-bionic:~$ vim /etc/postgresql/11/main/pg_hba.conf
# Add 192.168.34.20/32 at the end of the configuration file as slave ip 
host replication replication 192.168.34.20/32 md5

# Restart the postgresql service 
postgres@ubuntu-bionic:~$ exit;
 logout 

sudo service postgresql restart
```

## Slave server configuration:

Install postgres is the same master server's installation

Stop the postgresSQL service

```
sudo service postgresql stop
```
Switch to postgres user

```
sudo -iu postgres
```

Edit config file

```
vim /etc/postgresql/11/main/postgresql.conf

# Add flowing lines

listen_addresses ='*' 
wal_level = replica 
max_wal_senders = 10 
wal_keep_segments = 64 
hot_standby = on
```

Then delete the posgresql data on the slave

```
cd /var/lib/postgresql/11/main/

rm -rfv *

# Confirm whether the data is deleted 
ls -al
```

Use the pg_basebackup command to fetch the data on the master server to the slave

```
pg_basebackup -h 192.168.34.10 -U replication -p 5432 -D /var/lib/postgresql/11/main/  -Fp -Xs -P -R

>> Enter password of 'replication' user we created above
```

Quit postgres user and restart postgresql service
```
exit;

sudo service postgresql start
```

## Test Master-Slave Mode

Back to Master server cmd

Switch to postgres user
```
sudo -iu postgres
```

Enter psql query to check status replication
```
postgres=# SELECT * FROM pg_stat_replication;
```

- Result should be
```
 pid  | usesysid |   usename   | application_name |  client_addr  | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_lsn  | write_lsn | flush_lsn | replay_lsn | write_lag | flush_lag | replay_lag | sync_priority | sync_state
------+----------+-------------+------------------+---------------+-----------------+-------------+-------------------------------+--------------+-----------+-----------+-----------+-----------+------------+-----------+-----------+------------+---------------+------------
 5869 |    16384 | replication | walreceiver      | 192.168.34.20 |                 |       34560 | 2020-11-18 04:36:46.967368+00 |              | streaming | 0/3032AE0 | 0/3032AE0 | 0/3032AE0 | 0/3032AE0  |           |           |            |             0 | async
(1 row)
```

Test to add some data on master server

```
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

create table companies
(
    id           uuid primary key default uuid_generate_v4(),
    company_name varchar not null,
    address      varchar not null
);

insert into companies(company_name, address)
values ('Company A', 'Viet Nam'),
       ('Company B', 'Viet Nam'),
       ('Company C', 'Viet Nam');
```

Go to slave server

```
sudo -iu postgres

psql

postgres=# select * from public.companies;
                  id                  | company_name | address
--------------------------------------+--------------+----------
 9bb3d454-1b9c-4016-ae15-59cd9a4d81b6 | Company A    | Viet Nam
 e63c447e-48b6-482b-86ce-e0da4f2925b0 | Company B    | Viet Nam
 6400a3aa-01b0-4215-ae67-87b53b0d9431 | Company C    | Viet Nam
(3 rows)
```

## Pgbackrest Configuration - Master server configuration

Install pgbackrest
```
sudo apt-get -y install pgbackrest
```

Verify pgbackrest already installed

```
vagrant@ubuntu-bionic:~$ pgbackrest
pgBackRest 2.30 - General help

Usage:
    pgbackrest [options] [command]

Commands:
    archive-get     Get a WAL segment from the archive.
    archive-push    Push a WAL segment to the archive.
    backup          Backup a database cluster.
    check           Check the configuration.
    expire          Expire backups that exceed retention.
    help            Get help.
    info            Retrieve information about backups.
    restore         Restore a database cluster.
    stanza-create   Create the required stanza data.
    stanza-delete   Delete a stanza.
    stanza-upgrade  Upgrade a stanza.
    start           Allow pgBackRest processes to run.
    stop            Stop pgBackRest processes from running.
    version         Get version.

Use 'pgbackrest help [command]' for more information.
```

Update postgresql.conf

```
sudo vim /etc/postgresql/11/main/postgresql.conf
```

Add the below lines

```
archive_mode = on
archive_command = 'pgbackrest --stanza=demo archive-push %p'
```

Restart postgres
```
sudo service postgresql restart
```

Check postgresql configurations
```
sudo -iu postgres psql
```

```
SELECT name,setting,context,source FROM pg_settings WHERE NAME IN ('listen_addresses','archive_mode','password_encryption');
        name         |    setting    |  context   |       source
---------------------+---------------+------------+--------------------
 archive_mode        | on            | postmaster | configuration file
 listen_addresses    | *             | postmaster | configuration file
 password_encryption | scram-sha-256 | user       | configuration file
(3 rows)
```

Exit psql & logout postgres user

```
\q

exit;
```

### Config Pgbackrest

```
sudo mkdir -p /var/lib/pgbackrest 
sudo chmod 0750 /var/lib/pgbackrest 
sudo chown -R postgres:postgres /var/lib/pgbackrest
```

Set permission
```
sudo chown -R postgres:postgres /var/log/pgbackrest
```

Create backup file of pgbackrest
```
sudo cp /etc/pgbackrest.conf /etc/pgbackrest.conf.backup
```

Generate password
```
openssl rand -base64 48
```

Edit pgbackrest.conf
```
sudo vim /etc/pgbackrest.conf
```

Add flowing lines
```
[global]
repo1-cipher-pass=nFYC2Vy6cqRejtXAw7OAc8jhl9ENLHOhE2L9QpqgKdHS85opIpr0O++BK9BLxC4B
repo1-cipher-type=aes-256-cbc
repo1-path=/var/lib/pgbackrest
repo1-retention-diff=2
repo1-retention-full=2
log-level-console=info
log-level-file=debug
start-fast=y

[demo]
pg1-path=/var/lib/postgresql/11/main
```

Test create stanza
```
vagrant@ubuntu-bionic:~$ sudo -u postgres pgbackrest --stanza=demo stanza-create
2020-11-18 06:21:35.129 P00   INFO: stanza-create command begin 2.30: --log-level-console=info --log-level-file=debug --pg1-path=/var/lib/postgresql/11/main --repo1-cipher-pass=<redacted> --repo1-cipher-type=aes-256-cbc --repo1-path=/var/lib/pgbackrest --stanza=demo
2020-11-18 06:21:35.791 P00   INFO: stanza-create command end: completed successfully (662ms)
```

Check stanza
```
vagrant@ubuntu-bionic:~$ sudo -iu postgres pgbackrest --stanza=demo check
2020-11-18 06:22:33.042 P00   INFO: check command begin 2.30: --log-level-console=info --log-level-file=debug --pg1-path=/var/lib/postgresql/11/main --repo1-cipher-pass=<redacted> --repo1-cipher-type=aes-256-cbc --repo1-path=/var/lib/pgbackrest --stanza=demo
2020-11-18 06:22:34.145 P00   INFO: WAL segment 000000010000000000000003 successfully archived to '/var/lib/pgbackrest/archive/demo/11-1/0000000100000000/000000010000000000000003-188c9090400b4b47f71ec5dae2f3577acdecd38a.gz'
2020-11-18 06:22:34.145 P00   INFO: check command end: completed successfully (1104ms)
```

Perform first full backup

```
sudo -u postgres pgbackrest --stanza=demo --type=full backup

.......
2020-11-18 06:23:16.369 P00   INFO: new backup label = 20201118-062305F
2020-11-18 06:23:16.422 P00   INFO: backup command end: completed successfully (12085ms)
2020-11-18 06:23:16.422 P00   INFO: expire command begin 2.30: --log-level-console=info --log-level-file=debug --repo1-cipher-pass=<redacted> --repo1-cipher-type=aes-256-cbc --repo1-path=/var/lib/pgbackrest --repo1-retention-diff=2 --repo1-retention-full=2 --stanza=demo
2020-11-18 06:23:16.440 P00   INFO: expire command end: completed successfully (18ms)
```