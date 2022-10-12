## mysql-DB-replication

### Prerequisites
* Two VM and make password less authentication.

## Edit host file with ip address
```
192.168.143.16 masterdb.example.com masterdb	
192.168.143.17 slavedb.example.com slavedb
```

# On Master VM

## we need to install MySQL
```
yum -y install mariadb-server
```

## Then enable and start service on both machines.
```
systemctl enable mariadb
systemctl start mariadb
```

## Then open the firewall
```
firewall-cmd --permanent --add-service=mysql
firewall-cmd --reload
```

### Check the MySQL running port by using,
```		netstat -tnlp | grep -i 3306```

### Then do the secure installation of MySQL in both Vms,
```
mysql_secure_installation
```
		
```	
mysql -u root -p 
```

### Create easy access for hidden file for the root user

```	
  vi .my.cnf
	  [client]
	  user=root
	  password=root
```

### Create Database and Table
```	
mysql
  create database db1;
  show databases;
  use db1;
  create table country(name varchar(10));
  insert into user values('Sri-Lanka')
  flush tables with read lock;
  \q
```

### Now we need to take a dump of this database
```	
mysqldump db1 > db1.sql
scp db1.sql slavedb:/root
```

### ### Then again login to MySQL and need to unlock the tables.
```	
mysql
  unlock tables;
  \q
```

### Now we need to add replication configurations. For that we need to create a file added below details in that file.
```	
vi /etc/my.cnf
    server-id=1   —>  server id should less than for slave id
    log-bin=mysql-bin
```

### Then we need to restart the mariadb service
```	
systemctl restart mariadb
show master status;
```
### Now we need to enable replication.
```		grant replication slave on *.* to 'repl'@'192.168.252.129' identified by 'repl';```
		
### Provide privileges and check status
```
mysql
		flush the privileges;
		show master status;
		\q
		systemctl restart mariadb
		mysql -e 'show master status’;
```
## On Slave
```
	sudo yum -y install mariadb-server
	systemctl enable mariadb
	systemctl start mariadb
```
### Set root Password
```cmysql_secure_installation```
			- change password and others enter
		
```cmysql -u root -p``` 
    log using root password		
### Update vi .my.cnf file with credentials
```
vi .my.cnf
  [client]
  user=root 
  password=root
```	
### Create database on Slave machine
```
mysql -e 'create database db1'
mysql -e 'show databases'
mysql db1 < db1.sql
mysql -e ‘select * from db1.country’
```
### Update my.cnf file and restart(be careful with this, if there have any mistake then replication will not work properly)
```
vi /etc/my.cnf
  server-id=2
  replicate-wild-do-table=db1.%
```
### Restart database
```
systemctl restart mariadb
systemctl status mariadb
```
### Now we need to enable replication.
```
  mysql
		stop slave;
		change master to master_host='192.168.143.16',master_user=‘repl’,master_password='repl’,master_log_file='mysql-bin.000006’,master_log_pos=245;
		start slave;
		show slave states\G;
```
#### While executing this, if you faced access denied error you have to follow below steps.
```
mysql
  show grants;
  CREATE USER 'root'@'192.168.143.18' IDENTIFIED BY 'root';
  grant all privileges on *.* to 'root'@'192.168.143.18';
```

























