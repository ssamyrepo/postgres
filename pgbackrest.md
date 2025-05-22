Here is a **step-by-step guide** to configure **pgBackRest** for **full and incremental backups** on a PostgreSQL database.

---

## 🔧 Step-by-Step Setup: pgBackRest on PostgreSQL

> 🐧 This assumes you are using a Linux system with PostgreSQL already installed.

---

### ✅ **1. Install `pgBackRest`**

#### On RHEL/CentOS:

```bash
sudo yum install pgbackrest -y
```

#### On Debian/Ubuntu:

```bash
sudo apt install pgbackrest -y
```

---

### ✅ **2. Create pgBackRest Configuration**

#### Global config file:

```bash
sudo nano /etc/pgbackrest/pgbackrest.conf
```

#### Sample:

```ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2

[main]
pg1-path=/var/lib/pgsql/14/data   # adjust path and version accordingly
```

---

### ✅ **3. Set Permissions and Create Directories**

```bash
sudo mkdir -p /var/lib/pgbackrest
sudo chown -R postgres:postgres /var/lib/pgbackrest
sudo chmod 750 /var/lib/pgbackrest
```

---

### ✅ **4. Enable Archive Mode in PostgreSQL**

Edit `postgresql.conf`:

```conf
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
wal_level = replica
max_wal_senders = 3
```

Ensure the `pg_hba.conf` allows localhost access for `replication`.

Reload PostgreSQL:

```bash
sudo systemctl reload postgresql
```

---

### ✅ **5. Create the Stanza**

```bash
sudo -u postgres pgbackrest --stanza=main stanza-create
```

You should see:

```bash
stanza-create command end: completed successfully
```

---

## 📦 Backup Operations

### 🔁 **Full Backup**

```bash
sudo -u postgres pgbackrest --stanza=main --type=full backup
```

### 🔁 **Incremental Backup**

```bash
sudo -u postgres pgbackrest --stanza=main --type=incr backup
```

> 📝 Tip: pgBackRest tracks WAL files to create differential and incremental backups automatically.

---

## 🔁 Restore a Backup

```bash
sudo -u postgres pgbackrest --stanza=main restore
```

You might want to stop PostgreSQL before restore:

```bash
sudo systemctl stop postgresql
```

---

## 📊 Verify Backups

```bash
sudo -u postgres pgbackrest --stanza=main info
```

This will show available full, diff, and incr backups.

---

## 🕑 Optional: Schedule via `cron`

### Full backup every day at 2 AM

```bash
0 2 * * * postgres pgbackrest --stanza=main --type=full backup
```

### Incremental every hour

```bash
0 * * * * postgres pgbackrest --stanza=main --type=incr backup
```

---

Would you like a **Kubernetes**-based backup strategy or an **S3 target** setup as well?
