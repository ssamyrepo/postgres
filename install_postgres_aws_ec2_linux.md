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
You're logged in as the `postgres` user, and **sudo is requiring a password**. However, by default, the `postgres` user should not have a password for sudo. Instead, you should **run administrative commands as `ec2-user` or root**.

---

## **🔹 1. Switch Back to `ec2-user`**
Since `postgres` does not have sudo privileges, **exit the `postgres` user** and return to `ec2-user`:
```bash
exit
```
or
```bash
su - ec2-user
```

Then, **run the commands as `ec2-user`**.

---

## **🔹 2. Ensure the Correct PostgreSQL Data Directory**
Check what directory PostgreSQL is using:
```bash
ls -ld /var/lib/pgsql/*
```
If `/var/lib/pgsql/data` is **empty or missing**, PostgreSQL is not correctly initialized.

To fix it, **reinitialize PostgreSQL**:
```bash
sudo rm -rf /var/lib/pgsql/data  # Delete old empty directory
sudo ln -s /var/lib/pgsql/15/data /var/lib/pgsql/data
sudo chown -R postgres:postgres /var/lib/pgsql/15
sudo chmod 700 /var/lib/pgsql/15/data
```

---

## **🔹 3. Restart PostgreSQL Properly**
Once back in `ec2-user`, **start and enable PostgreSQL**:
```bash
sudo systemctl enable --now postgresql
```
Check its status:
```bash
sudo systemctl status postgresql
```
If PostgreSQL fails, check logs:
```bash
sudo journalctl -xeu postgresql.service
```

---

## **🔹 4. Verify PostgreSQL is Running**
Try:
```bash
psql -U postgres -d postgres
```
If successful, you should see:
```
postgres=#
```

---


