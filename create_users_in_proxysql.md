# Template Description

This template is used for creating a new ProxySQL MySQL user in an existing environment. It supports all types of MySQL, including Percona, MariaDB, MySQL Community, IGR, PXC, Galera, and others.

# Tags

- {ticket_number} = The ticket number for this implementation
- {client_ssh_ms_connect_name} = The identifier that you would use to connect to the this client using ssh_ms connect
- {mysql_source_host} = The source/writer host for the topology
- {proxysql_host} = The proxysql in question
- {user} = The Proxysql to be created

# Description of Work

Create a user in Proxysql.

# Estimated Time Requirement

A block of 30 minutes will be required in our maintenance calendar for this task.

# Expected Impact

This task has no expected performance impact or downtime on database hosts.

# Implementation Plan

## Pre-flight checks:

- Topology
```bash
db_tree

# NOTE: This is to confirm which host is the writer/source host as this is where you will need to create the user
```
- Proxysql servers
```bash
db_connect --list | grep -i proxysql  # For GASCAN
db_connect | grep proxysql            # For MSP

# NOTE: This command will show you how many proxysql servers the customer has
```

- Does the user already exist?
```bash
db_connect <host> 
select count(*) from mysql.user where user = '{new_mysql_user_username}'

ssh_connect {proxysql_host}
mysql -e "select * from mysql_users"

# Note: In Proxysql, it should return 0. This confirms that the user does not already exist on any of the nodes.
# We will create the user in Proxysql by inserting the necessary row/s 
# If the user exists, stop here and report it to the client
```
- mysql_hostgroups
```bash
ssh_connect {proxysql_host}
mysql -e "select * from mysql_replication_hostgroups"   # For async replication
mysql -e "select * from mysql_galera_hostgroups"        # For PXC/Galera setup 

# Note: The check will show you which is the writer and reader hostgroups
# We will use these values when creating the users in Proxysql
```


## Work Steps

1) Connect to the monitor host for the client and start a new screen session
    ```bash
    ssh_ms connect {client_ssh_ms_connect_name}
    mkdir -p ~/{ticket_number}/
    cd ~/{ticket_number}
    screen -SL {ticket_number}-create-proxysql-user
    ```
1) Connect to the database instance and get the user info
    ```bash
    db_connect {mysql_source_host}
    SELECT DISTINCT CONCAT("INSERT INTO mysql_users (username, password, default_hostgroup) VALUES (", CONCAT("'",User,"'"),","  ,CONCAT("'," authentication_string,"',"",2") ,");")
    from mysql.user WHERE user = '{user}';

    # This command will generate a statement like this
    INSERT INTO mysql_users (username,password, default_hostgroup) VALUES ('{user}','*197807CBFABA0820A315792313EB15F917DFQAC',2);

    # Change the default_hostgroup with the writer hostgroup obtained in the preflight section
    ```   

1) Connect to the proxysql, create the user, load to runtime,  save it to disk, and verify
    ```bash
    ssh_connect {proxysql_host}
    mysql
    INSERT INTO mysql_users (username,password, default_hostgroup) VALUES ('percona','*197807CBFABA0820A315792313EB15F917DFQAC',2);
    LOAD MYSQL USERS TO RUNTIME;
    SAVE MYSQL USERS TO DISK;
    SELECT * FROM mysql_users WHERE username='{user}';
    SELECT * FROM runtime_mysql_users WHERE username='{user}';
    ```

1) Inform the client that the user was created

# ROLLBACK

1) Connect to the monitor host for the client and start a new screen session
    ```bash
    ssh_ms connect {client_ssh_ms_connect_name}
    mkdir -p ~/{ticket_number}/
    cd ~/{ticket_number}
    screen -SL {ticket_number}-create-mysql-user-rollback
    ```

1) Connect to the Proxysql server, delete the user created, load to runtime, save it to disk, and verify
    ```bash
    ssh_connect {proxysql_host}
    mysql
    DELETE FROM mysql_users WHERE username = '{user}';
    LOAD MYSQL USERS TO RUNTIME;
    SAVE MYSQL USERS TO DISK;
    SELECT * FROM mysql_users WHERE username = '{user}';
    ```

1) Confirm the user has been removed
    ```bash
    ssh_connect {proxysql_host}
    mysql
    SELECT * FROM mysql_users WHERE username = {user};    
    # NOTE: Should return zero
    ```

1) Delete the generated passowrd file if it has not already been removed.
    ```bash
    rm ~/{ticket_number}/{new_mysql_user_username}_{new_mysql_user_host}_generated_password.txt
    ```
