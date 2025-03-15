step-by-step explanation suitable for an interview on setting up **PostgreSQL streaming replication** between two servers (either on-premise or AWS EC2):

---

## **Overview of PostgreSQL Streaming Replication**

Streaming replication allows you to create a continuous backup (replica) of your PostgreSQL primary server (master). The replica server continuously receives and applies changes via the Write-Ahead Log (WAL).

- **Primary (Master)**: handles read/write requests.
- **Replica (Standby)**: read-only server, continuously synced with primary.

---

## **Step-by-Step Setup of PostgreSQL Streaming Replication**

### **Step 1: Pre-requisites**

- Two PostgreSQL servers (Primary and Replica)
- PostgreSQL installed (same version recommended)
- Network connectivity between servers
- Firewall opened for PostgreSQL ports (default 5432)

---

### **Step 2: Configuration on Primary Server**

#### **A. Modify PostgreSQL Configuration (`postgresql.conf`)**
```bash
vim /var/lib/pgsql/data/postgresql.conf
```

Add/update:
```ini
listen_addresses = '*'
wal_level = replica
max_wal_senders = 5
wal_keep_size = 64MB  # or wal_keep_segments for older versions
archive_mode = on
archive_command = 'cd .'
```

- **`listen_addresses = '*'`** allows connections from replica servers.
- **`wal_level = replica`** enables replication logging.
- **`max_wal_senders`** specifies how many replicas can connect.
- **`wal_keep_size`** retains WAL files needed by replicas.

#### **B. Configure Replication Authentication (`pg_hba.conf`)**
```bash
vim /var/lib/pgsql/data/pg_hba.conf
```

Add a line allowing the replica to connect:
```ini
host  replication  replicator_user  REPLICA_IP_ADDRESS/32  md5
```

- `replication` is a special database type in PostgreSQL for replication connections.
- Replace `REPLICA_IP_ADDRESS` with your replica server’s IP.

#### **C. Restart Primary PostgreSQL**
```bash
systemctl restart postgresql
```

---

### **Step 3: Create a Replication User on Primary**
```sql
CREATE ROLE replicator_user WITH REPLICATION LOGIN ENCRYPTED PASSWORD 'securepassword';
```

---

### **Step 4: Backup Primary Database and Transfer to Replica**

On the **Replica Server**, ensure PostgreSQL is stopped:
```bash
systemctl stop postgresql
```

Clear the data directory (careful!):
```bash
rm -rf /var/lib/pgsql/data/*
```

Perform a base backup from the **Replica Server** using `pg_basebackup`:
```bash
pg_basebackup -h PRIMARY_IP_ADDRESS -D /var/lib/pgsql/data/ -U replicator_user -W -P --wal-method=stream
```

- `-h`: Primary server’s IP.
- `-D`: Data directory on replica.
- `-U`: Replication user.
- `-W`: Prompt for password.
- `-P`: Shows progress.
- `--wal-method=stream`: Streams WAL logs during backup.

---

### **Step 5: Configuration on Replica Server**

#### **A. Create a standby signal file (Postgres 12+)**
```bash
touch /var/lib/pgsql/data/standby.signal
```

*(In PostgreSQL 11 or older, create a `recovery.conf` file instead.)*

#### **B. Update replica PostgreSQL configuration (`postgresql.conf`)**
```ini
hot_standby = on
primary_conninfo = 'host=PRIMARY_IP_ADDRESS port=5432 user=replicator_user password=securepassword'
```

- `hot_standby = on` allows read-only queries on replica.

Ensure the file permissions are correct:
```bash
chown -R postgres:postgres /var/lib/pgsql/data
chmod 700 /var/lib/pgsql/data
```

#### **C. Start PostgreSQL on Replica**
```bash
systemctl start postgresql
```

---

### **Step 6: Verify Replication**

On the **Replica Server**, verify replication status:
```bash
psql -U postgres
```
```sql
SELECT * FROM pg_stat_wal_receiver;
```

On the **Primary Server**, verify connection status:
```sql
SELECT * FROM pg_stat_replication;
```

These commands show replication status and confirm that streaming is active.

---

## **High-Level Replication Flow Explained:**

- Replica connects to Primary via WAL protocol.
- WAL changes are continuously streamed.
- Replica applies received WAL changes, keeping it in sync.

---

## **Best Practices for Replication:**
- Monitor replication lag regularly.
- Ensure network stability between primary and replica.
- Regularly test failover procedures.
- Secure replication connections (e.g., VPN, SSL/TLS).

---

**Interview Tip:** Clearly explain each step, emphasize critical points like configuration parameters (`wal_level`, `primary_conninfo`), and stress on verifying replication status for confidence.

