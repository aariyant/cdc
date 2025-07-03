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

Zookeeper is the coordination service Kafka depends on.

```bash
docker-compose up -d zookeeper
```

✅ **Verify:**

```bash
docker logs -f zookeeper
```

Look for:
```
binding to port 0.0.0.0/0.0.0.0:2181
```

---

### 2. 📡 Start Kafka Broker

Kafka relies on Zookeeper to start.

```bash
docker-compose up -d broker
```

✅ **Verify:**

```bash
docker logs -f broker
```

Look for:
```
started (kafka.server.KafkaServer)
```

Kafka ports:
- `9092` — host access
- `29092` — internal Docker network access

---

### 3. 🖥️ Kafka UI (Optional)

Kafka UI helps you browse topics and inspect messages.

```bash
docker-compose up -d kafka-ui
```

🌐 Open: [http://localhost:8180](http://localhost:8180)

---

### 4. 📚 Start Schema Registry

Used for managing message schemas (e.g., Avro).

```bash
docker-compose up -d schema-registry
```

✅ **Verify:**

```bash
curl http://localhost:8081/subjects
```

Should return an empty array `[]` if no schema exists yet.

---

### 5. 🌐 Kafka REST Proxy (Optional)

Provides HTTP-based access to Kafka.

```bash
docker-compose up -d rest-proxy
```

✅ **Test:**

```bash
curl http://localhost:8082/topics
```

---

### 6. 🔄 Start Debezium Connect

Kafka Connect + Debezium plugin for CDC.

```bash
docker-compose up -d debezium
```

✅ **Test:**

```bash
curl http://localhost:8083/connectors
```

Returns `[]` if no connectors are registered yet.

---

### 7. 🧭 Debezium UI (Optional)

Manage and monitor your Debezium connectors visually.

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

