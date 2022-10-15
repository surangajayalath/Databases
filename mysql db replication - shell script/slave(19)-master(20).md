```
#!/bin/bash

sudo sed -i 's/server-id=2/server-id=1/g' /etc/my.cnf
sudo sed -i 's/#log-bin=mysql-bin/log-bin=mysql-bin/g' /etc/my.cnf
sudo sed -i 's/replicate-wild-do-table=heimdall.%/#replicate-wild-do-table=heimdall.%/g' /etc/my.cnf
sudo sed -i 's/replicate-wild-do-table=bifrost.%/#replicate-wild-do-table=bifrost.%/g' /etc/my.cnf


#sudo systemctl restart mariadb
#sudo systemctl status mariadb

ssh osssupport@192.168.143.19 sudo sed -i 's/server-id=1/server-id=2/g' /etc/my.cnf
ssh osssupport@192.168.143.19 sudo sed -i 's/log-bin=mysql-bin/#log-bin=mysql-bin/g' /etc/my.cnf
ssh osssupport@192.168.143.19 sudo sed -i 's/#replicate-wild-do-table=heimdall.%/replicate-wild-do-table=heimdall.%/g' /etc/my.cnf
ssh osssupport@192.168.143.19 sudo sed -i 's/#replicate-wild-do-table=bifrost.%/replicate-wild-do-table=bifrost.%/g' /etc/my.cnf

#ssh osssupport@192.168.143.19 sudo systemctl restart mariadb
#ssh osssupport@192.168.143.19 sudo status mariadb

x=1
a=heimdall
b=bifrost


while [[ $x -le 2 ]]; do
	if [[ $x -eq 1 ]]; then
		c=$a
	elif [[ $x -eq 2 ]]; then
		c=$b
	fi


DB=$c


DUMP_FILE="/apps/backups/dump/$DB-export-$(date +"%Y%m%d%H%M%S").sql"

USER=ifd
PASS=XYtBx9NbPG0E

MASTER_HOST=192.168.143.20
SLAVE_HOSTS=192.168.143.19


RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color

printf "${GREEN} Master ---> DB1    ip: ${MASTER_HOST} ${NC} \n"
printf "${GREEN} Slave  ---> DB2 ip: ${SLAVE_HOSTS} ${NC} \n"

# Dump the blacktown database
echo "--------------------------------------- Dumping DB2 ------------------------------"

##
# MASTER
# ------
# Export database and read log position from master, while locked
##

printf "${GREEN} Master ---> Mascot    ip: ${MASTER_HOST} ${NC} \n"

mysql -h $MASTER_HOST "-u$USER" "-p$PASS" $DB <<-EOSQL &
	GRANT REPLICATION
 SLAVE ON *.* TO '$USER'@'%' IDENTIFIED BY '$PASS';
	FLUSH PRIVILEGES;
	FLUSH TABLES WITH READ LOCK;
	DO SLEEP(3600);
EOSQL

echo "  - Waiting for database to be locked"
sleep 3

# Dump the database (to the client executing this script) while it is locked
printf "${RED} - Dumping database to ${DUMP_FILE} ${NC} \n"
mysqldump -h $MASTER_HOST "-u$USER" "-p$PASS" --opt $DB > $DUMP_FILE

echo "------------------------------------------ Dump complete ----------------------------------------"

# Take note of the master log position at the time of dump
MASTER_STATUS=$(mysql -h $MASTER_HOST "-u$USER" "-p$PASS" -ANe "SHOW MASTER STATUS;" | awk '{print $1 " " $2}')
LOG_FILE=$(echo $MASTER_STATUS | cut -f1 -d ' ')
LOG_POS=$(echo $MASTER_STATUS | cut -f2 -d ' ')

echo "-------------------------------------------------------------------------------------------------"
echo "  - Current log file is $LOG_FILE"
echo "  - Current log position is $LOG_POS"

printf "${GREEN} Current log file is ${LOG_FILE} ${NC} \n"
printf "${GREEN} Current log position is ${LOG_POS} ${NC} \n"

# When finished, kill the background locking command to unlock
kill $! 2>/dev/null
wait $! 2>/dev/null

echo "  - Master database unlocked"


##
# SLAVES
# ------
# Import the dump into slaves and activate replication with
# binary log file and log position obtained from master.
##

echo "------------------------------------- Import data into slave ------------------------------------"

printf "${GREEN} Master ---> Mascot ip: ${SLAVE_HOST} ${NC} \n"


for SLAVE_HOST in "${SLAVE_HOSTS[@]}"
do
	echo "SLAVE: $SLAVE_HOST"
	echo "  - Creating database copy"
	
	mysql -h $SLAVE_HOST "-u$USER" "-p$PASS" -e "DROP DATABASE IF EXISTS $DB; CREATE DATABASE $DB;"
	scp $DUMP_FILE $SLAVE_HOST:$DUMP_FILE >/dev/null
	mysql -h $SLAVE_HOST "-u$USER" "-p$PASS" $DB < $DUMP_FILE

	
echo "-------------------------------- Setting up slave replication -----------------------------------"

	mysql -h $SLAVE_HOST "-u$USER" "-p$PASS" $DB <<-EOSQL &
		STOP SLAVE;
		CHANGE MASTER TO MASTER_HOST='192.168.143.19',
		MASTER_USER='$USER',
		MASTER_PASSWORD='$PASS',
		MASTER_LOG_FILE='$LOG_FILE',
		MASTER_LOG_POS=$LOG_POS;
		START SLAVE;
	EOSQL
	# Wait for slave to get started and have the correct status
	sleep 2

	SLAVE_OK=$(mysql -h $SLAVE_HOST "-u$USER" "-p$PASS" -e "SHOW SLAVE STATUS\G;" | grep 'Waiting for master')
	if [ -z "$SLAVE_OK" ]; then
		echo "  - Error ! Wrong slave IO state."
	else
echo "------------------------- Database Replication Completed Successfully ---------------------------"
	fi

	
done

x=$(( $x + 1 ));
	
done
```
