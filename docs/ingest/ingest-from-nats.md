---
id: ingest-from-nats
title: Ingest data from NATS JetStream
description: Ingest data from NATS JetStream into RisingWave.
slug: /ingest-from-nats
---
<head>
  <link rel="canonical" href="https://docs.risingwave.com/docs/current/ingest-from-nats/" />
</head>

You can ingest data from NATS JetStream into RisingWave by using the NATS source connector in RisingWave.

[NATS](https://nats.io/) is an open-source messaging system for cloud-native applications. It provides a lightweight publish-subscribe architecture for high-performance messaging.

[NATS JetStream](https://docs.nats.io/nats-concepts/jetstream) is a streaming data platform built on top of NATS. It enables real-time and historical access to streams of data via durable subscriptions and consumer groups.

:::info Public Preview
This feature is in the public preview stage, meaning it's nearing the final product but is not yet fully stable. If you encounter any issues or have feedback, please contact us through our [Slack channel](https://www.risingwave.com/slack). Your input is valuable in helping us improve the feature. For more information, see our [Public preview feature list](/product-lifecycle/#features-in-the-public-preview-stage).
:::

## Prerequisites

Before ingesting data from NATS JetStream into RisingWave, please ensure the following:

- The NATS JetStream server is running and accessible from your RisingWave cluster.
- If authentication is required for the NATS JetStream server, make sure you have the client username and password. The client user must have the `subscribe` permission for the subject.
- Create the NATS subject from which you want to ingest data.
- Ensure that your RisingWave cluster is running.

## Ingest data into RisingWave

When creating a source, you can choose to persist the data from the source in RisingWave by using `CREATE TABLE` instead of `CREATE SOURCE` and specifying the connection settings and data format.

### Syntax

```sql
CREATE { TABLE | SOURCE} [ IF NOT EXISTS ] source_name
[ schema_definition ]
WITH (
   connector='nats',
   server_url='<your nats server>:<port>', [ <another_server_url_if_available>, ...]
   subject='<subject>[,<another_subject...]',
   stream='<stream_name>',

-- optional parameters
   connect_mode='<connect_mode>',
   username='<your user name>',
   password='<your password>',
   jwt=`<your jwt>`,
   nkey=`<your nkey>`, ...

-- delivery parameters
   scan.startup.mode=`<startup_mode>`,
   scan.startup.timestamp.millis='xxxxx'
)
FORMAT PLAIN ENCODE data_encode;
```

**schema_definition**:

```sql
(
   column_name data_type [ PRIMARY KEY ], ...
   [ PRIMARY KEY ( column_name, ... ) ]
)
```

:::note

RisingWave performs primary key constraint checks on tables with connector settings but not on regular sources. If you need the checks to be performed, please create a table with connector settings.

For a table with primary key constraints, if a new data record with an existing key comes in, the new record will overwrite the existing record.
:::

:::note

According to the [NATS documentation](https://docs.nats.io/running-a-nats-service/nats_admin/jetstream_admin/naming), stream names must adhere to subject naming rules as well as being friendly to the file system. Here are the recommended guidelines for stream names:

- Use alphanumeric values.
- Avoid spaces, tabs, periods (`.`), greater than (`>`) or asterisks (`*`).
- Do not include path separators (forward slash or backward slash).
- Keep the name length limited to 32 characters as the JetStream storage directories include the account, stream name, and consumer name.
- Avoid using reserved file names like `NUL` or `LPT1`.
- Be cautious of case sensitivity in file systems. To prevent collisions, ensure that stream or account names do not clash due to case differences. For example, `Foo` and `foo` would collide on Windows or macOS systems.

:::

### Parameters

|Field|Notes|
|---|---|
|`server_url`| Required. URLs of the NATS JetStream server, in the format of *address*:*port*. If multiple addresses are specified, use commas to separate them.|
|`subject`| Required. NATS subject that you want to ingest data from. To specify more than one subjects, use a comma.|
|`stream` | Required. NATS stream that you want to ingest data from.|
|`connect_mode`|Required. Authentication mode for the connection. Allowed values: <ul><li>`plain`: No authentication. </li><li>`user_and_password`: Use user name and password for authentication. For this option, `username` and `password` must be specified.</li><li> `credential`: Use JSON Web Token (JWT) and NKeys for authentication. For this option, `jwt` and `nkey` must be specified.</li></ul> |
|`jwt` and `nkey`|JWT and NKEY for authentication. For details, see [JWT](https://docs.nats.io/running-a-nats-service/configuration/securing_nats/auth_intro/jwt) and [NKeys](https://docs.nats.io/running-a-nats-service/configuration/securing_nats/auth_intro/nkey_auth).|
|`username` and `password`| Conditional. The client user name and password. Required when `connect_mode` is `user_and_password`.|
|`scan.startup.mode`|Optional. The offset mode that RisingWave will use to consume data. The supported modes are: <ul><li>`earliest`: Consume from the earliest available message, corresponding to [deliver policy] `DeliverAll`.</li><li>`latest`: Consume from the next message, corresponding to `DeliverNew` policy.</li><li>`timestamp`: Consume from a particular UNIX timestamp specified via `scan.startup.timestamp.millis`, corresponding to `DeliverByStartTime` policy.</li></ul>If not specified, the default value `earliest` will be used.|
|`scan.startup.timestamp.millis`|Conditional. Required when `scan.startup.mode` is `timestamp`. RisingWave will start to consume data from the specified UNIX timestamp.
|`data_encode`| Supported encodes: `JSON`, `PROTOBUF`, `BYTES`. |
| `consumer.deliver_subject` | Optional. Subject to deliver messages to. |
| `consumer.durable_name` | Required. Durable name for the consumer. |
| `consumer.name` | Optional. Name of the consumer. |
| `consumer.description` | Optional. Description of the consumer. |
| `consumer.deliver_policy` | Optional. Policy on how messages are delivered. |
| `consumer.ack_policy` | Optional. Acknowledgment policy for message processing (e.g., `None`, `All`, `Explicit`). |
| `consumer.ack_wait.sec` | Optional. Time to wait for acknowledgment before considering a message as undelivered. |
| `consumer.max_deliver` | Optional. Maximum number of times a message will be delivered. |
| `consumer.filter_subject` | Optional. Filter for subjects that the consumer will process. |
| `consumer.filter_subjects` | Optional. List of subjects that the consumer will filter on. |
| `consumer.replay_policy` | Optional. Policy for replaying messages (e.g., `Instant`, `Original`). |
| `consumer.rate_limit` | Optional. Rate limit for message delivery in bits per second. |
| `consumer.sample_frequency` | Optional. Frequency for sampling messages, ranging from 0 to 100. |
| `consumer.max_waiting` | Optional. Maximum number of messages that can be waiting for acknowledgment. |
| `consumer.max_ack_pending` | Optional. Maximum number of acknowledgments that can be pending. |
| `consumer.headers_only` | Optional. If true, only message headers will be delivered. |
| `consumer.max_batch` | Optional. Maximum number of messages to process in a single batch. |
| `consumer.max_bytes` | Optional. Maximum number of bytes to receive in a single batch. |
| `consumer.max_expires.sec` | Optional. Maximum expiration time for a message in seconds. |
| `consumer.inactive_threshold.sec` | Optional. Time in seconds before a consumer is considered inactive. |
| `consumer.num.replicas` | Optional. Number of replicas for the consumer. |
| `consumer.memory_storage` | Optional. If true, messages will be stored in memory. |
| `consumer.backoff.sec` | Optional. Backoff time in seconds for retrying message delivery. |

[deliver policy]: https://docs.nats.io/nats-concepts/jetstream/consumers#deliverpolicy

## Examples

The following SQL query creates a table that ingests data from a NATS JetStream source.

```sql
CREATE TABLE live_stream_metrics
WITH
  (
    connector = 'nats',
    server_url = 'nats-server:4222',
    subject = 'live_stream_metrics',
    stream = 'risingwave',
    connect_mode = 'plain'
  ) FORMAT PLAIN ENCODE PROTOBUF (
    message = 'livestream.schema.LiveStreamMetrics',
    schema.location = 'http://file_server:8080/schema'
  );
```

The parameters supported by the [`async_nats`](https://docs.rs/async-nats/latest/async_nats/jetstream/consumer/struct.Config.html) crate are all supported in the RisingWave NATS source connector.

```sql
CREATE SOURCE test_source
WITH (
    connector='nats',
    server_url='{{ env_var("SERVER") }}',
    subject='risingwave.test.source',
    stream='risingwave-test-source',
    scan.startup.mode='earliest',
    connect_mode='user_and_password',
    username='{{ env_var("USER") }}',
    password='{{ env_var("PASSWORD") }}',
    consumer.durable_name='risingwave-test-source',
    consumer.description='desc-test-source',
    consumer.ack_policy='all',
    consumer.ack_wait=10,
    consumer.max_deliver=10,
    consumer.filter_subjects='demo.subject.filter.*',
    consumer.filter_subjects='demo.subject.filter.1,demo.subject.filter.2',
    consumer.replay_policy='instant',
    consumer.rate_limit=100000000000,
    consumer.sample_frequency=100,
    consumer.max_waiting=10,
    consumer.max_ack_pending=10,
    -- consumer.idle_heartbeat=60, not available in async_nats crate
    consumer.max_batch=1000,
    consumer.max_bytes=1000000000,
    consumer.max_expires=3600,
    consumer.inactive_threshold=10000000,
    consumer.memory_storage='false',
    consumer.backoff='10,30,60',
    consumer.num_replicas=1
) FORMAT PLAIN ENCODE JSON;
```