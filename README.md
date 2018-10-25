# MySQL Group Replication

## Creating the environment

This tutorial uses Virtualbox and Vagrant. Is assumed that you are familiar with this tools (explanation of them is out of scope)

Follow this steps to get a setup:

### Install VirtualBox. 

Version 5.1.18 works. Download Virtualbox from [here](https://www.virtualbox.org/wiki/Downloads).

### Install Vagrant. 

Version 2.0.1 works. Download Vagrant from [here](http://vagrantup.com/).

### Install Vagrant plugin hostmanager

This plugin will take care of dealing with the /etc/hosts file so we can use hostnames instead of IPs. https://github.com/devopsgroup-io/vagrant-hostmanager

To install it, just run:

```bash
vagrant plugin install vagrant-hostmanager
```

### Create the environment

For this tutorial, the env consist of 3 MySQLs, which will become 1 Master and 2 Slaves within the same Cluster (GR)

### Clone the repo

```bash
git clone https://github.com/nethalo/gr-basics.git
```

### Start the build

Run 

```bash
cd gr-basics; vagrant up; vagrant hostmanager;
```

The whole process takes a while the first time (around 20 minutes) so go grab some coffee and be back later.

Continue when the environment is done.

## Bootstrap the cluster

In the mysql1 VM (vagrant ssh mysql1), run the following commands in the mysql client:

```mysql
INSTALL PLUGIN group_replication SONAME 'group_replication.so';
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF
```

## Create the repl user

We need an user that will connect to the GR channel replication:

```mysql
CREATE USER repl@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO repl@'%';
FLUSH PRIVILEGES;
CHANGE MASTER TO MASTER_USER='repl', MASTER_PASSWORD='password' FOR CHANNEL 'group_replication_recovery';
```


## Add new nodes

On **mysql2/mysql3**:

```mysql
INSTALL PLUGIN group_replication SONAME 'group_replication.so';
CHANGE MASTER TO MASTER_USER='repl', MASTER_PASSWORD='password' FOR CHANNEL 'group_replication_recovery';
START GROUP_REPLICATION;
```
## Check the cluster

Run the following query 

```mysql
SELECT * FROM performance_schema.replication_group_members\G
```

You should see this:

```mysql
mysql> SELECT * FROM performance_schema.replication_group_members\G
*************************** 1. row ***************************
CHANNEL_NAME: group_replication_applier
   MEMBER_ID: 2b4beba5-d899-11e8-9f94-525400cae48b
 MEMBER_HOST: mysql2
 MEMBER_PORT: 3306
MEMBER_STATE: ONLINE
*************************** 2. row ***************************
CHANNEL_NAME: group_replication_applier
   MEMBER_ID: 7a83200b-d898-11e8-977b-525400cae48b
 MEMBER_HOST: mysql1
 MEMBER_PORT: 3306
MEMBER_STATE: ONLINE
*************************** 3. row ***************************
CHANNEL_NAME: group_replication_applier
   MEMBER_ID: d233088a-d899-11e8-a538-525400cae48b
 MEMBER_HOST: mysql3
 MEMBER_PORT: 3306
MEMBER_STATE: ONLINE
3 rows in set (0.00 sec)
```

### Find out who is the master

Run this query:

```mysql
SELECT 
      member_host AS "primary master"
FROM 
      performance_schema.global_status
JOIN performance_schema.replication_group_members
WHERE 
   variable_name = 'group_replication_primary_member'
   AND member_id=variable_value;
```

In this case, is mysql1:

```mysql
mysql> SELECT member_host AS "primary master"
    -> FROM performance_schema.global_status
    -> JOIN performance_schema.replication_group_members
    -> WHERE variable_name = 'group_replication_primary_member'
    ->   AND member_id=variable_value;
+----------------+
| primary master |
+----------------+
| mysql1         |
+----------------+
1 row in set (0.06 sec)
```

Have fun!
