To run a simple `pgbench` benchmark on your PostgreSQL RDS instance (`rds-pg-labs.cj2s8sqw4bta.us-east-1.rds.amazonaws.com`) using the provided environment variables, we need to:
1. Verify if `pgbench` is installed on your Ubuntu EC2 instance.
2. Install `pgbench` if it’s not already installed.
3. Initialize a `pgbench` database and run a simple benchmark.
4. Ensure the connection to the RDS instance works with the provided SSL settings and credentials.

The environment variables you provided are mostly correct, but note that `DBENDP` and `PGHOST` reference `rds-pg-lab` (singular) while previous interactions used `rds-pg-labs` (plural). I’ll assume the correct endpoint is `rds-pg-labs.cj2s8sqw4bta.us-east-1.rds.amazonaws.com` based on prior context, but please confirm. Additionally, previous attempts showed issues with the password (`Password`) and SSL certificate verification, so we’ll address those as needed.

### Step-by-Step Instructions

#### Step 1: Verify if `pgbench` is Installed
Check if `pgbench` is installed on the Ubuntu EC2 instance:
```bash
pgbench --version
```

- **If installed**, you’ll see output like:
  ```
  pgbench (PostgreSQL) 16.x or similar
  ```
  Proceed to Step 3.
- **If not installed**, you’ll see an error like `command not found`. Proceed to Step 2.

#### Step 2: Install `pgbench`
The `pgbench` tool is part of the `postgresql-contrib` package. Install it:

```bash
# Update package list
sudo apt update

# Install postgresql-contrib to get pgbench
sudo apt install -y postgresql-contrib
```

Verify the installation:
```bash
pgbench --version
```

Expected output:
```
pgbench (PostgreSQL) 16.x or similar
```

#### Step 3: Verify Environment Variables
Ensure all environment variables are set correctly:
```bash
echo "DBENDP: $DBENDP"
echo "DBUSER: $DBUSER"
echo "DBPASS: $DBPASS"
echo "DBPORT: $DBPORT"
echo "PGHOST: $PGHOST"
echo "PGUSER: $PGUSER"
echo "PGPASSWORD: $PGPASSWORD"
echo "AWSREGION: $AWSREGION"
echo "PGSSLMODE: $PGSSLMODE"
echo "PGSSLROOTCERT: $PGSSLROOTCERT"
```

Expected output (based on your input, correcting `DBENDP` to match `PGHOST`):
```
DBENDP: rds-pg-labs.cj2s8sqw4bta.us-east-1.rds.amazonaws.com
DBUSER: postgres
DBPASS: password
DBPORT: 5432
PGHOST: rds-pg-labs.cj2s8sqw4bta.us-east-1.rds.amazonaws.com
PGUSER: postgres
PGPASSWORD: password
AWSREGION: us-east-1
PGSSLMODE: verify-full
PGSSLROOTCERT: /home/ubuntu/rds-combined-ca-bundle.pem
```

If `DBENDP` shows `rds-pg-lab` (singular), update it to match `PGHOST`:
```bash
export DBENDP=rds-pg-labs.cj2s8sqw4bta.us-east-1.rds.amazonaws.com
sed -i 's/export DBENDP=rds-pg-lab/export DBENDP=rds-pg-labs/' /home/ubuntu/.bashrc
source /home/ubuntu/.bashrc
```

#### Step 4: Verify SSL Certificate
Previous attempts showed an `SSL error: certificate verify failed`. Use the region-specific certificate (`us-east-1-bundle.pem`) as recommended earlier:

```bash
# Remove old certificate
rm -f /home/ubuntu/rds-combined-ca-bundle.pem

# Download us-east-1 certificate
curl -o /home/ubuntu/us-east-1-bundle.pem https://truststore.pki.rds.amazonaws.com/us-east-1/us-east-1-bundle.pem

# Update environment variable
export PGSSLROOTCERT=/home/ubuntu/us-east-1-bundle.pem
sed -i 's|export PGSSLROOTCERT=/home/ubuntu/rds-combined-ca-bundle.pem|export PGSSLROOTCERT=/home/ubuntu/us-east-1-bundle.pem|' /home/ubuntu/.bashrc
source /home/ubuntu/.bashrc
```

Verify the certificate file exists:
```bash
ls -l /home/ubuntu/us-east-1-bundle.pem
```

#### Step 5: Test RDS Connection
Before running `pgbench`, verify connectivity to the `pglab` database:
```bash
psql "host=$PGHOST port=$DBPORT user=$PGUSER password=$PGPASSWORD dbname=pglab sslmode=$PGSSLMODE sslrootcert=$PGSSLROOTCERT"
```

