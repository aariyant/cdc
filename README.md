# 🧪 Kafka + Schema Registry + Debezium Tutorial

This tutorial will guide you through running and verifying a Kafka-based CDC (Change Data Capture) pipeline using **Kafka**, **Schema Registry**, and **Debezium**, all orchestrated via Docker Compose.

---

## 📋 Prerequisites

- Docker
- Docker Compose
- Basic understanding of Kafka and Debezium

---

## 🗂️ Project Structure

```
.
├── docker-compose.yml
├── kafka-data/
├── config/
│   └── zookeeper.properties
├── kafka-ui-config.yaml
```

---

## 🚀 Step-by-Step Setup

### 1. 🧠 Start Zookeeper

Zookeeper is a distributed coordination service that Kafka depends on for broker coordination, metadata management, and leader election. In this setup, the Zookeeper service is defined in the `docker-compose.yaml` file with the following configuration:

- **Image**: `confluentinc/cp-zookeeper:7.3.1` — The official Zookeeper image provided by Confluent.
- **Hostname**: `zookeeper` — The hostname used within the Docker network.
- **Ports**: Exposes port `2181` for client connections.
- **Environment Variables**:
  - `ZOOKEEPER_CLIENT_PORT`: The port Zookeeper listens on for client connections (default: `2181`).
  - `ZOOKEEPER_TICK_TIME`: The basic time unit in milliseconds used by Zookeeper for heartbeats and timeouts (default: `2000` ms).
  - `KAFKA_OPTS` and `KAFKA_HEAP_OPTS`: JVM options for tuning Zookeeper's performance.
- **Volumes**: Mounts the `zookeeper.properties` configuration file from the `./config/` directory to `/etc/kafka/zookeeper.properties` inside the container.
- **Healthcheck**: Ensures the service is healthy by checking if Zookeeper responds on port `2181`.

To start Zookeeper, run:

```bash
docker-compose up -d zookeeper
```

✅ **Verify**:

```bash
docker logs -f zookeeper
```

Look for:

```
binding to port 0.0.0.0/0.0.0.0:2181
```

---

### 2. 📡 Start Kafka Broker

Kafka is the core message broker in this CDC pipeline. It relies on Zookeeper for broker coordination and metadata management. The Kafka service is defined in the `docker-compose.yaml` file with the following configuration:

- **Image**: `confluentinc/cp-kafka:7.3.1` — The official Kafka image provided by Confluent.
- **Hostname**: `broker` — The hostname used within the Docker network.
- **Ports**:
  - `9092`: Exposed for host access.
  - `29092`: Used for internal Docker network communication.
- **Environment Variables**:
  - `KAFKA_ZOOKEEPER_CONNECT`: Specifies the Zookeeper connection string (`zookeeper:2181`).
  - `KAFKA_ADVERTISED_LISTENERS`: Configures how Kafka advertises itself to clients.
  - `KAFKA_AUTO_CREATE_TOPICS_ENABLE`: Enables automatic topic creation (default: `true`).
- **Volumes**: Mounts the `kafka-data` directory to persist Kafka data.
- **Healthcheck**: Ensures the service is healthy by checking if Kafka responds on port `9092`.

To start Kafka, run:

```bash
docker-compose up -d broker
```

✅ **Verify**:

```bash
docker logs -f broker
```

Look for:

```
started (kafka.server.KafkaServer)
```

---

### 3. 🖥️ Kafka UI (Optional)

Kafka UI is a web-based interface for browsing Kafka topics and inspecting messages. The Kafka UI service is defined in the `docker-compose.yaml` file with the following configuration:

- **Image**: `provectuslabs/kafka-ui:latest` — The official Kafka UI image.
- **Hostname**: `kafka-ui` — The hostname used within the Docker network.
- **Ports**: Exposes port `8180` for accessing the UI.
- **Environment Variables**:
  - `DYNAMIC_CONFIG_ENABLED`: Enables dynamic configuration (default: `true`).
- **Volumes**: Mounts the `kafka-ui-config.yaml` file for custom configurations.

To start Kafka UI, run:

```bash
docker-compose up -d kafka-ui
```

