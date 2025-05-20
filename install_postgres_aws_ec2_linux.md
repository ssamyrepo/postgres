### ** Installing PostgreSQL on Amazon Linux 2023**
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

It looks like you're making progress with setting up PostgreSQL and your Flask application on your AWS EC2 instance. However, there are a few issues and steps that need to be addressed to get your Flask app running properly. Let's go through them step by step:

---

### **1. Fix the `flask` Module Not Found Error**
The error `ModuleNotFoundError: No module named 'flask'` occurs because the `flask` package is not installed in your virtual environment.

1. Activate your virtual environment (if not already activated):
   ```bash
   source venv/bin/activate
   ```
2. Install the required Python packages:
   ```bash
   pip install flask psycopg2-binary
   ```

---

### **2. Fix the `psql` Connection Error**
The error `role "ec2-user" does not exist` occurs because PostgreSQL does not recognize the `ec2-user` as a valid database user. You need to use the `postgres` user or the `flaskuser` you created to interact with PostgreSQL.

To connect to PostgreSQL as the `flaskuser`:
```bash
psql -U flaskuser -d myflaskdb -h localhost
```
You will be prompted for the password (`securepassword` in your case).

---

### **3. Update Your Flask Application**
Ensure your Flask application (`app.py`) is correctly configured to connect to the PostgreSQL database. Here's an example:

```python
from flask import Flask, request, jsonify
import psycopg2

app = Flask(__name__)

# Database connection
def get_db_connection():
    conn = psycopg2.connect(
        host='localhost',
        database='myflaskdb',
        user='flaskuser',  # Use the user you created
        password='securepassword'  # Use the password you set
    )
    return conn

# Routes
@app.route('/users', methods=['GET'])
def get_users():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('SELECT * FROM users;')
    users = cur.fetchall()
    cur.close()
    conn.close()
    return jsonify(users)

@app.route('/users', methods=['POST'])
def create_user():
    data = request.get_json()
    name = data['name']
    email = data['email']

    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('INSERT INTO users (name, email) VALUES (%s, %s) RETURNING id;', (name, email))
    user_id = cur.fetchone()[0]
    conn.commit()
    cur.close()
    conn.close()
    return jsonify({'id': user_id, 'name': name, 'email': email}), 201

@app.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('SELECT * FROM users WHERE id = %s;', (user_id,))
    user = cur.fetchone()
    cur.close()
    conn.close()
    if user is None:
        return jsonify({'error': 'User not found'}), 404
    return jsonify(user)

@app.route('/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    data = request.get_json()
    name = data['name']
    email = data['email']

    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('UPDATE users SET name = %s, email = %s WHERE id = %s RETURNING id;', (name, email, user_id))
    updated_user = cur.fetchone()
    conn.commit()
    cur.close()
    conn.close()
    if updated_user is None:
        return jsonify({'error': 'User not found'}), 404
    return jsonify({'id': updated_user[0], 'name': name, 'email': email})

@app.route('/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('DELETE FROM users WHERE id = %s RETURNING id;', (user_id,))
    deleted_user = cur.fetchone()
    conn.commit()
    cur.close()
    conn.close()
    if deleted_user is None:
        return jsonify({'error': 'User not found'}), 404
    return jsonify({'message': 'User deleted successfully'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

### **4. Run the Flask Application**
1. Ensure your virtual environment is activated:
   ```bash
   source venv/bin/activate
   ```
2. Run the Flask app:
   ```bash
   python3 app.py
   ```
3. The app should now be running on `http://<your-ec2-public-ip>:5000`.

---

### **5. Test the API**
Use tools like `curl` or Postman to test your API endpoints:
- **GET** `/users` - Fetch all users.
- **POST** `/users` - Create a new user.
- **GET** `/users/<user_id>` - Fetch a specific user.
- **PUT** `/users/<user_id>` - Update a user.
- **DELETE** `/users/<user_id>` - Delete a user.

---

### **6. Allow External Access to Flask**
By default, Flask only listens on `localhost`. To allow external access:
1. Update the `app.run()` line in `app.py`:
   ```python
   app.run(host='0.0.0.0', port=5000)
   ```
2. Ensure your EC2 security group allows inbound traffic on port 5000.

---

### **7. (Optional) Use a Production Server**
For production, use a WSGI server like `gunicorn`:
1. Install `gunicorn`:
   ```bash
   pip install gunicorn
   ```
2. Run the app with `gunicorn`:
   ```bash
   gunicorn -w 4 app:app
   ```

---

