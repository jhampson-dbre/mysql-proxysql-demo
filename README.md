# mysql-proxysql-demo

We will use Docker to configure MySQL replication and then setup ProxySQL to offload read-only workload to MySQL replica server.

1. Download container images for MySQL and ProxySQL

   ```powershell
   docker pull mysql:8.0.28-debian
   docker pull proxysql/proxysql:2.3.2
   ```

1. Create a Docker network to use for setting up replication

   ```powershell
   docker network create mysql-net
   ```

1. Create the container that will act as our read/write source

   ```powershell
   docker run `
   --name source `
   --env MYSQL_ROOT_PASSWORD=rootpassword `
   --publish 33060:3306 `
   --network mysql-net `
   --detach `
   mysql:8.0.28-debian `
   --default-authentication-plugin=mysql_native_password `
   --server-id=1 `
   --log-bin='mysql-bin-1.log' `
   --binlog-format=ROW `
   --gtid-mode=ON `
   --enforce-gtid-consistency `
   --log-replica-updates
   ```

1. Log into the source container

   ```powershell
   docker exec -it source /bin/bash

   mysql -uroot -prootpassword --prompt='Source> '
   ```

1. Create a database called `mydb` with a table named `object` and insert some data

   ```sql
   -- create database
   CREATE SCHEMA `mydb` DEFAULT CHARACTER SET utf8;
   -- change current database
   USE mydb;
   -- create table
   DROP TABLE IF EXISTS `object`;
   CREATE TABLE `object` (
     `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
     `text` varchar(45) NOT NULL,
     PRIMARY KEY (`id`)
   ) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;
   -- insert some data
   INSERT INTO `object` VALUES (1,'a'),(2,'b'),(3,'c');
   -- create the user used to perform replication
   CREATE USER 'repluser'@'%' IDENTIFIED BY 'replpassword';
   GRANT REPLICATION SLAVE ON *.* TO 'repluser'@'%';
   FLUSH PRIVILEGES;

   -- GTIDs are stored in gtid_executed table, check the current status
   SELECT * FROM mysql.gtid_executed;
   ```

1. Since we will need to switch back and forth between containers, open a new terminal prompt to create the container that will be setup as our read-only replica

   ```powershell
   docker run `
   --name replica1 `
   --env MYSQL_ROOT_PASSWORD=rootpassword `
   --publish 33061:3306 `
   --network mysql-net `
   --detach `
   mysql:8.0.28-debian `
   --default-authentication-plugin=mysql_native_password `
   --server-id=2 `
   --log-bin=mysql-bin-1.log `
   --binlog-format=ROW `
   --gtid-mode=ON `
   --enforce-gtid-consistency `
   --log-replica-updates `
   --report-host=replica1 `
   --report-port=3306
   ```

1. Log into the replica container

   ```powershell
   docker exec -ti replica1 /bin/bash
   mysql -u root -prootpassword --prompt='Replica1> '
   ```

1. Start replication

   ```sql
   CHANGE REPLICATION SOURCE TO
   SOURCE_HOST='source',
   SOURCE_PORT=3306,
   SOURCE_USER='repluser',
   SOURCE_PASSWORD='replpassword',
   SOURCE_AUTO_POSITION=1;
   START REPLICA;

   -- Check that our demo data has be replicated
   select * from mydb.object;
   ```

   If replication was configured successfully, we should see the data we inserted on the source

   ```sh
   +----+------+
   | id | text |
   +----+------+
   |  1 | a    |
   |  2 | b    |
   |  3 | c    |
   +----+------+
   3 rows in set (0.00 sec)
   ```

   Congratulations! You have successfully configured MySQL replication between the source container and replica1 container.

   Now, let's configure ProxySQL to route read queries to the replica and write queries to the source.

1. Create a proxysql.cnf file to mount inside the ProxySQL container

   A starter file can be found on the ProxySQL website under [Running ProxySQL with Docker](https://proxysql.com/documentation/installing-proxysql/). The starter file is all we need for this demo.

1. Open a new terminal window to setup ProxySQL

   NOTE: If the proxysql.cnf file you create in the previous step is not in the current working directory, update the path in the command below

   ```powershell
   docker run --name mysql-proxy --network mysql-net -p 16032:6032 -p 16033:6033 -p 16070:6070 -d -v $PWD/proxysql.cnf:/etc/proxysql.cnf proxysql/proxysql
   ```

1. The ProxySQL container does not include a MySQL client. We will use the MySQL client in the source container to configure ProxySQL.

   ```powershell
   docker exec -it source /bin/bash
   mysql -u radmin -pradmin -h 172.18.0.4 -P6032 --prompt='ProxyAdmin> '
   ```

   ```sql
   -- No servers are configured for ProxySQL yet
   SELECT * FROM mysql_servers;

   -- Insert the source and replica containers as ProxySQL backends
   INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (1,'source',3306);
   INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (2,'replica1',3306);
   ```

1. Run the follow SQL statements in the Source window to create a user we will use to test our connection to ProxySQL

   ```sql
   CREATE USER 'proxyuser'@'%' IDENTIFIED BY 'proxypassword';
   GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, RELOAD, PROCESS, REFERENCES, INDEX, ALTER, SHOW DATABASES, CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE, REPLICATION SLAVE, REPLICATION CLIENT, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, CREATE USER, EVENT, TRIGGER ON *.* TO 'proxyuser'@'%';
   FLUSH PRIVILEGES;
   ```

1. Allow the user we created to connect to ProxySQL. In the ProxyAdmin window, run:

   ```sql
   insert into mysql_users(username,password,default_hostgroup,transaction_persistent) values ('proxyuser','proxypassword',1,1);
   ```

1. In the Source window, create a user that ProxySQL will use for monitoring backend health

   ```sql
   CREATE USER 'monitoruser'@'%' IDENTIFIED BY 'monitorpassword';
   GRANT SELECT ON *.* TO ' monitoruser'@'%';
   FLUSH PRIVILEGES;
   ```

1. In the ProxyAdmin window, configure monitoring

   ```sql
   set mysql-monitor_username='monitoruser';
   set mysql-monitor_password='monitorpassword';

   -- set monitoring intervals
   UPDATE global_variables SET variable_value='2000' WHERE variable_name IN ('mysql-monitor_connect_interval','mysql-monitor_ping_interval','mysql-monitor_read_only_interval');
   ```

1. Configure routing rules to split the read and write traffic

   ```sql
   -- Set hostgroup 1 as writer and hostgroup 2 as reader
   INSERT INTO mysql_replication_hostgroups (writer_hostgroup,reader_hostgroup,comment) VALUES (1,2,'cluster1');

   -- Rule to send write traffic to to hostgroup 1
   insert into mysql_query_rules(rule_id,active,match_digest,destination_hostgroup,apply)values(1,1,'^SELECT.*FOR UPDATE$,^SELECT',1,1);

   insert into mysql_query_rules(rule_id,active,match_digest,destination_hostgroup,apply)values(2,1,'SELECT',2,1);

   load mysql users to runtime;
   load mysql servers to runtime;
   load mysql query rules to runtime;
   load mysql variables to runtime;
   load admin variables to runtime;

   save mysql users to disk;
   save mysql servers to disk;
   save mysql query rules to disk;
   save mysql variables to disk;
   save admin variables to disk;
   ```

1. Open a new terminal window to test the connection to MySQL through the proxy

```
docker exec -it source /bin/bash
mysql -h172.18.0.4 -uproxyuser -pproxypassword -P6033 --prompt='ProxyUser> '
```

1. Run select and update queries from the ProxyUser connection

   ```sql
   SELECT * FROM object;
   UPDATE object SET text='d' WHERE id=3;
   ```

1. From the ProxyAdmin window, confirm that the write and read workloads were routed to the expected hostgroups. The SELECT query should have ran on hostgroup 2, and the UPDATE statement should have ran on hostgroup 1

   ```sql
   SELECT * FROM stats_mysql_query_digest;

   +-----------+--------------------+-----------+----------------+--------------------+------------------------------------------+------------+------------+------------+----------+----------+----------+-------------------+---------------+
   | hostgroup | schemaname         | username  | client_address | digest             | digest_text                              | count_star | first_seen | last_seen  | sum_time | min_time | max_time | sum_rows_affected | sum_rows_sent |
   +-----------+--------------------+-----------+----------------+--------------------+------------------------------------------+------------+------------+------------+----------+----------+----------+-------------------+---------------+
   | 1         | information_schema | proxyuser |                | 0x6CFCF15A9E33D944 | UPDATE object SET text=? WHERE id=?      | 1          | 1649639703 | 1649639703 | 2434     | 2434     | 2434     | 0                 | 0             |
   | 2         | information_schema | proxyuser |                | 0x9B6FEA539C240940 | SELECT * FROM mydb.object                | 2          | 1649639631 | 1649639659 | 10003487 | 2632     | 10000855 | 0                 | 3             |
   | 1         | information_schema | proxyuser |                | 0x8666F6E344CEA089 | UPDATE mydb.object SET text=? WHERE id=? | 1          | 1649639713 | 1649639713 | 14668    | 14668    | 14668    | 1                 | 0             |
   | 2         | information_schema | proxyuser |                | 0xDDA8CA732A4BBD3B | SELECT * FROM object                     | 3          | 1649638655 | 1649639610 | 30001643 | 10000282 | 10001055 | 0                 | 0             |
   | 1         | information_schema | proxyuser |                | 0x226CD90D52A2BA0B | select @@version_comment limit ?         | 1          | 1649638628 | 1649638628 | 0        | 0        | 0        | 0                 | 0             |
   +-----------+--------------------+-----------+----------------+--------------------+------------------------------------------+------------+------------+------------+----------+----------+----------+-------------------+---------------+
   5 rows in set (0.00 sec)

   -- Other queries to run to see ProxySQL usage
   SELECT * FROM stats_mysql_commands_counters WHERE Total_cnt;
   SELECT * FROM stats.stats_mysql_connection_pool;
   ```

Congratulations! You have configured ProxySQL to transparently route our write workload to the source host and offload read queries to our replica.

## References

https://zzdjk6.medium.com/step-by-step-setup-gtid-based-mysql-replica-and-automatic-failover-with-mysqlfailover-using-docker-489489d2922
https://techcommunity.microsoft.com/t5/azure-database-for-mysql-blog/setting-up-proxysql-as-a-connection-pool-for-azure-database-for/ba-p/2589350
https://mydbops.wordpress.com/2018/02/19/proxysql-series-mysql-replication-read-write-split-up/
https://proxysql.com/documentation/proxysql-read-write-split-howto/
