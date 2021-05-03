## Assignment

* presentation is available in `presentation.pdf`
* deployment is executed with  **Aiven CLI** scripts

## Aiven CLI

Install [Aiven CLI](https://github.com/aiven/aiven-client#getting-started), or use Docker:
```sh
docker build -t avn .
docker run -ti --rm avn bash
```

Login
```sh
avn  user login iradul@gmail.com
```

**Scripts below must be updated with appropriate desired/optimal setup.**

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

Create sink to write pricing/analitics to Postgres
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
