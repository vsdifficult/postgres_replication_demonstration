# PostgreSQL Master-Replica Demo (Docker)

Demo project for setting up **PostgreSQL replication (master ↔ replica)** using Docker Compose.  
Allows you to create master and replica instances, automatically synchronize data, and verify replication.

---

## 🔹 Features

- Master + Replica in separate Docker containers  
- Full physical replication via `pg_basebackup`  
- Replica runs in **read-only** mode (`hot_standby`)  
- Automatic creation of `testdb` database  
- Configured `pg_hba.conf` for replication connections  

---

## 📂 Project Structure

```
postgres_replication_demonstration/
├─ docker-compose.yml       # Docker Compose configuration
├─ pg_hba.conf              # Access configuration for replication
└─ README.md                # This file
```

---

## ⚙️ Setup and Launch

1. **Clean up old containers and volumes:**

```bash
docker compose down -v
docker volume prune -f
```

2. **Start master and replica:**

```bash
docker compose up -d
```

3. **Verify both containers are running:**

```bash
docker ps
```

Expected output:

```
pg-master   Up
pg-replica  Up
```

---

## ✅ Testing

### 1. Check databases on master

```bash
docker exec -it pg-master psql -U postgres -c "\l"
```

### 2. Create table and insert data on master

```bash
docker exec -it pg-master psql -U postgres -d testdb -c "CREATE TABLE test (id SERIAL, data TEXT); INSERT INTO test (data) VALUES ('hello from master');"
```

### 3. Verify replica

```bash
docker exec -it pg-replica psql -U postgres -d testdb -c "SELECT * FROM test;"
```

- Data added on master should be visible
- Replica operates in **read-only** mode

### 4. Check replication status

On master:

```bash
docker exec -it pg-master psql -U postgres -c "SELECT * FROM pg_stat_replication;"
```

On replica:

```bash
docker exec -it pg-replica psql -U postgres -c "SELECT pg_is_in_recovery();"
```

---

## 🔹 How Replication Works

```
┌────────────────┐          WAL           ┌────────────────┐
│   pg-master    │ ────────────────────►  │   pg-replica   │
│                │                        │                │
│ CREATE/INSERT  │                        │  APPLY WAL     │
│ UPDATE/DELETE  │                        │  (read-only)   │
└────────────────┘                        └────────────────┘
```

1. Master generates **WAL (Write-Ahead Logs)** for all changes.
2. Replica connects to master via `pg_basebackup` and receives WAL.
3. Replica applies WAL and maintains current database state.
4. Replica is available for **reads only**, writes go through master only.

---

## 🔧 Configuration

### pg_hba.conf

Allows replica to connect to master:

```conf
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
host    all             all             ::1/128                 trust
host    replication     all             0.0.0.0/0               trust
```

### docker-compose.yml (master + replica)

```yaml
version: "3.9"

services:
  pg-master:
    image: postgres:15
    container_name: pg-master
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: testdb
    ports:
      - "5432:5432"
    volumes:
      - pg-master-data:/var/lib/postgresql/data
    command: >
      postgres
      -c wal_level=replica
      -c max_wal_senders=10
      -c hot_standby=on

  pg-replica:
    image: postgres:15
    container_name: pg-replica
    depends_on:
      - pg-master
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "5433:5432"
    volumes:
      - pg-replica-data:/var/lib/postgresql/data
    entrypoint: >
      bash -c "
      until pg_isready -h pg-master -p 5432; do sleep 2; done &&
      rm -rf /var/lib/postgresql/data/* &&
      pg_basebackup -h pg-master -U postgres -D /var/lib/postgresql/data -Fp -Xs -P -R &&
      exec docker-entrypoint.sh postgres
      "

volumes:
  pg-master-data:
  pg-replica-data:
```

---

## ⚠️ Important

- **`trust` authentication for all connections** — for local demo only
- In production use **md5 / SCRAM + SSL**
- `pg_basebackup` overwrites replica data — volume must be empty before first sync

---

## 📚 Useful Commands

- Stop containers:

```bash
docker compose down
```

- Connect to master:

```bash
docker exec -it pg-master psql -U postgres
```

- Connect to replica:

```bash
docker exec -it pg-replica psql -U postgres
```

---

## ⚡ Result

After launch:

- Master accepts changes and creates WAL
- Replica receives WAL and synchronizes
- Replica is available for reading and always up-to-date

---

## 📝 License

MIT
