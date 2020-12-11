## Kafka Connect for MS SQL Server and PostgreSQL

Stream rows from DataBase table to topic Kafka via Kafka Connect (used JdbcSourceConnector)

[Kafka Connect](https://kafka.apache.org/documentation/#connect)

[Kafka Connect JDBC](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc)

[JDBC Connector Source Connector Configuration Properties](https://docs.confluent.io/current/connect/kafka-connect-jdbc/source-connector/source_config_options.html#jdbc-source-configs)

[Best article about Kafka Connect](https://www.confluent.io/blog/kafka-connect-deep-dive-jdbc-source-connector/)

Sample DB table
```sqlserver
CREATE TABLE test
(
    id    BIGINT IDENTITY (1,1) PRIMARY KEY NOT NULL,
    value VARCHAR(255)                      NOT NULL,
    date  DATETIME2                         NOT NULL
);

--don't forget create index for best performance on big data
CREATE INDEX inx_test ON test (date);

DECLARE @i int = 0
WHILE @i < 100000
    BEGIN
        SET @i = @i + 1
        INSERT INTO test (value, date) VALUES ('Testing ' + CAST(@i AS VARCHAR), SYSDATETIME());
    END

--
select count(*) from test;
```

```postgresql
SET search_path = calculator;

CREATE TABLE test
(
    id    SERIAL PRIMARY KEY          NOT NULL,
    value VARCHAR(255)                NOT NULL,
    date  TIMESTAMP(3) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

--don't forget create index for best performance on big data
CREATE INDEX inx_test ON test (date);

do
$$
    begin
        for r in 1..1000
            loop
                PERFORM pg_sleep(0.01);

                INSERT INTO test (value) VALUES ('Testing');
                commit;
                --raise notice 'insert: %', r;
            end loop;
    end;
$$;

--
select count(*) from test;
```

See
1. config /kafka_2.13-2.6.0/config/connect-standalone-jdbc.properties. Look at property `plugin.path`.
1. config /kafka_2.13-2.6.0/config/connect-mssql-source.properties
1. config /kafka_2.13-2.6.0/config/connect-postgres-source.properties
1. JDBC Connector plugin folder /kafka_2.13-2.6.0/libs/jdbc-plugin
1. MS SQL Server JDBC-driver /kafka_2.13-2.6.0/libs/jdbc-plugin/mssql-jdbc-8.4.1.jre8.jar
1. PostgreSQL JDBC-driver /kafka_2.13-2.6.0/libs/jdbc-plugin/postgresql-42.2.10.jar
1. config /kafka_2.13-2.6.0/config/connect-log4j.properties to set DEBUG

```ssh
cd kafka_2.13-2.6.0

# Step 1. Run Zookepeer (execute in Console 1) 
bin/zookeeper-server-start.sh config/zookeeper.properties

# Step 2. Run Kafka (execute in Console 2)
bin/kafka-server-start.sh config/server.properties
```

Connect to Kafka via [Kafka Tool](https://www.kafkatool.com/) - GUI application for managing and using Apache Kafka clusters.

***
# Standalone Mode. mode=incrementing

```ssh
# run Kafka Connect for MS SQL Server (execute in Console 3)
CLASSPATH=/Users/u16713891/Documents/projects/GitHub/kafka-connect-jdbc/kafka_2.13-2.6.0/libs/jdbc-plugin/mssql-jdbc-8.4.1.jre8.jar ./bin/connect-standalone.sh config/connect-standalone-jdbc.properties ./config/connect-mssql-source.properties

# read topic from MS SQL Server via Kafka Connect (execute in Console 4)
bin/kafka-console-consumer.sh --topic mssql_test --from-beginning --bootstrap-server localhost:9092

# try to add another row in DB table and check streaming
```

```ssh
# run Kafka Connect for PostgreSQL (execute in Console 3)
CLASSPATH=/Users/u16713891/Documents/projects/GitHub/kafka-connect-jdbc/kafka_2.13-2.6.0/libs/jdbc-plugin/postgresql-42.2.10.jar ./bin/connect-standalone.sh config/connect-standalone-jdbc.properties ./config/connect-postgres-source.properties

# read topic from PostgreSQL via Kafka Connect (execute in Console 4)
bin/kafka-console-consumer.sh --topic postgres_test --from-beginning --bootstrap-server localhost:9092

# try to add another row in DB table and check streaming
```

```ssh
# see all Kafka topics
bin/kafka-topics.sh --zookeeper localhost --describe

# clean Kafka logs
rm -rf /tmp/kafka-logs /tmp/zookeeper
```

***

[REST API](https://kafka.apache.org/documentation/#connect_rest)
[Monitoring Kafka Connect and Connectors](https://docs.confluent.io/current/connect/managing/monitoring.html)

```ssh
# usefull util to view formatted JSON in console
brew install jq

curl -X GET http://localhost:8083/ | jq
curl -X GET http://localhost:8083/connectors | jq
curl -X GET http://localhost:8083/connectors/postgres-connector | jq

# should be "state":"RUNNING". If "state":"FAILED" - try restart
curl -X GET http://localhost:8083/connectors/postgres-connector/status | jq
curl -X POST http://localhost:8083/connectors/postgres-connector/restart
curl -X GET http://localhost:8083/connectors/postgres-connector/topics | jq
curl -X PUT http://localhost:8083/connectors/postgres-connector/topics/reset
curl -X GET http://localhost:8083/connectors/postgres-connector/config | jq
# to pause streaming
curl -X PUT http://localhost:8083/connectors/postgres-connector/pause
# to resume streaming
curl -X PUT http://localhost:8083/connectors/postgres-connector/resume
# to find task id
curl -X GET http://localhost:8083/connectors/postgres-connector/tasks | jq
# to check task with id 0
curl -X GET http://localhost:8083/connectors/postgres-connector/tasks/0/status | jq
# to restart task with id 0 (usefull if task failed)
curl -X POST http://localhost:8083/connectors/postgres-connector/tasks/0/restart
curl -X GET http://localhost:8083/connector-plugins | jq
```

How view "Number of records output from the transformations and written to Kafka for the task belonging to the named source connector in the worker (since the task was last restarted)"?
Connect via jconsole to process and check MBeans: 
kafka.connect/source-task-metrics/.../0/source-record-write-total

How view "The average per-second number of records output from the transformations and written to Kafka for this task belonging to the named source connector in this worker. This is after transformations are applied and excludes any records filtered out by the transformations."?
Check MBeans:
kafka.connect/source-task-metrics/.../0/source-record-write-rate

How view "The total number of records sent for a topic"?
Check MBeans:
kafka.producer/producer-topic-metrics/.../record-send-total


***
# Distributed Mode. mode=timestamp

See
1. /kafka-connect-jdbc/kafka_2.13-2.6.0/config/connect-distributed-jdbc.properties
1. /kafka-connect-jdbc/kafka_2.13-2.6.0/config/connect-mssql-source.json
1. /kafka-connect-jdbc/kafka_2.13-2.6.0/config/connect-postgres-source.json

Note, very important!
See at property `connection.url`, additional connection properties for MSSQL Server `selectMethod=cursor;responseBuffering=adaptive`. [Using adaptive buffering](https://docs.microsoft.com/en-us/sql/connect/jdbc/using-adaptive-buffering?view=sql-server-ver15)
1. Property `selectMethod` needs for processing large DB columns (30KB and much more) and iteration streaming data from DB, non-stop.
1. Property `responseBuffering` for correct Java Memory utilization (to avoid OutOfMemoryError Exception). For example, java 11 memory usage in range 25-225MB.

Performance - 50 TPS

Use https://www.epochconverter.com/ for convert timestamp (--topic connect-offsets) to date and check streaming state.

MS SQL Server (tested on Microsoft SQL Server 2016)
```ssh
CLASSPATH=/Users/u16713891/Documents/projects/GitHub/kafka-connect-jdbc/kafka_2.13-2.6.0/libs/jdbc-plugin/mssql-jdbc-8.4.1.jre8.jar ./bin/connect-distributed.sh config/connect-distributed-jdbc.properties

curl -X POST -H "Content-Type: application/json" --data @config/connect-mssql-source.json http://localhost:8083/connectors/ | jq

curl -X GET http://localhost:8083/connectors/mssql-connector/status | jq

curl -X GET http://localhost:8083/connectors/mssql-connector/tasks/0/status | jq

bin/kafka-console-consumer.sh --topic mssql_test --from-beginning --bootstrap-server localhost:9092
bin/kafka-console-consumer.sh --topic test-errors --from-beginning --bootstrap-server localhost:9092
bin/kafka-console-consumer.sh --topic connect-status --from-beginning --bootstrap-server localhost:9092 | jq
bin/kafka-console-consumer.sh --topic connect-offsets --from-beginning --bootstrap-server localhost:9092 | jq
bin/kafka-console-consumer.sh --topic connect-configs --from-beginning --bootstrap-server localhost:9092 | jq

curl -X DELETE http://localhost:8083/connectors/mssql-connector/
```

PostgreSQL
```ssh
CLASSPATH=/Users/u16713891/Documents/projects/GitHub/kafka-connect-jdbc/kafka_2.13-2.6.0/libs/jdbc-plugin/postgresql-42.2.10.jar ./bin/connect-distributed.sh config/connect-distributed-jdbc.properties

curl -X POST -H "Content-Type: application/json" --data @config/connect-postgres-source.json http://localhost:8083/connectors/ | jq

curl -X GET http://localhost:8083/connectors/postgres-connector/status | jq

curl -X DELETE http://localhost:8083/connectors/postgres-connector/
```

Running Kafka Connect on Windows. After starting Kafka Connect, ignore any exceptions about ClassNotFoundException - it's ok. Just check messages in Kafka.
```
set CLASSPATH=d:\kafka_2.13-2.6.0\libs\jdbc-plugin\mssql-jdbc-8.4.1.jre8.jar
d:\kafka_2.13-2.6.0\bin\windows\connect-distributed.bat d:\kafka_2.13-2.6.0\config\connect-distributed-jdbc.properties
pause
```
