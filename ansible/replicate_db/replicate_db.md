**Copy MariaDB database from one server to another and set up replication**

This playbook is a continuation of the [Install Matomo /phpMyAdmin playbook](../README.md) and is intended for the case where 
you do not just need an empty Matomo installation, but want to move a running instance to a new server.

Of course, there is nothing Matomo-specific in it, so you can use it for any MariaDB database replication.

**Prerequisites**
 - Ansible installed on the machine where this will run from (tested with version 2.16.3.)
 - slave (target) server running Ubuntu (tested on version 24.04)
 - MariaDB 10.5 and higher on the target server
 - SSH access to both source target server with a user that has sudo privileges
 - there are some commented-out sections pertaining to the source server, which you can uncomment if it runs Fedora

Command (on where you will be running this from):
ansible-galaxy collection install community.mysql community.general
ansible-galaxy collection install ansible.posix

What this playbook does:
 - connects to a running MariaDB server ("source") and configures it to become a replication master
 - creates a new user on the source server with replication privileges
 - allows remote connection to the database (both in the MariaDB settings, and in the firewall)
Note that this is currently not limited to the target host, due to potential issues with reverse DNS lookups. 
You will have to edit the master_create.yml and change that if you want tighter limitations.

 - on the target server, mariadb-client is installed (if not already present)
 - a new, empty database is created with the given name (note: same database name on both servers; this can be changed, but is currently out of scope, you will need to edit the playbook accordingly)
 - if a database with that name already exists, the playbook will fail, which is a deliberate safety measure
 - the database is owned by an application user which has to be created beforehand (e.g. by the Matomo installation playbook)
 - the database is dumped from the source server into a local file
 - the dump is then restored to the target server; it carries in it information about exactly when it was taken (data position as at the time when data dump started)
 - the server is then configured to become a replication slave, replicating the source server

**Motivation**

The playbook was created to automate what is otherwise a fiddly and thankless task, which is error-prone and very time-consuming if done manually.

It is also meant to allow the switching between two database servers to happen with minimal disruption and loss of data.

The normal approach would be:
 - log into the source server, dump database to a file
 - copy the file to the target server
 - import the file into the target server
 - configure Matomo on the new server
 - update the site(s) that Matomo is tracking to point to the new server

These operations take time - minutes, or more. When you finally get everything going again, you will have lost all data that was collected in the period between the start of the data dump and the moment that tracking requests start coming into the new server - from minutes to hours.
With the approach of this playbook, the target server will be up to date until you make the switch. With some sleight of hand, you can stop replication and point your tracking code to the new server within seconds, and you will only lose data during these seconds.
In other words, this is still not "production grade", but is "good enough" for most use cases.

**How to Use**
First, edit the inventory.ini file to match your servers and users.  Needless to say, **do not commit those changes to git**.
Then, edit the local_vars.yml file to match your database settings and users. For example, settings like:
source_mariadb_conf_dir: /etc/my.cnf.d
source_mariadb_unix_socket: /var/lib/mysql/mysql.sock
fit a Fedora server, while:
target_mariadb_unix_socket: /run/mysqld/mysqld.sock
target_mariadb_conf_dir: /etc/mysql/mariadb.conf.d
are for Ubuntu.

Within a replication setup, the source server usually has server_id = 1; each other server (slave) should have a unique ID. 
For the purposes of this playbook, there is only one target server (slave) which is set to 101, but if you edit it and add more (or run the slave setup multiple times for different hosts) - the others should get different IDs.

Then, from command line run:
```bash
ansible-playbook -i inventory.ini replicate_db.yml -e target_group=local
```

This is the "main" playbook, which sets up SSH access to the servers, configures the source server as a master, configures the replication slave, creates the database dump, transfers it to the target server, restores it there, and configures replication.
You can also run the mini-playbooks separately:
    - add_known_hosts.yml - adds the source and target servers to known_hosts file, so that Ansible can connect to them via SSH
    - master_create.yml - configures the source server as a replication master
    - slave_create.yml - configures the target server as a replication slave
    - slave_check.yml - checks the status of the replication on the target server (not that this expects slave to be running and will time out if it's stopped)

When you no longer need replications, run (in this order):
    - slave_stop.yml - stops replication on the target server and removes the "read-only" status of the database; you should immediately also change your site's tracking code to point to the new location
    - master_stop.yml - removes the replication settings and user, and disables remote access on the source server

**Technical Notes**
The service name (needed when restarting mysql) on both servers is supposed to be "mariadb" (works on Fedora and Ubuntu). If it is differently named on your source server, you will need to edit the master_create.yml playbook accordingly.

Data Import and Replication Compatibility:
Replication generally works between different MariaDB versions as long as the target version is equal to or newer than the source version.
However, major version differences (e.g., 10.3 to 10.6) may introduce incompatibilities in replication features or SQL syntax.  
The mysqldump output from the source may include features or syntax not supported by the target version, leading to errors during the restore process.  

For simplicity, the database name is set to be the same on both sides (target and source).
They could be different, but then the replica will need to be set via a config file (replicate-rewrite-db=source_db_name->target_db_name) and not with a query as it is now.

The command START|STOP REPLICA used to be START|STOP SLAVE in older MariaDB versions. If you have an older version, you will need to change the respective commands in the playbook.

File vs Stream
The initial approach tried with the playbook was to pipe the output of mysqldump directly into the mysql command on the target server, avoiding the need for a temporary file. 
This actually worked quite nicely. The problem with though was that, on starting the slave server, you need to give it a "binlog position" from which to start replicating.
That position is obtained from the output of "SHOW MASTER STATUS" on the source server but, as ever, the devil is in the details... If captured after the dump, then all data saved during the dump will be lost (the dump happens in a transaction which is consistent to its start time), thu defeating the purpose of the exercise.
If the position is captured before the dump, then duplicate rows appear when replication tries to start, and the slave setup crashes. It can work (and did work, once...) on a small and "quiet" database with not many updates, but if that is your case - you would not need this playbook to begin with.
In the end, the playbook uses GTID (whatever that is...), which uses a start position embedded as part of the dump file. Ansible cannot parse it from the data stream, so it had to all be saved to a file.

Users
The replication user on the source database will be deleted then created again, to avoid conflicts with existing privileges.
To debug user privileges, you can run this query in your database:
SHOW GRANTS FOR 'replication_user'@'%';
Note the '%' part - it means that the user can connect from any host. You can change that to a specific host if you want to limit access, at which point you can also try:
SHOW GRANTS FOR 'replication_user'@'target_host';
And you may be surprised to see that the same user has different privileges when connecting from different hosts...

Whatever machine you run the playbook from, it will need to be able to connect to one or both source and target servers via SSH, for which their keys are added by the add_known_hosts.yml mini-playbook.
There may be an intermittent message like 'Using a SSH password instead of a key is not possible because Host Key checking is enabled and sshpass does not support this." but a key will be saved later.'

Debug master status on source from CLI:
mysql -u root -p -e "SHOW MASTER STATUS;"

Debug slave status on target from CLI:
mysql -u root -p -e "SHOW SLAVE STATUS;"
Or, run the slave_check.yml mini-playbook

**Fallout After Install**
There may be an issue if Matomo has different versions which require different database structure. Normally, when Matomo is upgraded, it also requires an upgrade of the database schema. 
If that is the case, then - after database import and replication - Matomo will not be in working order yet.
You will need to stop replication and perform database upgrade (from the Matomo panel) to make it usable.