🌐 Open: [http://localhost:8180](http://localhost:8180)

---

### 4. 📚 Start Schema Registry

The Schema Registry manages schemas for Kafka messages, ensuring compatibility between producers and consumers. The Schema Registry service is defined in the `docker-compose.yaml` file with the following configuration:

- **Image**: `confluentinc/cp-schema-registry:7.3.1` — The official Schema Registry image provided by Confluent.
- **Hostname**: `schema-registry` — The hostname used within the Docker network.
- **Ports**: Exposes port `8081` for accessing the Schema Registry API.
- **Environment Variables**:
  - `SCHEMA_REGISTRY_HOST_NAME`: Sets the hostname for the Schema Registry.
  - `SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS`: Specifies the Kafka bootstrap servers.
- **Healthcheck**: Ensures the service is healthy by checking if the Schema Registry API responds.

To start the Schema Registry, run:

```bash
docker-compose up -d schema-registry
```

✅ **Verify**:

```bash
curl http://localhost:8081/subjects
```

Should return an empty array `[]` if no schema exists yet.

---

### 5. 🌐 Kafka REST Proxy (Optional)

The Kafka REST Proxy provides HTTP-based access to Kafka, allowing you to produce and consume messages via REST APIs. The REST Proxy service is defined in the `docker-compose.yaml` file with the following configuration:

- **Image**: `confluentinc/cp-kafka-rest:7.3.1` — The official Kafka REST Proxy image.
- **Hostname**: `rest-proxy` — The hostname used within the Docker network.
- **Ports**: Exposes port `8082` for accessing the REST API.
- **Environment Variables**:
  - `KAFKA_REST_BOOTSTRAP_SERVERS`: Specifies the Kafka bootstrap servers.
  - `KAFKA_REST_LISTENERS`: Configures the REST Proxy listeners.

To start the REST Proxy, run:

```bash
docker-compose up -d rest-proxy
```

✅ **Test**:

```bash
curl http://localhost:8082/topics
```

---

### 6. 🔄 Start Debezium Connect

Debezium Connect is a Kafka Connect instance with the Debezium plugin for capturing database changes. The Debezium service is defined in the `docker-compose.yaml` file with the following configuration:

- **Image**: `quay.io/debezium/connect:3.0` — The official Debezium Connect image.
- **Hostname**: `debezium` — The hostname used within the Docker network.
- **Ports**: Exposes port `8083` for accessing the Kafka Connect REST API.
- **Environment Variables**:
  - `BOOTSTRAP_SERVERS`: Specifies the Kafka bootstrap servers.
  - `KEY_CONVERTER` and `VALUE_CONVERTER`: Configures the message format (e.g., JSON).
  - `OFFSET_STORAGE_TOPIC`: Specifies the topic for storing offsets.
- **Healthcheck**: Ensures the service is healthy by checking if the Kafka Connect API responds.

To start Debezium Connect, run:

```bash
docker-compose up -d debezium
```

✅ **Test**:

```bash
curl http://localhost:8083/connectors
```

Returns `[]` if no connectors are registered yet.

---

### 7. 🧭 Debezium UI (Optional)

Debezium UI provides a web-based interface for managing and monitoring Debezium connectors. The Debezium UI service is defined in the `docker-compose.yaml` file with the following configuration:

- **Image**: `debezium/debezium-ui:latest` — The official Debezium UI image.
- **Hostname**: `debezium-ui` — The hostname used within the Docker network.
- **Ports**: Exposes port `8080` for accessing the UI.
- **Environment Variables**:
  - `KAFKA_CONNECT_URIS`: Specifies the Kafka Connect REST API URL.

To start Debezium UI, run:

```bash
docker-compose up -d debezium-ui
```

🌐 Open: [http://localhost:8080](http://localhost:8080)

---

## ➕ Register a Debezium Connector (MySQL Example)

Replace the database connection details with your setup.

```bash
curl -X POST http://localhost:8083/connectors \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "my-connector",
    "config": {
      "connector.class": "io.debezium.connector.mysql.MySqlConnector",
      "database.hostname": "mysql",
      "database.port": "3306",
      "database.user": "debezium",
      "database.password": "dbz",
      "database.server.id": "184054",
      "database.server.name": "dbserver1",
      "database.include.list": "mydb",
      "database.history.kafka.bootstrap.servers": "broker:29092",
      "database.history.kafka.topic": "schema-changes.mydb"
    }
  }'
```

---

## ✅ Check Container Status

```bash
docker-compose ps
```

All services should show `Up`.

---

## 🧼 Cleanup

To stop and remove everything:

```bash
docker-compose down -v
```

---

## 🔍 Troubleshooting Tips

- Use container logs to debug issues:

  ```bash
  docker logs <container_name>
  ```

- Use Kafka UI and Debezium UI to inspect topics and connector states.

---

## 📌 Notes

- You can add a **MySQL/PostgreSQL** service to your compose for a full CDC demo.
- Topics will be auto-created by Kafka by default.
- Adjust ports via `.env` or directly in `docker-compose.yml`.

---

Happy streaming! 🚀

