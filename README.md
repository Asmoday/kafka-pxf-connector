# PXF Kafka plugin

[PXF](https://gpdb.docs.pivotal.io/6-8/pxf/overview_pxf.html) plugin to write data to Kafka topic.

## How to deploy and use

You should have working GPDB cluster with installed PXF.

Build project and deploy as described [here](https://gpdb.docs.pivotal.io/6-8/pxf/sdk/deploy_conn.html).

```shell script
cp <project>/pxf-kafka/env/pxf-profiles.xml $PXF_CONF/conf/
cp <project>/pxf-kafka/build/libs/pxf-kafka-*.jar $PXF_CONF/lib/
cp <project>/pxf-kafka/build/libs/kafka-clients-*.jar $PXF_CONF/lib/
pxf cluster sync
```
 
From GPDB SQL console try to execute:

```sql
CREATE WRITABLE EXTERNAL TABLE kafka_tbl (a TEXT, b TEXT, c TEXT)
  LOCATION ('pxf://<topic>?PROFILE=kafka&BOOTSTRAP_SERVERS=<server>')
  FORMAT 'CUSTOM' (FORMATTER='pxfwritable_export');

insert into kafka_tbl values ('a', 'b,c', 'd'), ('x', 'y', 'z');

drop external table kafka_tbl;
```

## How to debug locally

Build local GPDB docker image from [gpdb-pxf-docker project](https://github.com/arenadata/gpdb-pxf-docker).

```shell script
docker run -d --rm -v <project>:/home/gpadmin/pxf_demo -p 15432:6000 -p 35285:35285 --name gpdb-pxf-cluster gpdb-pxf
docker exec -it gpdb-pxf-cluster bash
```

```shell script
/initialize_cluster
sudo su - gpadmin
echo "host all gpadmin 0.0.0.0/0 trust" >> $MASTER_DATA_DIRECTORY/pg_hba.conf
gpstop -au
cp ~/pxf_demo/pxf-kafka/env/pxf-profiles.xml ~/pxf_conf/conf/
mkdir ~/pxf_conf/servers/demo
cp ~/pxf_demo/pxf-kafka/env/kafka-site.xml ~/pxf_conf/servers/demo/
cp ~/pxf_demo/pxf-kafka/build/libs/pxf-kafka-*.jar ~/pxf_conf/lib/
cp ~/pxf_demo/pxf-kafka/build/libs/kafka-clients-*.jar ~/pxf_conf/lib/
cp ~/pxf_demo/pxf-kafka/build/libs/paranamer-*.jar ~/pxf_conf/lib/
sed -i '1 a CATALINA_OPTS=-agentlib:jdwp=transport=dt_socket,address=35285,server=y,suspend=n' /usr/local/gpdb/pxf/pxf-service/bin/catalina.sh
pxf restart
```

```shell script
docker-compose -f <project>/pxf-kafka/env/docker-compose-kafka.yml up -d
docker exec -it kafka.local /opt/kafka/bin/kafka-topics.sh --zookeeper zk.local:2181 --create --topic demo --partitions 1 --replication-factor 1
docker exec -it kafka.local /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo --from-beginning
```

SQL statements to check (`postgresql://localhost:15432/postgres` `gpadmin/gpadmin`):
```sql
CREATE EXTENSION pxf;

-- kafka config from kafka-site.xml
CREATE WRITABLE EXTERNAL TABLE kafka_tbl (a TEXT, b TEXT, c TEXT)
  LOCATION ('pxf://demo?PROFILE=kafka&SERVER=demo')
  FORMAT 'CUSTOM' (FORMATTER='pxfwritable_export');
 
-- kafka config by options
CREATE WRITABLE EXTERNAL TABLE kafka_tbl (a TEXT, b TEXT, c TEXT)
  LOCATION ('pxf://demo?PROFILE=kafka&BOOTSTRAP_SERVERS=172.17.0.1:9094')
  FORMAT 'CUSTOM' (FORMATTER='pxfwritable_export');
 
insert into kafka_tbl values ('a', 'b,c', 'd'), ('x', 'y', 'z');

drop external table kafka_tbl;
```

Attach debugger to `localhost:35285`.
