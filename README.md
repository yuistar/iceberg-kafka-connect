# Iceberg Kafka Connect Example

This example demonstrates how to use Kafka Connect to write to a Iceberg table with Trino.
https://substack.com/home/post/p-137733070?source=queue
![](/Users/yhuang/Desktop/kafka-connect-iceberg-trino.png)

## Starting the environment

### Minio

This is a S3 compatible storage that is used to store the iceberg tables.
https://blog.min.io/a-developers-introduction-to-apache-iceberg-using-minio/
It has a UI that can be accessed at `http://localhost:9000` it will look very similar to S3

the login credentials (configured in the `docker-compose.yml` file) are:

username: `admin` | password: `password`

In the UI, in the user browser, you should see 1 bucket called `demo-iceberg`


### Iceberg Rest Catalog

This is a rest catalog that is used to manage/store meta information about the iceberg tables.
it is backed by a `postgres` database.

There are other alternatives, such as: `nessie`, `hivemetastore` etc


### Trino

This will mount the `iceberg.properties` file into `/etc/trino/catalog/`  in the trino container.
This file is a `catalog` config used to configure the trino iceberg connector. 

Trino looks for catalogs in `/etc/trino/catalog/`.

There is a trino UI at `http://localhost:8080` the login is `admin`


### Kafka

Run this in a separate terminal, as it will run in the foreground,
so we can easily look the Kafka Connect Logs

- Three Kafka brokers running in KRaft mode
- Schema Registry + UI
- Kafka Connect + UI

The Kafka Connect image is built from the `Dockerfile` to have the Iceberg Sink Connector


## Create the Iceberg table

You can connect to trino in any way you like. Here is an example using the trino-cli.
see https://trino.io/docs/current/client/cli.html

```bash
trino http://localhost:8080
# you can verify the connection by running `show catalogs;`
```

first the db schema
```sql
create schema iceberg.blockchain;
```

then the table

```sql
create table iceberg.blockchain.ethereum_mainnet_blocks(
        number BIGINT,
        hash VARCHAR,
        parent_hash VARCHAR,
        gas_used BIGINT,
        timestamp TIMESTAMP(6)
)
WITH (
    partitioning = ARRAY['day(timestamp)'],
    sorted_by = ARRAY['number']
    );
```
you can check the table by running `show tables from iceberg.blockchain;`

exit from the trino-cli by running `exit;`

### Generating the data

Create the topic `ethereum_mainnet_blocks` and start the producer
```shell
kafka-topics --bootstrap-server localhost:9092 --topic ethereum.mainnet.blocks --partitions 3 --replication-factor 1 --create
```

```bash
go run main.go
```

## Adding the Kafka Connector
Open the Kafka Connect UI on `http://localhost:8000` and click on `New` to add a new connector.

You should see `IcebergSinkConnector` in the list of available connectors.

paste in values from `connector.json` and click `Create`