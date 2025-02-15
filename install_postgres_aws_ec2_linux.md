### **🚀 Installing PostgreSQL on Amazon Linux 2023**
Since you are using **Amazon Linux 2023 (AL2023)**, the installation method is **different** from Amazon Linux 2.

---

### **✅ 1. Install PostgreSQL 15 (Latest Available on AL2023)**
Amazon Linux 2023 uses `dnf` instead of `yum` and does **not** have `amazon-linux-extras`. Follow these steps:

```bash
sudo dnf install -y postgresql15 postgresql15-server
```
This will install **PostgreSQL 15**, which is the default version in AL2023.

---

### **✅ 2. Initialize the Database**
Once PostgreSQL is installed, initialize the database:

```bash
sudo /usr/bin/postgresql-15-setup initdb
```

---

### **✅ 3. Start & Enable PostgreSQL Service**
To start PostgreSQL and ensure it runs on system boot:

```bash
sudo systemctl enable --now postgresql-15
```

Check if PostgreSQL is running:
```bash
sudo systemctl status postgresql-15
```

If it's **active (running)**, your database is working! 🎉

---

### **✅ 4. Verify PostgreSQL Installation**
Run:
```bash
psql --version
```
Expected Output:
```
psql (PostgreSQL) 15.x
```

---

### **✅ 5. Create a PostgreSQL User & Database**
Switch to the **PostgreSQL system user**:
```bash
sudo su - postgres
```

Enter the **PostgreSQL shell**:
```bash
psql
```

Inside `psql`, create a **new user and database**:
```sql
CREATE USER myuser WITH PASSWORD 'mypassword';
ALTER USER myuser WITH SUPERUSER;
CREATE DATABASE mydb OWNER myuser;
\q
```

Exit the `postgres` user:
```bash
exit
```

---

### **✅ 6. Enable Remote Connections (If Needed)**
By default, PostgreSQL **only allows local connections**. To enable remote access:

#### **🔹 Modify `postgresql.conf`**
```bash
sudo nano /var/lib/pgsql/15/data/postgresql.conf
```
Find this line:
```
#listen_addresses = 'localhost'
```
Change it to:
```
listen_addresses = '*'
```
Save and exit (**CTRL + X → Y → ENTER**).

#### **🔹 Modify `pg_hba.conf`**
```bash
sudo nano /var/lib/pgsql/15/data/pg_hba.conf
```
At the bottom, add this line to allow external access:
```
host all all 0.0.0.0/0 md5
```
Save and exit.

#### **🔹 Restart PostgreSQL**
```bash
sudo systemctl restart postgresql-15
```

---

### **✅ 7. Test PostgreSQL Connection**
Run:
```bash
psql -h localhost -U myuser -d mydb -W
```
Enter the password (`mypassword`) when prompted. If you see:
```
psql (15.x)
Type "help" for help.
mydb=>
```
🎉 **PostgreSQL is installed and working!**

---

### **🚀 Final Checks**
1️⃣ **Ensure PostgreSQL is Running**
```bash
sudo systemctl status postgresql-15
```
2️⃣ **Check Firewall Rules (If External Access Fails)**
```bash
sudo firewall-cmd --add-service=postgresql --permanent
sudo firewall-cmd --reload
```
3️⃣ **Security Group in AWS (If on EC2)**
- Go to AWS Console → **EC2 Dashboard** → **Security Groups**.
- Allow **port 5432** for PostgreSQL.

---

### **🎯 PostgreSQL 15 is Now Installed on Amazon Linux 2023!**
Try the steps and let me know if you need help! 🚀🔥
