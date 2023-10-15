# MariaDB-master-master-replication
![MariaDB 10.6.12](https://img.shields.io/badge/MariaDB-10.6.12-%234479A1.svg?style=for-the-badge&logo=mariadb&logoColor=white) ![Master-Master Replication](https://img.shields.io/badge/Master--Master%20Replication-%234479A1.svg?style=for-the-badge) [![YouTube](https://img.shields.io/badge/YouTube-%23FF0000.svg?style=for-the-badge&logo=youtube&logoColor=white)](https://www.youtube.com/@1devops2)

## Environment Setup
  - Two fresh ubuntu (jammy 22.04) machines
  - 1cpu, 1GB ram, 30GB, root user
  - installed mariaDB or to install:
    
```
apt update; apt install mariadb-server -y
```

## Configuring Master-Master Replication
### Configuration of Master-1
- Navigate to the mariadb 50-server configuration file:
```
nano /etc/mysql/mariadb.conf.d/50-server.cnf
```
- add/replace the following content into that file:
```
server_id               = 1
bind-address            = 0.0.0.0
log_bin                 = /var/lib/mysql/mariadb-bin
log_bin_index           = /var/lib/mysql/mariadb-bin.index
relay_log               = /var/lib/mysql/relay-bin
relay_log_index         = /var/lib/mysql/relay-bin.index
```
- restart the mariadb
```
systemctl restart mariadb
```
- to create a user to configure replication, Open up the MariaDB prompt from your terminal: ```mysql``` and run below commands:
```
create user 'master1'@'%' identified by 'pinkapple';
grant replication slave on *.* to 'master1'@'%';
```
- To add a Slave, we need to get bin_log data from the Master-1 server:

```
show master status;
```
- output #

```
MariaDB [(none)]> show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000002 |      342 |              |                  |
+------------------+----------+--------------+------------------+
1 row in set (0.000 sec)
```

### Configuration of Master-2
- Navigate to the mariadb 50-server configuration file:
```
nano /etc/mysql/mariadb.conf.d/50-server.cnf
```
- add/replace the following content into that file:
```
server_id               = 2
bind-address            = 0.0.0.0
log_bin                 = /var/lib/mysql/mariadb-bin
log_bin_index           = /var/lib/mysql/mariadb-bin.index
relay_log               = /var/lib/mysql/relay-bin
relay_log_index         = /var/lib/mysql/relay-bin.index
```
- restart the mariadb
```
systemctl restart mariadb
```
- to create a user to configure replication, Open up the MariaDB prompt from your terminal: ```mysql``` and run below commands:
```
create user 'master2'@'%' identified by 'pinkapple';
grant replication slave on *.* to 'master2'@'%';
```
- To add a Slave, we need to get bin_log data from the Master-2 server:

```
show master status;
```
- output #

```
MariaDB [(none)]> show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |      665 |              |                  |
+------------------+----------+--------------+------------------+
1 row in set (0.000 sec)
```
Let’s configure the connection between MariaDB servers in our software replication cluster.
- Stop the slave:
```
STOP SLAVE;
```
- Add Master-1 to the second server:
```
CHANGE MASTER TO MASTER_HOST='<your-master1-ip-here>', MASTER_USER='master1', MASTER_PASSWORD='pinkapple', MASTER_LOG_FILE='mysql-bin.000002', MASTER_LOG_POS=342;
```
- Start the replication:
```
START SLAVE;
```
- check slave status:
```
show slave status \G
```
- output #
```
MariaDB [(none)]> show slave status \G
*************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                   Master_Host: 54.144.126.136
                   Master_User: master1
                   Master_Port: 3306
                 Connect_Retry: 60
               Master_Log_File: mysql-bin.000002
           Read_Master_Log_Pos: 342
                Relay_Log_File: relay-bin.000003
                 Relay_Log_Pos: 641
         Relay_Master_Log_File: mysql-bin.000002
              Slave_IO_Running: Yes
             Slave_SQL_Running: Yes
               Replicate_Do_DB:
...
```
### Returning back to Master-1 server
- Open up the MariaDB prompt from your terminal: ```mysql```.
- Stop the slave:
```
STOP SLAVE;
```
- Add Master-1 to the second server:
```
CHANGE MASTER TO MASTER_HOST='<your-master2-ip-here>', MASTER_USER='master2', MASTER_PASSWORD='pinkapple', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=665;
```
- Start the replication:
```
START SLAVE;
```
- check slave status:
```
show slave status \G
```
- output #
```
MariaDB [(none)]> show slave status \G                                                                                                       
*************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                   Master_Host: 54.221.110.98
                   Master_User: master2
                   Master_Port: 3306
                 Connect_Retry: 60
               Master_Log_File: mysql-bin.000001
           Read_Master_Log_Pos: 665
                Relay_Log_File: relay-bin.000002
                 Relay_Log_Pos: 964
         Relay_Master_Log_File: mysql-bin.000001
              Slave_IO_Running: Yes
             Slave_SQL_Running: Yes
...
```
## Check Replication Between MariaDB Servers
to make sure that the replication between two MariaDB servers works in master+master, we will create a new database & create a table on Master-1.
- On Master-1:
```
# create db and insert some records
create database replic;
use replic;

# creating table in replic database
CREATE TABLE replictable (
    employee_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    department VARCHAR(50),
    salary DECIMAL(10, 2)
);

# insert records into `replictable` 
INSERT INTO replictable (first_name, last_name, department, salary) VALUES
    ('John', 'Doe', 'HR', 50000.00),
    ('Jane', 'Smith', 'IT', 60000.00),
    ('Bob', 'Johnson', 'Finance', 55000.00);
```
- Validate on Master-2:
```
show databases;
use replic;

# to show tables
select * from replictable;

# add more records
INSERT INTO replictable (first_name, last_name, department, salary) VALUES
    ('lisa', 'lily', 'devops', 990000.00),
    ('rick', 'ifri', 'css', 90000.00);
```