### **8. Debugging Tips**
- If you encounter errors, check the Flask logs in the terminal.
- Ensure the PostgreSQL service is running:
  ```bash
  sudo systemctl status postgresql
  ```
- Verify the database connection details in `app.py`.


Ubuntu Version :

# PostgreSQL Installation and Flask Setup Guide for Ubuntu

## Installing PostgreSQL on Ubuntu

### 1. Install PostgreSQL
```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib
```

### 2. Check PostgreSQL Status
```bash
sudo systemctl status postgresql
```

### 3. Switch to PostgreSQL User
```bash
sudo -i -u postgres
```

### 4. Access PostgreSQL Shell
```bash
psql
```

### 5. Create Database and User
```sql
CREATE DATABASE myflaskdb;
CREATE USER flaskuser WITH PASSWORD 'securepassword';
GRANT ALL PRIVILEGES ON DATABASE myflaskdb TO flaskuser;
ALTER USER flaskuser WITH SUPERUSER;
\q
```

### 6. Exit PostgreSQL User
```bash
exit
```

## Configure PostgreSQL for Remote Access (Optional)

### 1. Edit PostgreSQL Config
```bash
sudo nano /etc/postgresql/14/main/postgresql.conf
```
Uncomment/modify:
```
listen_addresses = '*'
```

### 2. Edit Client Authentication
```bash
sudo nano /etc/postgresql/14/main/pg_hba.conf
```
Add:
```
host    all             all             0.0.0.0/0               md5
```

### 3. Restart PostgreSQL
```bash
sudo systemctl restart postgresql
```

## Setting Up Flask with PostgreSQL

### 1. Install Python and Virtual Environment
```bash
sudo apt install -y python3 python3-pip python3-venv
```

### 2. Create Project Directory
```bash
mkdir flaskapp && cd flaskapp
```

### 3. Create Virtual Environment
```bash
python3 -m venv venv
source venv/bin/activate
```

### 4. Install Required Packages
```bash
pip install flask psycopg2-binary gunicorn
```

### 5. Create app.py
```python
from flask import Flask, request, jsonify
import psycopg2

app = Flask(__name__)

# Database connection
def get_db_connection():
    conn = psycopg2.connect(
        host='localhost',
        database='myflaskdb',
        user='flaskuser',
        password='securepassword'
    )
    return conn

# Create users table if not exists
def init_db():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            email VARCHAR(100) UNIQUE NOT NULL
        )
    ''')
    conn.commit()
    cur.close()
    conn.close()

# Routes
@app.route('/users', methods=['GET'])
def get_users():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('SELECT * FROM users;')
    users = cur.fetchall()
    cur.close()
    conn.close()
    return jsonify(users)

@app.route('/users', methods=['POST'])
def create_user():
    data = request.get_json()
    name = data['name']
    email = data['email']

    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('INSERT INTO users (name, email) VALUES (%s, %s) RETURNING id;', (name, email))
    user_id = cur.fetchone()[0]
    conn.commit()
    cur.close()
    conn.close()
    return jsonify({'id': user_id, 'name': name, 'email': email}), 201

if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=5000)
```

### 6. Run Flask Application
```bash
python3 app.py
```

## Running in Production with Gunicorn

### 1. Create WSGI Entry Point
```bash
nano wsgi.py
```
Add:
```python
from app import app

if __name__ == "__main__":
    app.run()
```

### 2. Run with Gunicorn
```bash
gunicorn -w 4 -b 0.0.0.0:5000 wsgi:app
```

### 3. Set Up Systemd Service (Optional)
```bash
sudo nano /etc/systemd/system/flaskapp.service
```
Add:
```
[Unit]
Description=Gunicorn instance to serve Flask app
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/flaskapp
Environment="PATH=/home/ubuntu/flaskapp/venv/bin"
ExecStart=/home/ubuntu/flaskapp/venv/bin/gunicorn -w 4 -b 0.0.0.0:5000 wsgi:app

[Install]
WantedBy=multi-user.target
```

### 4. Enable and Start Service
```bash
sudo systemctl daemon-reload
sudo systemctl start flaskapp
sudo systemctl enable flaskapp
```

## Firewall Configuration

### Allow Port 5000
```bash
sudo ufw allow 5000
```

## Troubleshooting

### Check PostgreSQL Logs
```bash
sudo tail -f /var/log/postgresql/postgresql-14-main.log
```

### Check Flask Application Logs
```bash
journalctl -u flaskapp -f
```

### Test Database Connection
```bash
psql -h localhost -U flaskuser -d myflaskdb -W
```

This guide provides a complete setup for running a Flask application with PostgreSQL on Ubuntu, including both development and production configurations.

