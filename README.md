# double synchronization mysql and postgresql
This example contains real-time double synchronization between mysql and postgresql databases using kafka and debezium
```
# Install Dependencies
sudo apt-get -y install openjdk-21-jdk postgresql-server-dev-16 libprotobuf-c-dev
```
##### Debezium
###### Install Debezium mysql connector
```
wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/2.5.4.Final/debezium-connector-mysql-2.5.4.Final-plugin.tar.gz \
&& tar -xzf debezium-connector-mysql-2.5.4.Final-plugin.tar.gz \
&& sudo cp -r debezium-connector-mysql /usr/local/share/kafka/plugins
```
###### Install Debezium postgresql connector
```
wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/2.5.4.Final/debezium-connector-postgres-2.5.4.Final-plugin.tar.gz \
&& tar -xzf debezium-connector-postgres-2.5.4.Final-plugin.tar.gz \
&& sudo cp -r debezium-connector-postgres /usr/local/share/kafka/plugins
```
###### Install Debezium jdbc sink connector
```
wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-jdbc/2.5.4.Final/debezium-connector-jdbc-2.5.4.Final-plugin.tar.gz \
&& tar -xzf debezium-connector-jdbc-2.5.4.Final-plugin.tar.gz \
&& sudo cp -r debezium-connector-jdbc /usr/local/share/kafka/plugins
```
#### Kafka
###### Download and start kafka
```
wget https://dlcdn.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz \
&& tar -xzf kafka_2.13-3.7.0.tgz \
&& cd kafka_2.13-3.7.0 \
&& KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)" \
&& bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties \
&& bin/kafka-server-start.sh config/kraft/server.properties
```
```
# add plugins.path to config/connect-standalone.properties or config/connect-distributed.properties
bin/connect-standalone.sh config/connect-standalone.properties
```
##### Mysql
###### configuration to /etc/mysql/mysql.cnf
```
[mysqld]
server-id               = 184054
log_bin                 = mysql-bin
binlog_format           = ROW
expire_logs_days        = 10
binlog_row_image        = FULL
```
Restart mysql `sudo systemctl restart mysql`
##### Postgresql
###### Install PostgreSQL Decoding Plugins
```
git clone https://github.com/debezium/postgres-decoderbufs -b v{debezium-version} --single-branch \
&& cd postgres-decoderbufs \
&& export PATH=/usr/lib/postgresql/16/bin:$PATH \
&& sudo make && sudo make install \
&& cd .. \
&& rm -rf postgres-decoderbufs
```
###### configuration to /etc/postgresql/16/main/postgresql.conf
```
# MODULES
shared_preload_libraries = 'decoderbufs'

# REPLICATION
wal_level = logical             # minimal, archive, hot_standby, or logical (change requires restart)
max_wal_senders = 4             # max number of walsender processes (change requires restart)
#wal_keep_segments = 4          # in logfile segments, 16MB each; 0 disables
#wal_sender_timeout = 60s       # in milliseconds; 0 disables
max_replication_slots = 4       # max number of replication slots (change requires restart)
```
Restart postgresql `sudo systemctl restart postgresql`

##### Commands for manage synchronization
1. Start mysql connector and wait complete
```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @connect-mysql.json
```
2. Start postgresql main sink connector and wait complete
```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @sink-postgresql.json
```
3. Restart sequnces id
   ALTER SEQUENCE industries_id_seq RESTART WITH 1453;
5. Start postgresql connector with snapshot.mode=never
```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @connect-postgresql.json
```
5. Start mysql sink connector and wait complete
```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @sink-mysql.json
```
