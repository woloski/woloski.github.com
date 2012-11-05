---
layout: post
title: Installing PostgreSQL on Ubuntu 12 running on Windows Azure 
tags : [postgres, azure, iaas, vm]
---
{% include JB/setup %}

Create a new VM on the Windows Azure portal (NEW -> Virtual Machine -> Ubuntu Server 12.04). Make sure to enable SSH (here are some [instructions to generate a key](https://www.windowsazure.com/en-us/manage/linux/how-to-guides/ssh-into-linux/))

<img src="http://puu.sh/1fq8Y" width="600" />

Open the default Postgres port (5432) on the Windows Azure portal. You will find the "endpoints" tab on the virtual machine.

<img src="http://puu.sh/1fq9P" width="600" />

Connect via SSH

    ssh -i <postgres-db>.key <someuser>@<postgres-db>.cloudapp.net

Once connected, run the following commands to install Postgres

    sudo apt-get install postgresql
    # we need postgis features
    sudo apt-get install postgresql-9.1-postgis
    sudo apt-get install postgresql-contrib

    # create a user and assign a password
    sudo -u postgres createuser --superuser <someuser> -P

    sudo vi /etc/postgresql/9.1/main/pg_hba.conf
    #add the following line to allow any IP connecting with user and password hashed with md5, you can also specify specific address or range
    host    all all 0.0.0.0 0.0.0.0 md5

    sudo vi /etc/postgresql/9.1/main/postgresql.conf 
    #uncomment and change the listen_addresses line to remote connections
    listen_addresses = '*'

    # restart
    sudo service postgresql restart


### Restoring a database from a dump

    createdb somedb -T template0
    pg_restore -d somedb somedb.dump

### Connecting to it

You can try connecting through the psql command line by doing

    psql -h <postgres-db>.cloudapp.net -U <someuser>

Enjoy!

