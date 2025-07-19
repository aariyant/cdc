# Kafka + Schema Registry + Debezium Tutorial

This tutorial will guide you through running and verifying a Kafka-based CDC (Change Data Capture) pipeline using **Kafka**, **Schema Registry**, and **Debezium**, all orchestrated via Docker Compose.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Structure](#project-structure)
3. [Step-by-Step Setup](#step-by-step-setup)
4. [Debezium Connector](#debezium-connector)
5. [Debezium Connector Offsets](#debezium-connector-offsets)
6. [CDC Verification Example](#cdc-verification-example)
7. [Cleanup](#cleanup)
8. [Troubleshooting Tips](#troubleshooting-tips)
9. [Notes](#notes)

---

## Prerequisites

- Docker: Required to run each component in isolated containers.
- Docker Compose: Simplifies the orchestration of all required services.
- Basic understanding of Kafka and Debezium: Helps understand how messages are streamed and managed via connectors.

---

## Project Structure

This layout outlines what each file/folder is used for in the tutorial.

```
.
├── db-compose
│   └── docker-compose.yaml
├── debezium-compose
│   ├── docker-compose.yml
│   ├── data/
│   │   └── kafka-data/
│   ├── config/
│   │   └── kafka-ui/
│   │       └── kafka-ui-config.yaml
```

---

## Step-by-Step Setup Debezium

Follow these steps in order to bring up the services. Each service should be started and verified before moving to the next to ensure proper communication and dependencies are resolved.

---

### 1. Start Zookeeper

Zookeeper is a required coordination service that Kafka depends on.

```bash
docker-compose up -d zookeeper
```

Verify that Zookeeper has started:

```bash
docker logs -f zookeeper
```

You should see a line similar to:

```
binding to port 0.0.0.0/0.0.0.0:2181
```

---

### 2. Start Kafka Broker

The Kafka broker handles message queues, topics, producers, and consumers.

```bash
docker-compose up -d broker
```

Check Kafka logs to confirm it started correctly:

```bash
docker logs -f broker
```

Look for a message like:

```
started (kafka.server.KafkaServer)
```

---

### 3. Kafka UI (Optional)

Launches a web-based interface to view and interact with Kafka topics and consumer groups.

```bash
docker-compose up -d kafka-ui
```

Access the UI via: [http://localhost:8180](http://localhost:8180)

---

### 4. Start Schema Registry

Schema Registry manages Avro, Protobuf, or JSON schemas used by Kafka producers and consumers.

```bash
docker-compose up -d schema-registry
```

Check it’s working:

```bash
curl http://localhost:8081/subjects
```

If no schemas exist yet, it should return:

```
[]
```

---

### 5. Kafka REST Proxy (Optional)

Enables HTTP access to Kafka, allowing REST clients to interact with Kafka topics.

```bash
docker-compose up -d rest-proxy
```

Test if it works:

```bash
curl http://localhost:8082/topics
```

---

### 6. Start Debezium Connect

This starts the Kafka Connect service with Debezium plugins for capturing database changes.

```bash
docker-compose up -d debezium
```

Confirm it's ready to register connectors:

```bash
curl http://localhost:8083/connectors
```

It should return:

```
[]
```

---

### 7. Debezium UI (Optional)

A visual interface to create and manage Debezium connectors without using raw REST API.

```bash
docker-compose up -d debezium-ui
```

Access it at: [http://localhost:8080](http://localhost:8080)

---

## Check Container Status

You can verify the state of all containers using:

```bash
docker-compose ps
```

Make sure all required services are listed and running.

---

## Step-by-Step Setup: Database

### 1. Start Databases (MariaDB & PostgreSQL)

To test the CDC (Change Data Capture) pipeline, this tutorial uses **MariaDB** as the **source** and **PostgreSQL** as the **target**.

Start all required services (including both databases):

```bash
docker-compose up -d
```

This will launch:
- MariaDB (source connector target)
- PostgreSQL (sink connector target)
- Other services defined in `docker-compose.yml`

---

### 2. Import Sample Data into MariaDB

We’ll use the classic `classicmodels` sample database as the data source. Make sure the `classicmodels.sql` file is located in your project directory.

Run the following command to import it into the MariaDB container:

```bash
mysql -u root -pmysql-pass -P 4406 -h 127.0.0.1 < classicmodels.sql
```

**Notes**:
- Adjust the `-P` port and `-p` password if they differ from your container setup.
- Ensure MariaDB is fully started before executing this command.

---

## Debezium Connector

This section defines the **source connector** configuration for Debezium. It establishes the connection to the **MariaDB** database and tells Debezium which schema and tables to monitor for changes.

In this example, we are capturing changes from the `classicmodels.customers` table.

### Source Connector Configuration (`source-classicmodels`)

Save the following JSON content as `source-classicmodels.json`:

```json
{
  "name": "source-classicmodels",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "autoReconnect": "true",

    "database.hostname": "mariadb",
    "database.port": "3306",
    "database.user": "root",
    "database.password": "mysql-pass",
    "database.server.id": "3363",

    "database.include.list": "classicmodels",
    "table.include.list": "classicmodels.customers",
    "topic.prefix": "mysql-source-classicmodels",

    "transforms.escapeReservedKeywords.replacement": "`$1`",

    "schema.history.internal.kafka.bootstrap.servers": "broker:29092",
    "schema.history.internal.kafka.topic": "schema-changes.mysql-source-classicmodels",

    "signal.data.collection": "classicmodels.source-classicmodels-signal"
  }
}
```

#### Key Fields Explained

| Field | Description |
|-------|-------------|
| `connector.class` | Type of Debezium connector; in this case, MySQL |
| `database.include.list` | Only monitor the `classicmodels` schema |
| `table.include.list` | Only capture changes for the `customers` table |
| `topic.prefix` | Kafka topics will be prefixed with this value |
| `schema.history.*` | Where Debezium stores schema changes internally |
| `signal.data.collection` | Optional: used for incremental snapshot signaling |

---

#### Applying the Connector

##### 1. Save the Configuration

Save the configuration above as `source-classicmodels.json`.

##### 2. Register the Connector with Kafka Connect

```bash
curl -i -X POST http://localhost:8083/connectors \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d @source-classicmodels.json
```

##### 3. Verify the Connector Is Running

List all connectors:

```bash
curl -s http://localhost:8083/connectors | jq
```

Check connector status:

```bash
curl -s http://localhost:8083/connectors/source-classicmodels/status | jq
```

Expected output:

```json
{
  "name": "source-classicmodels",
  "connector": {
    "state": "RUNNING",
    "worker_id": "172.20.0.7:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "172.20.0.7:8083"
    }
  ],
  "type": "source"
}
```

##### 4. View CDC Events in Kafka

After inserting or updating data in `classicmodels.customers`, you can verify CDC events by consuming the topic:

```bash
docker exec -it broker kafka-console-consumer \
  --bootstrap-server broker:9092 \
  --topic mysql-source-classicmodels.classicmodels.customers \
  --from-beginning
```

You should see JSON messages representing change events (insert, update, delete).

---

## Debezium Connector Offsets

Debezium uses **offsets** to track which changes it has already processed in the source database. These offsets correspond to the **binlog file name and position** in MySQL/MariaDB.

You can **inspect**, **modify**, or **delete** offsets using the following Kafka Connect REST API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET    | /connectors/<connector_name>/offsets | View current offset |
| PATCH  | /connectors/<connector_name>/offsets | Modify offset to a specific binlog position |
| DELETE | /connectors/<connector_name>/offsets | Remove offset and restart from beginning or snapshot |

---

### Get Current Binlog File (MariaDB/MySQL)

To manually retrieve the current binlog file and position from MariaDB, connect to the database and run:

```sql
SHOW MASTER STATUS;
```

Example output:

```
+---------------------+----------+--------------+------------------+-------------------+
| File                | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+---------------------+----------+--------------+------------------+-------------------+
| mariadb-bin.000038  | 46451523 |              |                  |                   |
+---------------------+----------+--------------+------------------+-------------------+
```

These values (`File` and `Position`) can be used to **manually reset Debezium’s offset** using the PATCH method.

---

### Get Binlog Position from Binlog Events

If you need to find a specific binlog position based on an event (e.g. a known `INSERT` or `UPDATE`), you can inspect the binary logs directly using the `mysqlbinlog` utility.

#### 1. Locate the Binlog File

First, determine the available binlog files:

```sql
SHOW BINARY LOGS;
```

Example output:

```
+---------------------+-----------+
| Log_name            | File_size |
+---------------------+-----------+
| mariadb-bin.000037  | 18768452  |
| mariadb-bin.000038  | 29152291  |
+---------------------+-----------+
```

#### 2. Use `SHOW BINLOG EVENTS`

This gives you a list of **all raw binlog events** in a given file:

```sql
SHOW BINLOG EVENTS IN 'mariadb-bin.000038'
[FROM 46450000] LIMIT 10;
```

Example:

```
+---------------------+-----------+-------------+-----------+-------------+---------------------------------------+
| Log_name            | Pos       | Event_type  | Server_id | End_log_pos | Info                                  |
+---------------------+-----------+-------------+-----------+-------------+---------------------------------------+
| mariadb-bin.000038  | 46451523  | Table_map   | 1         | 46451600    | table_id: 108 (classicmodels.customers) |
| mariadb-bin.000038  | 46451600  | Write_rows  | 1         | 46451800    | table_id: 108 flags: STMT_END_F       |
+---------------------+-----------+-------------+-----------+-------------+---------------------------------------+
```

You can identify the event you're interested in (e.g. `Write_rows`) and extract its `Pos` value for the offset.

#### 3. Use It in a PATCH Offset Request

```json
"file": "mariadb-bin.000038",
"pos": 46451523
```

Paste this into your PATCH request as shown in the previous example.

**Tips**:
- Ensure the file path is correct and accessible (e.g. `/var/lib/mysql/mariadb-bin.000038` inside Docker).
- You may need to copy the binlog file from the container to your host using `docker cp`.


---

### Modify Offset Manually (PATCH)

Example: Set the offset for the connector `source-ehrm` to a specific binlog file and position.

```bash
curl --location --globoff --request PATCH 'http://localhost:8083/connectors/source-ehrm/offsets/' \
--header 'Content-Type: application/json' \
--data '{
  "offsets":[
     {
        "partition":{
           "server":"mysql-classicmodels"
        },
        "offset":{
          "ts_sec": 1740005145,
          "file": "mariadb-bin.000038",
          "pos": 46451523
        }
     }
  ]
}'
```

**Note**: Replace `server`, `file`, and `pos` with your actual values. The `ts_sec` is optional and may be ignored depending on connector version.

---

### Clear Offset (DELETE)

To remove the stored offset and force Debezium to restart (from snapshot or earliest binlog position depending on connector settings):

```bash
curl -X DELETE http://localhost:8083/connectors/source-ehrm/offsets
```

---

## CDC Verification Example

This section shows how to test that CDC is working correctly, from insert → Kafka topic → target DB.

---

### 1. Insert Data Into Source MySQL

Manually insert a record into the source database:

```sql
USE classicmodels;
UPDATE customerName SET WHERE customerNumber=103;
```

---

### 2. Consume Data From Kafka

Use Kafka console tools to verify the message was produced:

```bash
docker exec -it broker kafka-console-consumer \
  --bootstrap-server broker:9092 \
  --topic mysql-source-classicmodels.classicmodels.customers \
  --from-beginning
```

Expected output:

```json
{
      "schema": { ... },
      "payload": {
            "before": {
                  "customerNumber": 103,
                  "customerName": "Atelier graphique",
                  "contactLastName": "Schmitt",
                  "contactFirstName": "Carine ",
                  "phone": "40.32.2555",
                  "addressLine1": "54, rue Royale",
                  "addressLine2": null,
                  "city": "Nantes",
                  "state": null,
                  "postalCode": "44000",
                  "country": "France",
                  "salesRepEmployeeNumber": 1370,
                  "creditLimit": "IAsg"
            },
            "after": {
                  "customerNumber": 103,
                  "customerName": "Atelier graphie",
                  "contactLastName": "Schmitt",
                  "contactFirstName": "Carine ",
                  "phone": "40.32.2555",
                  "addressLine1": "54, rue Royale",
                  "addressLine2": null,
                  "city": "Nantes",
                  "state": null,
                  "postalCode": "44000",
                  "country": "France",
                  "salesRepEmployeeNumber": 1370,
                  "creditLimit": "IAsg"
            },
            "source": { ... },
            "transaction": null,
            "op": "u",
            "ts_ms": 1752933503893,
            "ts_us": 1752933503893363,
            "ts_ns": 1752933503893363201
      }
}
```

---

### 3. Confirm Inserted Into Target DB

Query the target database to confirm replication occurred:

```sql
SELECT * FROM phone_book;
-- Should return: 1 | Alice | 12345
```

---

---

## Manage Debezium Connectors (Pause, Resume, Stop, Delete)

Kafka Connect provides multiple ways to control connector behavior. Each affects how the connector interacts with the database and Kafka.

---

### Pause a Connector

Temporarily halts the connector but keeps it registered. It stops processing binlog events and closes the database connection.

```bash
curl -X PUT http://localhost:8083/connectors/<connector_name>/pause
```

#### Behavior:
- Disconnects from the database
- Stops CDC event publishing
- Can be resumed later (retains config and offset)

---

### Resume a Connector

Resumes a paused connector, reconnecting to the database and continuing from the last offset.

```bash
curl -X PUT http://localhost:8083/connectors/<connector_name>/resume
```

#### Behavior:
- Re-establishes DB connection
- Resumes CDC and Kafka publishing

---

### Stop a Connector

**Stops the connector completely** — similar to `pause`, but cannot be resumed via `resume`. You must explicitly **restart** it using:

```bash
curl -X PUT http://localhost:8083/connectors/<connector_name>/restart
```

```bash
curl -X PUT http://localhost:8083/connectors/<connector_name>/stop
```

Example:

```bash
curl -X PUT http://localhost:8083/connectors/source-classicmodels/stop
```

#### Behavior:
- Connector transitions to `STOPPED` state
- Disconnects from database
- Offset is preserved
- Must be restarted (not resumed)

---

### Restart a Stopped Connector

```bash
curl -X POST http://localhost:8083/connectors/<connector_name>/restart
```

---

### Delete a Connector

Removes the connector entirely.

```bash
curl -X DELETE http://localhost:8083/connectors/<connector_name>
```

#### Behavior:
- Unregisters the connector
- Disconnects DB
- Stops publishing
- Offsets are retained unless explicitly removed:

```bash
curl -X DELETE http://localhost:8083/connectors/<connector_name>/offsets
```

---

### Check Connector Status

```bash
curl -s http://localhost:8083/connectors/<connector_name>/status | jq
```

Returns one of: `RUNNING`, `PAUSED`, `FAILED`, `STOPPED`, etc.

---

## Dynamically Add a New Table Using Debezium Signals

Debezium allows you to modify what tables are captured **at runtime**, without restarting the connector. This is done via the **incremental snapshot signal** sent to a control table inside the database.

### Prerequisites

Ensure your connector config includes:

```json
"signal.data.collection": "classicmodels.source-classicmodels-signal"
```

> This tells Debezium to listen to `classicmodels.source-classicmodels-signal` table for special instructions.

Also, make sure the table exists in the database:

```sql
CREATE TABLE classicmodels.source-classicmodels-signal (
  id VARCHAR(64) PRIMARY KEY,
  type VARCHAR(32) NOT NULL,
  data JSON NOT NULL
);
```

---

### Step-by-Step: Add a Table to the Connector

Let’s say we want to start capturing the `classicmodels.orders` table in addition to `customers`.

#### 1. Insert a Signal Record

Run this SQL in MariaDB:

```sql
INSERT INTO classicmodels.`source-classicmodels-signal` (id, `type`, `data`) VALUES(
  'orders',
  'execute-snapshot',
  '{
    "data-collections": ["classicmodels.orders"],
    "type":"blocking"
  }'
);
```

#### 2. Verify Signal Was Consumed

You should see this reflected in Kafka logs or Debezium UI — the connector will start snapshotting the new table.

Then, use this to verify Kafka topic:

```bash
docker exec -it broker kafka-console-consumer \
  --bootstrap-server broker:9092 \
  --topic mysql-source-classicmodels.classicmodels.orders \
  --from-beginning
```

You should start seeing events from `orders`.

---

### Notes

- This works even if the table was not originally included in `table.include.list`, as long as the connector config allows it (i.e. if `table.include.list` is not restrictive).
- For more robust control, you can combine this with configuration changes via the Kafka Connect REST API.

---

## Cleanup

Stop and remove all containers and volumes.

```bash
docker-compose down -v
```

Use this when you're done testing or want to reset everything.

---

## Troubleshooting Tips

Useful commands to debug common issues:

- View container logs:

  ```bash
  docker logs <container_name>
  ```

- Check container health:

  ```bash
  docker inspect --format='{{json .State.Health}}' <container_name>
  ```

---

## Notes

- Debezium uses the topic name format: `<prefix>.<database>.<table>`
- Kafka will auto-create topics unless otherwise configured
- Default ports can be changed in `.env` or `docker-compose.yml`
- You can pause or resume a connector using:

  ```bash
  curl -X PUT http://localhost:8083/connectors/<name>/pause
  curl -X PUT http://localhost:8083/connectors/<name>/resume
  ```

---

Happy streaming!