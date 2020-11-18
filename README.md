# Postgres DBA with Vagrant

Demo how to configure postgres cluster, integrate backup tools from scratch using Vagrant VM

```
Master's IP address: 192.168.34.10 

Slave's IP address: 192.168.34.20
```

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

## Pgbackrest Configuration

[TODO]