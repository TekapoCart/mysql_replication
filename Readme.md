
### traditional way
source .env

MASTER_IP=$(docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mysql_replication_master_1)

### Master
docker exec -i mysql_replication_master_1 mysql -uroot -p${MY_ROOT_PASSWORD} <<< "GRANT REPLICATION SLAVE ON *.* TO '${MY_USER}'@'%'; FLUSH PRIVILEGES;"
docker exec -i mysql_replication_master_1 mysql -uroot -p${MY_ROOT_PASSWORD} <<< "SHOW MASTER STATUS"

MY_STATUS=$(docker exec -i mysql_replication_master_1 mysql -uroot -p${MY_ROOT_PASSWORD} <<< "SHOW MASTER STATUS")
# binlog 文件名字, 對應 File 字段, 值如：my.000003
CURRENT_LOG=`echo $MY_STATUS | awk '{print $6}'`
# binlog 位置, 對應 Position 字段, 值如：156
CURRENT_POS=`echo $MY_STATUS | awk '{print $7}'`

### Slave
SLAVE_SQL="CHANGE MASTER TO MASTER_HOST='${MASTER_IP}', MASTER_USER='${MY_USER}', MASTER_PASSWORD='${MY_PASSWORD}',
        MASTER_LOG_FILE='${CURRENT_LOG}', MASTER_LOG_POS=${CURRENT_POS};"
docker exec -i mysql_replication_slave_1 mysql -uroot -p${MY_ROOT_PASSWORD} <<< "${SLAVE_SQL}"
docker exec -i mysql_replication_slave_1 mysql -uroot -p${MY_ROOT_PASSWORD} <<< "START SLAVE;"
docker exec -i mysql_replication_slave_1 mysql -uroot -p${MY_ROOT_PASSWORD} <<< "SHOW SLAVE STATUS \G"

### Test

docker exec -i mysql_replication_master_1 mysql -uroot -p${MY_ROOT_PASSWORD} ${MY_DATABASE} <<< "CREATE TABLE code(code int); INSERT INTO code VALUES (100), (200)"
docker exec -i mysql_replication_slave_1 mysql -uroot -p${MY_ROOT_PASSWORD} ${MY_DATABASE} <<< "SELECT * FROM code \G"


### gtid_mode
https://github.com/chanjarster/mysql-master-slave-docker-example
