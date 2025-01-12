# Template Description

This template is to be used when you need to create a new mysql user in an existing environment. This should be able to be used for all types of MySQL including Percona, MariaDB, MySQL community, IGR, PXC, Galera, etc.

This template assumes that the user needs to be available on all hosts in the topology/cluster. If this use is only supposed to be created on a single machine you will need to add steps to limit the creation by disbaling sql_log_bin and only running the create user statement on the host where the user should be created.

# Tags

- {ticket_number} = The ticket number for this implementation
- {client_ssh_ms_connect_name} = The identifier that you would use to connect to the this client using ssh_ms connect
- {new_mysql_user_username} = The username portion of the new user. Example '**user**'@'host'
- {new_mysql_user_host} = The host portion of the new user. Example 'user'@'**host**'
- {new_mysql_user_grants} = The grants for the new user. This should be the full grant scope. Example: ALL on *.*
- {mysql_source_host} = The source / writer host for the topology
- {mysql_source_host_ip_address} = The IP address of the source / writer host for the topology
- {mysql_source_host_port} = The port of the source / writer host for the topology

# Description of Work

Create mysql user '{new_mysql_user_username}'@'{new_mysql_user_host}'

# Estimated Time Requirement

A block of 30 minutes will be required in our maintenance calendar for this task.

# Expected Impact

There is no expected performance impact or downtime on database hosts for this task.

# Implementation Plan

## Pre-flight checks:

- Topology
```bash
db_tree

# NOTE: This is to confirm which host is the writer / source host as this is where you will need to create the user
```

- Does user already exist?
```bash
# For all hosts in the environment

db_connect <host> 
select count(*) from mysql.user where user = '{new_mysql_user_username}' and host = '{new_mysql_user_host}'

# Note: Should return 0. This is to confirm that the user does not already exist on any of the nodes. We will create the user using
# create user if not exists, but we should still check to make sure the user does not exist as, if the use exists,
# then this request may be invalid.
# In short, if the user exists stop here and report it to the client
```

## Work Steps

1) Connect to the monitor host for the client and start a new screen session
    ```bash
    ssh_ms connect {client_ssh_ms_connect_name}
    mkdir -p ~/{ticket_number}/
    cd ~/{ticket_number}
    screen -SL {ticket_number}-create-mysql-user
    ```

1) Generate a strong password and save this on the monitor host so that it can be retrieved by the client
    ```bash
    tr -dc 'A-Za-z0-9!?%=' < /dev/urandom | head -c 20 > ./{new_mysql_user_username}_{new_mysql_user_host}_generated_password.txt
    cat ./generated_password.txt
    ```

1) Create the user on {mysql_source_host}
    ```bash
    mysql -h {mysql_source_host_ip_address} -P {mysql_source_host_port} -e "create user if not exists '{new_mysql_user_username}'@'{new_mysql_user_host}' identified by '<password generated in previous step>'";

    # Note: In the event that you need to create a password using the native mysql password hashing method, please replace "identified by" with "identified with mysql_native_password by"
    ```

1) Provide grants to the new user
    ```bash
    mysql -h {mysql_source_host_ip_address} -P {mysql_source_host_port} -e "grant {new_mysql_user_grants} to '{new_mysql_user_username}'@'{new_mysql_user_host}'"

    # Note: Dependng on the grant you may need to include "with grant option" at the end
    ```

1) Confirm that the new user exists
    ```bash
    # For all hosts in the environment
    db_connect <host>
    SHOW GRANTS FOR '{new_mysql_user_username}'@'{new_mysql_user_host}'
    ```

1) Inform the client that the password can be found on the monitor host in file ~/{ticket_number}/{new_mysql_user_username}_{new_mysql_user_host}_generated_password.txt. Delete this file as soon as the client has confirmed that the password has been collected. If the customer is not able to reach the monitor host, please ask for the password in their slack channel and we will find a way to provide it to them

# ROLLBACK

1) Connect to the monitor host for the client and start a new screen session
    ```bash
    ssh_ms connect {client_ssh_ms_connect_name}
    mkdir -p ~/{ticket_number}/
    cd ~/{ticket_number}
    screen -SL {ticket_number}-create-mysql-user-rollback
    ```

1) Drop the user on {mysql_source_host} and let it propogate via replication
    ```bash
    mysql -h {mysql_source_host_ip_address} -P {mysql_source_host_port} -e "drop user if exists '{new_mysql_user_username}'@'{new_mysql_user_host}'";
    ```

1) Confirm the user has been removed
    ```bash
    # For all hosts in the environment

    db_connect <host> 
    select count(*) from mysql.user where user = '{new_mysql_user_username}' and host = '{new_mysql_user_host}'

    # NOTE: Should return zero
    ```

1) Delete the generated passowrd file if it has not already been removed.
    ```bash
    rm ~/{ticket_number}/{new_mysql_user_username}_{new_mysql_user_host}_generated_password.txt
    ```