If the connection fails:
- **Password authentication failed**: The password `password` may be incorrect. Reset the RDS master password:
  ```bash
  aws rds modify-db-instance \
    --db-instance-identifier rds-pg-labs \
    --master-user-password "password" \
    --apply-immediately
  ```
  Or use a new password and update:
  ```bash
  export DBPASS="new_password"
  export PGPASSWORD="new_password"
  sed -i 's/export DBPASS="password"/export DBPASS="new_password"/' /home/ubuntu/.bashrc
  sed -i 's/export PGPASSWORD="password3889"/export PGPASSWORD="new_password"/' /home/ubuntu/.bashrc
  source /home/ubuntu/.bashrc
  ```
- **SSL error: certificate verify failed**: Try `sslmode=require`:
  ```bash
  psql "host=$PGHOST port=$DBPORT user=$PGUSER password=$PGPASSWORD dbname=pglab sslmode=require"
  ```
  If `require` works, update `PGSSLMODE`:
  ```bash
  export PGSSLMODE=require
  sed -i 's/export PGSSLMODE=verify-full/export PGSSLMODE=require/' /home/ubuntu/.bashrc
  source /home/ubuntu/.bashrc
  ```
- **Connection refused**: Verify security groups:
  - RDS security group (`sg-007cae376e5791402`): Allow inbound TCP `5432` from EC2’s IP (`10.0.11.150/32`) or security group.
  - EC2 security group: Allow outbound TCP `5432` to RDS’s IP (`10.0.33.133/32`) or security group.
  Test connectivity:
  ```bash
  nc -zv $PGHOST 5432
  ```

#### Step 6: Initialize `pgbench` Database
`pgbench` requires a database with its test schema. Since `pglab` exists (per RDS configuration), initialize it with `pgbench` tables:

```bash
pgbench -i -h $PGHOST -p $DBPORT -U $PGUSER -d pglab --no-vacuum "sslmode=$PGSSLMODE sslrootcert=$PGSSLROOTCERT"
```

- `-i`: Initializes the `pgbench` schema.
- `--no-vacuum`: Skips vacuuming for faster setup.
- The command creates tables (`pgbench_accounts`, `pgbench_branches`, `pgbench_tellers`, `pgbench_history`) in the `pglab` database.

If initialization fails due to permissions, ensure the `postgres` user has sufficient privileges. Alternatively, connect to `pglab` and create the schema manually:
```bash
psql "host=$PGHOST port=$DBPORT user=$PGUSER password=$PGPASSWORD dbname=pglab sslmode=$PGSSLMODE sslrootcert=$PGSSLROOTCERT" -c "CREATE SCHEMA pgbench; CREATE TABLE pgbench_accounts (aid bigint not null, bid bigint, abalance bigint, filler char(84));"
```

#### Step 7: Run a Simple `pgbench` Benchmark
Run a basic `pgbench` test to measure performance:
```bash
pgbench -h $PGHOST -p $DBPORT -U $PGUSER -d pglab -c 10 -j 2 -t 1000 "sslmode=$PGSSLMODE sslrootcert=$PGSSLROOTCERT"
```

- `-c 10`: 10 concurrent clients.
- `-j 2`: 2 worker threads.
- `-t 1000`: Each client runs 1000 transactions.
- The default test uses a simple TPC-B-like workload.

Expected output includes metrics like:
```
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 2
number of transactions per client: 1000
number of transactions actually processed: 10000/10000
latency average = X.XXX ms
tps = XXX.XXX (including connections establishing)
tps = XXX.XXX (excluding connections establishing)
```

#### Step 8: Persist Environment Variables
Ensure all environment variables are saved:
```bash
cat << 'EOF' > /home/ubuntu/.bashrc
export PYENV_ROOT="$HOME/.pyenv"
command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
export PATH=$PATH:/usr/local/bin:/usr/bin
export DBPASS="password3889"  # Replace with correct password
export DBUSER=postgres
export DBENDP=rds-pg-labs.cj2s8sqw4bta.us-east-1.rds.amazonaws.com
export DBPORT=5432
export PGUSER=postgres
export PGPASSWORD="password3889"  # Replace with correct password
export PGHOST=rds-pg-labs.cj2s8sqw4bta.us-east-1.rds.amazonaws.com
export AWSREGION=us-east-1
export PGSSLMODE=verify-full  # Use require if verify-full fails
export PGSSLROOTCERT=/home/ubuntu/us-east-1-bundle.pem
EOF

source /home/ubuntu/.bashrc
```

