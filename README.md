**Install Apache Kafka and Debezium Db2 Connector**

LOM:
TOOLS | VERSION
------|--------|
OS | Oracle Linux 8.5
Debezium | 2.2.1 Final
https://repo1.maven.org/maven2/io/debezium/debezium-connector-db2/2.2.1.Final/debezium-connector-db2-2.2.1.Final-plugin.tar.gz
JDK | 11
Apache Kafka|3.4.0
https://downloads.apache.org/kafka/3.4.0/kafka_2.13-3.4.0.tgz
db2 | docker image db2 11.5.8 (LUW Version)
jcc|11.5.0.0
https://repo1.maven.org/maven2/com/ibm/db2/jcc/11.5.8.0/jcc-11.5.8.0.jar

Note: Before start install steps, install Oracle Linux 8.5 then docker.

**Install Steps:**

1- Install JDK 11:
```
dnf install java-11-openjdk.x86_64 -y
```
2- Download Apache Kafka:
```
mkdir /app
cd /app
curl -O https://downloads.apache.org/kafka/3.4.0/kafka_2.13-3.4.0.tgz
tar xzfv kafka_2.13-3.4.0.tgz
mv kafka_2.13-3.4.0 kafka
```
3- Create plugins and connectors path:
```
mkdir /app/kafka/plugins
mkdir /app/kafka/connectors
```
4- Setup Plugin Path:
```
cp /app/kafka/config/connect-standalone.properties{,.bak}
echo "plugin.path=/app/kafka/plugins,/app/kafka/connectors" >> /app/kafka/config/connect-standalone.properties
```
Note: "plugins" path setup for plugins (for example: debezium connector and ...) and "connectors" setup for connectors config files.

5- Preapare Debezium DB2 connector as Kafka plugin:
```
curl -O https://repo1.maven.org/maven2/io/debezium/debezium-connector-db2/2.2.1.Final/debezium-connector-db2-2.2.1.Final-plugin.tar.gz
tar xzfv debezium-connector-db2-2.2.1.Final-plugin.tar.gz
mv debezium-connector-db2 debezium
mv debezium /app/kafka/plugins/
cd /app/kafka/plugins/debezium/
curl -O https://repo1.maven.org/maven2/com/ibm/db2/jcc/11.5.8.0/jcc-11.5.8.0.jar
```

6- Create Zookeeper Systemd Service:
```
cat <<EOF>/etc/systemd/system/zookeeper.service
[Unit]
Description=Apache Zookeeper server
Documentation=http://zookeeper.apache.org
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=simple
ExecStart=/usr/bin/bash /app/kafka/bin/zookeeper-server-start.sh /app/kafka/config/zookeeper.properties
ExecStop=/usr/bin/bash /app/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
EOF

```
7- Create Kafka Systemd Service:
```
cat <<EOF>/etc/systemd/system/kafka.service
[Unit]
Description=Apache Kafka Server
Documentation=http://kafka.apache.org/documentation.html
Requires=zookeeper.service

[Service]
Type=simple
Environment="JAVA_HOME=/usr/lib/jvm/jre-11-openjdk"
ExecStart=/usr/bin/bash /app/kafka/bin/kafka-server-start.sh /app/kafka/config/server.properties
ExecStop=/usr/bin/bash /app/kafka/bin/kafka-server-stop.sh

[Install]
WantedBy=multi-user.target
EOF
```

```
systemctl enable --now zookeeper
systemctl enable --now kafka
```
8- Create DB2 customized docker image for CDC process:
You can use Dockerfile for create customized db2 docker image.

```
tar xzfv db2-cdc-enabled.tar.gz
cd db2-cdc-enabled
docker build . -t ibmcom/db2-cdc-enabled:v1
```

9- Run db2 customized container:
donload db2
```
docker run -itd --name mydb2 --privileged=true -p 50000:50000 -e LICENSE=accept -e DB2INST1_PASSWORD=12345678 -e DBNAME=testdb -v /app/db2:/database ibmcom/db2-cdc-enabled:v1
```
10- Create db2 db2 debezium connector config file:
```
cat <<EOF> /app/kafka/connectors/db2.properties
tasks.max=1
name=my-db2-connector
connector.class=io.debezium.connector.db2.Db2Connector
database.hostname=localhost
database.port=50000
database.user=db2inst1
database.password=12345678
database.dbname=testdb
#    database.cdcschema=ASNCDC
#    topic.prefix=db2server
topic.prefix=DB2INST1
#    topic.prefix=DB2INST1
table.include.list=DB2INST1.*
#schema.history.internal.kafka.bootstrap.servers=debezium-cluster-kafka-bootstrap:9092
schema.history.internal.kafka.bootstrap.servers=localhost:9092
#    schema.history.internal.kafka.topic=schemahistory.DB2INST1
schema.history.internal.kafka.topic=schema-changes.inventory
EOF

```
11- Run Debezium DB2 Connector:
```
/app/kafka/bin/connect-standalone.sh config/connect-standalone.properties connectors/db2.properties
```

12- Test connector:

![image](https://github.com/IMAN-NAMJOOYAN/Install-Apache-Kafka-and-Debezium-Db2-Connector/assets/16554389/9b06c808-0616-4166-a64f-58a248f3d30b)


13- Test CDC:

```
docker exec -it <db2 container name or container id> bash
su - db2inst1
db2 "connect to testdb"
db2 "list tables"
```
![image](https://github.com/IMAN-NAMJOOYAN/Install-Apache-Kafka-and-Debezium-Db2-Connector/assets/16554389/f7f66fd1-3b00-4ad5-a758-ac8bbd956b54)

![image](https://github.com/IMAN-NAMJOOYAN/Install-Apache-Kafka-and-Debezium-Db2-Connector/assets/16554389/0a90dd9b-c83a-4b09-9d00-63fe5b62bea6)



#----------------------

Refrences:

[1] https://github.com/debezium/debezium-examples/tree/main/tutorial/debezium-db2-init/db2server

[2] https://debezium.io/documentation/reference/stable/connectors/db2.html

