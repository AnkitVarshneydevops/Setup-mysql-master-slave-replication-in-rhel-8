# Setup-mysql-master-slave-replication-in-rhel-8
Install a MySQL in Master Server
First, proceed with MySQL installation using YUM command. If you already have MySQL installation, you can skip this step.

# yum install @mysql

Configure a MySQL in Master Server

Open my.cnf configuration file with VI editor.

# vi /etc/my.cnf
Add the following entries under [mysqld] section and don’t forget to replace DevOps with database name that you would like to replicate on Slave.

server-id = 1
binlog-do-db=DevOps
relay-log = /var/lib/mysql/mysql-relay-bin
relay-log-index = /var/lib/mysql/mysql-relay-bin.index
log-error = /var/lib/mysql/mysql.err
master-info-file = /var/lib/mysql/mysql-master.info
relay-log-info-file = /var/lib/mysql/mysql-relay-log.info
log-bin = /var/lib/mysql/mysql-bin
Restart the MySQL service.

# systemctl restart mysqld

Login into MySQL as root user and create the slave user and grant privileges for replication. Replace slave_user with user and your_password with password.

# mysql
mysql> CREATE USER 'slave_user'@'%' IDENTIFIED BY 'password'; 
mysql> GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%';
mysql> FLUSH PRIVILEGES;
mysql> FLUSH TABLES WITH READ LOCK;
mysql> SHOW MASTER STATUS;

+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000003 | 1520     | DevOps    	 |                  |
+------------------+----------+--------------+------------------+
1 row in set (0.00 sec)

mysql> quit;
Please write down the File (mysql-bin.000003) and Position (11128001) numbers, we required these numbers later on Slave server. Next apply READ LOCK to databases to export all the database and master database information with mysqldump command.

#  mysqldump -u root -p --all-databases --master-data > /root/dbdump.db
Once you’ve dump all the databases, now again connect to mysql as root user and unlcok tables.

mysql> UNLOCK TABLES;
mysql> quit;
Upload the database dump file on Slave Server (192.168.1.2) using SCP command.

scp /root/dbdump.db root@192.168.1.2:/root/
That’s it we have successfully configured Master server, let’s proceed to Phase II section.

Phase II: Configure Slave Server (192.168.1.2) for Replication
In Phase II, we do the installation of MySQL, setting up Replication and then verifying replication.

Install a MySQL in Slave Server
If you don’t have MySQL installed, then install it using YUM command.

# yum install mysql-server mysql
Configure a MySQL in Slave Server
Open my.cnf configuration file with VI editor.

# vi /etc/my.cnf
Add the following entries under [mysqld] section and don’t forget to replace IP address of Master server, tecmint with database name etc, that you would like to replicate with Master.

[mysqld]
server-id = 2

Now import the dump file that we exported in earlier command and restart the MySQL service.

# mysql -u root -p < /root/dbdump.db
# /etc/init.d/mysqld restart
Login into MySQL as root user and stop the slave. Then tell the slave to where to look for Master log file, that we have write down on master with SHOW MASTER STATUS; command as File (mysql-bin.000003) and Position (1520) numbers. You must change 192.168.1.1 to the IP address of the Master Server, and change the user and password accordingly.

# mysql -u root -p
mysql> slave stop;
mysql> CHANGE MASTER TO MASTER_HOST='192.168.1.1', MASTER_USER='slave_user', MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-bin.000003', MASTER_LOG_POS=1520;
mysql> slave start;
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.1.1
                  Master_User: slave_user
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 12345100
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 11381900
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: DevOps
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 12345100
              Relay_Log_Space: 11382055
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
1 row in set (0.00 sec)
Verifying MySQL Replication on Master and Slave Server
It’s really very important to know that the replication is working perfectly. On Master server create table and insert some values in it.

On Master Server
mysql> create database dev;
mysql> use dev;
mysql> CREATE TABLE employee (c int);
mysql> INSERT INTO employee (c) VALUES (1);
mysql> SELECT * FROM employee;
+------+
|  c  |
+------+
|  1  |
+------+
1 row in set (0.00 sec)
On Slave Server
Verifying the SLAVE, by running the same command, it will return the same values in the slave too.

mysql> use dev;
mysql> SELECT * FROM employee;
+------+
|  c  |
+------+
|  1  |
+------+
1 row in set (0.00 sec)
That’s it, finally you’ve configured MySQL Replication in a few simple steps.