#### Full Workflow
```bash
# Step 1: Check if pgbench is installed
pgbench --version || {
  # Step 2: Install pgbench
  sudo apt update
  sudo apt install -y postgresql-contrib
}

# Verify pgbench
pgbench --version

# Step 3: Verify environment variables
echo "DBENDP: $DBENDP"
echo "DBUSER: $DBUSER"
echo "DBPASS: $DBPASS"
echo "DBPORT: $DBPORT"
echo "PGHOST: $PGHOST"
echo "PGUSER: $PGUSER"
echo "PGPASSWORD: $PGPASSWORD"
echo "AWSREGION: $AWSREGION"
echo "PGSSLMODE: $PGSSLMODE"
echo "PGSSLROOTCERT: $PGSSLROOTCERT"

# Step 4: Update SSL certificate
rm -f /home/ubuntu/rds-combined-ca-bundle.pem
curl -o /home/ubuntu/us-east-1-bundle.pem https://truststore.pki.rds.amazonaws.com/us-east-1/us-east-1-bundle.pem
export PGSSLROOTCERT=/home/ubuntu/us-east-1-bundle.pem
sed -i 's|export PGSSLROOTCERT=/home/ubuntu/rds-combined-ca-bundle.pem|export PGSSLROOTCERT=/home/ubuntu/us-east-1-bundle.pem|' /home/ubuntu/.bashrc
source /home/ubuntu/.bashrc

# Step 5: Test RDS connection
psql "host=$PGHOST port=$DBPORT user=$PGUSER password=$PGPASSWORD dbname=pglab sslmode=$PGSSLMODE sslrootcert=$PGSSLROOTCERT"

# Step 6: Initialize pgbench schema
pgbench -i -h $PGHOST -p $DBPORT -U $PGUSER -d pglab --no-vacuum "sslmode=$PGSSLMODE sslrootcert=$PGSSLROOTCERT"

# Step 7: Run pgbench
pgbench -h $PGHOST -p $DBPORT -U $PGUSER -d pglab -c 10 -j 2 -t 1000 "sslmode=$PGSSLMODE sslrootcert=$PGSSLROOTCERT"

# Step 8: Persist environment variables
cat << 'EOF' > /home/ubuntu/.bashrc
export PYENV_ROOT="$HOME/.pyenv"
command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
export PATH=$PATH:/usr/local/bin:/usr/bin
export DBPASS="password3889"  # Replace with correct password
export DBUSER=postgres
export DBENDP=rds-pg-labs.cj2s8sqw4bta.us-east-1.rds.amazonaws.com
export DBPORT=5432
export PGUSER=postgres
export PGPASSWORD="password3889"  # Replace with correct password
export PGHOST=rds-pg-labs.cj2s8sqw4bta.us-east-1.rds.amazonaws.com
export AWSREGION=us-east-1
export PGSSLMODE=verify-full  # Use require if verify-full fails
export PGSSLROOTCERT=/home/ubuntu/us-east-1-bundle.pem
EOF

source /home/ubuntu/.bashrc
```

### Troubleshooting
- **If `pgbench` fails to initialize**:
  - Ensure the `pglab` database is accessible:
    ```bash
    psql "host=$PGHOST port=$DBPORT user=$PGUSER password=$PGPASSWORD dbname=pglab sslmode=$PGSSLMODE sslrootcert=$PGSSLROOTCERT" -c "\l"
    ```
  - Check for permission errors and verify the `postgres` user has `CREATEDB` privileges.
- **If SSL errors persist**:
  - Switch to `sslmode=require`:
    ```bash
    pgbench -h $PGHOST -p $DBPORT -U $PGUSER -d pglab -c 10 -j 2 -t 1000 "sslmode=require"
    ```
  - Verify the certificate:
    ```bash
    openssl x509 -in /home/ubuntu/us-east-1-bundle.pem -text -noout
    ```
- **If password authentication fails**:
  - Reset the password via AWS CLI (as shown in Step 5).
  - If using the secret ARN from the previous script, retrieve the password:
    ```bash
    aws secretsmanager get-secret-value --secret-id "<SECRET_ARN>" --region us-east-1 --query SecretString --output text
    ```
- **If connection refused**:
  - Recheck security groups:
    ```bash
    aws ec2 describe-security-groups --group-ids sg-007cae376e5791402
    ```
  - Add inbound rule if needed:
    ```bash
    aws ec2 authorize-security-group-ingress --group-id sg-007cae376e5791402 --protocol tcp --port 5432 --cidr 10.0.11.150/32
    ```

Please confirm the RDS endpoint (`rds-pg-lab` vs. `rds-pg-labs`) and run the workflow. Share any errors or outputs if issues persist!
