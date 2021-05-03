## Assignment

* presentation is available in [`presentation.pdf`](https://github.com/iradul/7E02F26B082C5F3BC7027F72335DD1186A2CD382/blob/master/presentation.pdf)
* deployment is executed with  **Aiven CLI** scripts

## Aiven CLI

Install [Aiven CLI](https://github.com/aiven/aiven-client#getting-started), or use Docker:
```sh
docker build -t avn .
docker run -ti --rm avn bash
```

Login
```sh
avn user login YOUR_EMAIL
```

**Scripts below must be updated with appropriate settings.**

### Create project

```sh
avn project create cgj --cloud google-us-east4
avn project switch cgj
```

### Postgres

Create Postgres
```sh
avn service create -t pg -p premium-32 -c "pg_version=13" --enable-termination-protection pg-avn
```

Migrate existing Database
```sh
# create a backup of existing DB
pg_dump -d 'postgres://OLD_USN:OLD_PWD@OLD_HOST_PORT/OLD_DATABASE?sslmode=require' --jobs $(nproc --all) --format directory -f DB_DUMP_DIR
# restore it to the new DB
pg_restore -d 'postgres://NEW_USN:NEW_PWD@NEW_HOST_PORT/NEW_DATABASE?sslmode=require' --jobs $(nproc --all) DB_DUMP_DIR
```

### Kafka

Create Kafka cluster (`premium` for Kafka Connect)
```sh
avn service create -t kafka -p premium-6x-16 -c "kafka_version=2.7" --enable-termination-protection kafka-avn
```

Create `in_pricing` inbound topic for pricing ingestion
```sh
avn service topic-create --partitions 3 --replication 3 --retention 604800000 --cleanup-policy delete kafka-avn in_pricing
```

Create sink to write pricing/analytics to Postgres **
```sh
# create sink config (this needs more work)
cat << 'EOF' >> kafka_jdbc_sink_pg.json
{
    "name": "pg-sink",
    "connector.class": "io.aiven.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:postgresql://HOST_PORT/DATABASE?sslmode=require",
    "connection.user": "USN",
    "connection.password": "PWD",
    "insert.mode": "insert",
    "pk.mode": "PRIMARY_KEY"
}
EOF

# create sink service
avn service connector create kafka-avn @kafka_jdbc_sink_pg.json
```

Create source connector for third party integration **
```
# create source config (this needs more work)
cat << 'EOF' >> kafka_jdbc_source_xyz.json
{
    "name": "pg-source_xyz",
    "connector.class": "io.aiven.connect.jdbc.JdbcSourceConnector",
    "connection.url": "CONNECT_URL",
    "connection.user": "USN",
    "connection.password": "PWD",
    "mode":"incrementing",
    "topic.prefix":"jdbc_source_pg_increment.",
    "poll.interval.ms":"5000",
    "timestamp.delay.interval.ms":"1000",
    "batch.max.rows":"1"
}
EOF
# create source connector service
avn service connector create kafka-avn @kafka_jdbc_source_xyz.json
```

## Notes

* The scripts are covering only the first stage of modernization.
* The scripts are not complete and should be used for demonstration purposes only, additional work is required for production setup.
* Flink deployment is not included.

