# ClickHouse Kafka Connect Sink

## About

clickhouse-kafka-connect is the official Kafka Connect sink connector for [ClickHouse](https://clickhouse.com/).

The Kafka connector delivers data from a Kafka topic to a ClickHouse table.

## Documentation

See the [ClickHouse website](https://clickhouse.com/docs/en/integrations/kafka/clickhouse-kafka-connect-sink) for the full documentation entry.

## Design

For a full overview of the design and how exactly-once delivery semantics are achieved, see the [design document](./docs/DESIGN.md).

## Help

For additional help, please [file an issue in the repository](https://github.com/ClickHouse/clickhouse-kafka-connect/issues) or raise a question in [ClickHouse public Slack](https://clickhouse.com/slack).

# LoyaltyLion Fork

We've forked this project to build a connector compatible with Kafka 3.6.0+

## Flattening records

The official connector expects Debezium to publish changes in flattened JSON. This is required for ClickHouse to correctly
match fields to columns.

Debezium does not flatten records by default, and we're not doing so in our existing connector. We've added an option
on this fork to flatten records before pushing them to ClickHouse. To activate it, set the following property when creating
the ClickHouse connector:

```
"flattenRecords": "true"
```

### Deployment

1. `./gradlew createConfluentArchive`
2. `aws --profile terraform-cli s3 cp build/confluentArchive/clickhouse-kafka-connect-v1.1.2/lib/clickhouse-kafka-connect-v1.1.2-confluent.jar s3://loyaltylion-eu-west-1-devops/docker/clickhouse-kafka-connect-v1.1.2-$(date +"%Y%m%d%H%M")-confluent.jar`
3. Adjust `hogwarts/dockerfiles/kafka-connect-worker/Dockerfile` to download the new file.
4. Adjust `aws-terraform/modules/kafka/kafka-connect-ecs.tf` to deploy the new image.

### Local testing pushing to ClickHouse Cloud

1. Change `hogwarts/dockerfiles/kafka-connect-worker/Dockerfile` to copy the JAR file from the build folder.
2. Rebuild and relaunch your `kafka-connect` container:

```
docker compose stop kafka-connect \
  && docker compose rm -f kafka-connect \
  && docker compose build --no-cache kafka-connect \
  && docker compose up -d \
  && sleep 60 \
  && ./kafka/configure.rb \
  && docker compose logs -f kafka-connect
```

3. Connect to the VPN.
4. Go to https://kafka.loyaltylion.dev/ui/clusters/development/connectors
5. Make sure you have Debezium publishing changes to Kafka. Alternatively, create a topic for the table you want to test.
6. Create a ClickHouse connector with the following config:

```
{
	"connector.class": "com.clickhouse.kafka.connect.ClickHouseSinkConnector",
	"flattenRecords": "true",
	"tasks.max": "1",
	"clickhouseSettings": "date_time_input_format=best_effort",
	"ssl": "true",
	"security.protocol": "SSL",
	"ssl.truststore.location": "/tmp/kafka.client.truststore.jks",
	"port": "8443",
	"jdbcConnectionProperties": "?sslmode=STRICT",
	"value.converter.schemas.enable": "false",
	"value.converter": "org.apache.kafka.connect.json.JsonConverter",
	"exactlyOnce": "true",
    "schemas.enable": "false"
	"hostname": "wou4dumjny.eu-west-1.vpce.aws.clickhouse.cloud",
	"database": "lion_stage",
	"username": "loyaltylion",
    "password": "<get-it-from-1password>",
	"topics": "<table-topic>",
    "topic2TableMap": "<table-topic>=<_changes_table>",
}
```

6. If you have Debezium active, just make an update on your target table. If you're using a custom topic, publish an
   event to it.

### Sample event

```
{
	"before": {
		"id": 15920825,
		"site_id": 1669,
		"email": null,
		"merchant_id": "7149578387772",
		"properties": "{\"last_name\": \"test\", \"first_name\": \"test\"}",
		"auth_token": null,
		"points_approved": 100,
		"points_pending": 0,
		"points_spent": 0,
		"enrolled": false,
		"enrolled_at": null,
		"auth_log": "{}",
		"referred_by": null,
		"created_at": "2023-07-17T08:09:54.215000Z",
		"updated_at": "2023-10-06T10:04:36.996000Z",
		"referral_id": null,
		"metadata": "{\"tags\": [\"\"], \"newsletter\": {\"1669\": {\"signed_up\": false, \"first_known_sign_up_date\": null}}, \"shopify_source_url\": \"pawel-staging-07.myshopify.com\", \"auto_enroll_from_pos_with_email\": true}",
		"blocked": false,
		"guest": true,
		"registered_customer_id": null,
		"merchant_created_at": "2023-07-17T08:09:52.000000Z",
		"merchant_updated_at": "2023-10-06T10:04:34.000000Z",
		"linked_merchant_ids": null,
		"program_id": 1639,
		"legacy_id": null,
		"loyalty_tier_id": null,
		"birthday": null,
		"points_expired": null,
		"last_expired_points_at": null,
		"points_reserved": 0,
		"referral_location": null,
		"deleted_at": null,
		"expiry_basis": null,
		"expire_points_at": null,
		"instagram_username": null,
		"instagram_id": null,
		"instagram_id_updated_at": null,
		"history": "[]"
	},
	"after": {
		"id": 15920825,
		"site_id": 1669,
		"email": null,
		"merchant_id": "7149578387772",
		"properties": "{\"last_name\": \"test\", \"first_name\": \"test\"}",
		"auth_token": null,
		"points_approved": 100,
		"points_pending": 0,
		"points_spent": 0,
		"enrolled": false,
		"enrolled_at": null,
		"auth_log": "{}",
		"referred_by": null,
		"created_at": "2023-07-17T08:09:54.215000Z",
		"updated_at": "2024-06-17T17:04:15.902435Z",
		"referral_id": null,
		"metadata": "{\"tags\": [\"\"], \"newsletter\": {\"1669\": {\"signed_up\": false, \"first_known_sign_up_date\": null}}, \"shopify_source_url\": \"pawel-staging-07.myshopify.com\", \"auto_enroll_from_pos_with_email\": true}",
		"blocked": false,
		"guest": true,
		"registered_customer_id": null,
		"merchant_created_at": "2023-07-17T08:09:52.000000Z",
		"merchant_updated_at": "2023-10-06T10:04:34.000000Z",
		"linked_merchant_ids": null,
		"program_id": 1639,
		"legacy_id": null,
		"loyalty_tier_id": null,
		"birthday": null,
		"points_expired": null,
		"last_expired_points_at": null,
		"points_reserved": 0,
		"referral_location": null,
		"deleted_at": null,
		"expiry_basis": null,
		"expire_points_at": null,
		"instagram_username": null,
		"instagram_id": null,
		"instagram_id_updated_at": null,
		"history": "[]"
	},
	"source": {
		"version": "2.6.0.Final",
		"connector": "postgresql",
		"name": "dbz.lion_db",
		"ts_ms": 1718643855923,
		"snapshot": "false",
		"db": "lion_stage",
		"sequence": "[\"18543018007848\",\"18543018104528\"]",
		"ts_us": 1718643855923448,
		"ts_ns": 1718643855923448000,
		"schema": "public",
		"table": "customers",
		"txId": 1237793649,
		"lsn": 18543018104528,
		"xmin": null
	},
	"op": "u",
	"ts_ms": 1718643856323,
	"ts_us": 1718643856323844,
	"ts_ns": 1718643856323844600,
	"transaction": null
}
```
