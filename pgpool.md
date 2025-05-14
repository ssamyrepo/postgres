Here’s a **step-by-step guide** to set up **Pgpool-II with PostgreSQL cascading streaming replication**, including:

* ✅ One primary PostgreSQL server
* ✅ One standby that replicates directly from the primary
* ✅ One **cascading standby** that replicates from the standby
* ✅ Pgpool-II in front, handling load balancing, health checks, and failover

---

## 🗺️ **Architecture Overview**

```
               +----------------+
               |   Pgpool-II    |
               +----------------+
               |   |     |
         [Primary] [Standby1] [Standby2 (cascading)]
         (Node 0)  (Node 1)   (Node 2)
             ↑        ↑
        WAL sender   WAL receiver (from standby1)
```

---

## 🧰 **Prerequisites**

* PostgreSQL installed on all 3 nodes
* Pgpool-II installed on the Pgpool host
* SSH access and passwordless SSH between all DB hosts (for failover scripts)

---

## ✅ **Step 1: Set Up PostgreSQL Primary (Node 0)**

1. Update `postgresql.conf`:

```conf
wal_level = replica
max_wal_senders = 10
hot_standby = on
archive_mode = on
archive_command = 'cp %p /var/lib/pgsql/wal_archive/%f'
```

2. Set `pg_hba.conf` to allow replication:

```conf
host replication replicator 0.0.0.0/0 md5
```

3. Create replication user:

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN ENCRYPTED PASSWORD 'replpass';
```

---

## ✅ **Step 2: Set Up Standby1 (Node 1)**

On standby1:

1. Stop PostgreSQL if running:

```bash
pg_ctl stop -D $PGDATA
```

2. Run base backup:

```bash
pg_basebackup -h primary_host -U replicator -D $PGDATA -Fp -Xs -P -R
```

3. Start PostgreSQL. `standby.signal` is created automatically in PG 12+.

---

## ✅ **Step 3: Set Up Cascading Standby2 (Node 2)**

1. Similar to Standby1, but change the base backup source:

```bash
pg_basebackup -h standby1_host -U replicator -D $PGDATA -Fp -Xs -P -R
```

2. Edit `primary_conninfo` in `postgresql.auto.conf` if needed:

```conf
primary_conninfo = 'host=standby1_host user=replicator password=replpass'
```

3. Start PostgreSQL.

---

## ✅ **Step 4: Install and Configure Pgpool-II**

1. Install Pgpool-II on a separate node (or one of the DB nodes):

```bash
sudo apt install pgpool2  # or use yum
```

2. Update `pgpool.conf` with backend info:

```conf
backend_hostname0 = 'primary_host'
backend_port0 = 5432
backend_weight0 = 1
backend_flag0 = 'ALLOW_TO_FAILOVER'

backend_hostname1 = 'standby1_host'
backend_port1 = 5432
backend_weight1 = 1
backend_flag1 = 'ALLOW_TO_FAILOVER'

backend_hostname2 = 'standby2_host'
backend_port2 = 5432
backend_weight2 = 1
backend_flag2 = 'DISALLOW_TO_FAILOVER'  # Cascading standby should not be promoted

load_balance_mode = on
master_slave_mode = on
sr_check_user = 'postgres'
sr_check_password = 'postgres'
```

3. Enable failover and watchdog options as needed.

4. Start Pgpool-II:

```bash
pgpool -n
```

---

## ✅ **Step 5: Verify the Setup**

1. Connect to Pgpool and test read queries:

```bash
psql -h pgpool_host -U postgres -d mydb
```

2. Check if reads are balanced across standby1 and standby2:

```sql
SELECT inet_server_addr(), now();
```

3. Simulate failover by stopping the primary:

```bash
pg_ctl stop -D $PGDATA
```

Pgpool should promote standby1 if `failover_command` is configured.

---

## 🧪 Optional: Configure `failover_command`

Create a shell script (`failover.sh`) and specify in `pgpool.conf`:

```conf
failover_command = '/etc/pgpool2/failover.sh %d %h %p %D %m %H %P %r %R'
```

The script should:

* Promote standby using `pg_ctl promote`
* Update recovery configs
* Reconfigure other standbys if needed

---

## 🧠 Tips

* Always disable failover on cascading standbys (`backend_flag = 'DISALLOW_TO_FAILOVER'`)
* Monitor using `pcp_node_info`, `SHOW pool_nodes;`
* Tune `sr_check_period`, `health_check_period` for proactive monitoring

---

Would you like a **Docker Compose setup** or a **single-node simulation with 3 data directories** for local testing?
