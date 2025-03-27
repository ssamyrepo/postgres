PostgreSQL replication setup using `pgactive` on Amazon RDS** across two AWS regions

---

## 🔧 **1. Prerequisites**

- Two AWS regions (e.g., `us-east-1`, `us-east-2`)
- PostgreSQL RDS instances in each region
- VPCs and public subnets in each region
- VPC peering between regions
- Security groups with open ports (5432) for each RDS

---

## 🌐 **2. Networking Setup**

### 🔹 VPC and Subnet Setup (in both regions)
- Create a VPC in **us-east-1** and **us-east-2**
- Create **public subnets** in each VPC
- Attach **Internet Gateway** to each VPC
- Set route tables to allow internet traffic

### 🔹 VPC Peering
- In **us-east-1**, create a VPC peering connection to the **us-east-2** VPC
- Go to **us-east-2** and **accept the peering request**
- **Edit route tables** in both regions:
  - Add routes for each other's VPC CIDR blocks
- Enable **DNS resolution** in both VPCs:
  - Go to VPC → **Edit DNS settings** → Check "Enable DNS resolution"

---

## 🛠️ **3. RDS Setup**

### 🔹 Parameter Group Setup (for each RDS)
- Create a **custom parameter group** for PostgreSQL 15 or higher
- Set:
  ```text
  rds.enable_pgactive = 1
  rds.custom_dns_resolution = 1
  ```
- Associate this parameter group during RDS instance creation
- Reboot RDS after assigning the custom parameter group

### 🔹 Create RDS Instances
- In **us-east-1**: Create RDS for PostgreSQL
- In **us-east-2**: Create another RDS for PostgreSQL
- DB name: `postgres`, username: `postgres`, password: `postgres12345`

### 🔹 Configure Security Groups
- Allow inbound PostgreSQL (port 5432) from the other RDS's **CIDR or SG**
- Save the rules

---

## 🗃️ **4. Database & Extension Setup**

### 🔹 On Source RDS (us-east-1)
```sql
-- Connect to postgres DB
CREATE DATABASE demo;
\c demo;
CREATE EXTENSION pgactive;

-- Verify shared_preload_libraries
SHOW shared_preload_libraries;

-- Create test table
CREATE SCHEMA inventory;
CREATE TABLE inventory.products (
  id INT PRIMARY KEY,
  product_name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO inventory.products (id, product_name) VALUES (1, 'Soap'), (2, 'Shampoo');
```

### 🔹 Initialize pgactive Group
```sql
SELECT pgactive.pgactive_create_group(
  node_name := 'node1',
  node_dsn := 'dbname=demo host=sourcedb-endpoint user=postgres password=postgres12345'
);
```

---

## 🌎 **5. Target RDS Setup (us-east-2)**

```sql
-- Connect to postgres DB
CREATE DATABASE demo;
\c demo;
CREATE EXTENSION pgactive;

-- Same schema and table
CREATE SCHEMA inventory;
CREATE TABLE inventory.products (
  id INT PRIMARY KEY,
  product_name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 🔹 Join pgactive Group
```sql
SELECT pgactive.pgactive_join_group(
  node_name := 'node2',
  node_dsn := 'dbname=demo host=targetdb-endpoint user=postgres password=postgres12345',
  join_using_dsn := 'dbname=demo host=sourcedb-endpoint user=postgres password=postgres12345'
);
```

---

## 🔁 **6. Validate Active-Active Replication**

- Insert data into **node2** and check it appears in **node1**
- Insert data into **node1** and check it appears in **node2**

---

## 🧪 **7. Simulate Failover Test**

1. **Stop RDS in us-east-1**
2. Insert data into us-east-2
3. Restart us-east-1
4. Check if data syncs

Repeat in reverse:
- Stop us-east-2 → Insert on us-east-1 → Restart → Validate replication

---

## 🧹 **8. Cleanup**

- Delete:
  - RDS Instances
  - Security Groups
  - VPC Peering
  - VPCs, Subnets, Route Tables, Internet Gateways

---

# Terraform script to set up VPCs, Subnets, Peering, Security Groups, and RDS instances for pgactive replication

provider "aws" {
  alias  = "use1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "use2"
  region = "us-east-2"
}

# Variables
variable "db_username" {
  default = "postgres"
}

variable "db_password" {
  default = "postgres12345"
}

variable "db_name" {
  default = "postgres"
}

# VPCs in both regions
resource "aws_vpc" "vpc_use1" {
  provider = aws.use1
  cidr_block = "10.0.0.0/16"
  enable_dns_support = true
  enable_dns_hostnames = true
  tags = { Name = "vpc-use1" }
}

resource "aws_vpc" "vpc_use2" {
  provider = aws.use2
  cidr_block = "10.1.0.0/16"
  enable_dns_support = true
  enable_dns_hostnames = true
  tags = { Name = "vpc-use2" }
}

# Subnets (Public for simplicity)
resource "aws_subnet" "subnet_use1" {
  provider = aws.use1
  vpc_id     = aws_vpc.vpc_use1.id
  cidr_block = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  tags = { Name = "subnet-use1" }
}

resource "aws_subnet" "subnet_use2" {
  provider = aws.use2
  vpc_id     = aws_vpc.vpc_use2.id
  cidr_block = "10.1.1.0/24"
  availability_zone = "us-east-2a"
  tags = { Name = "subnet-use2" }
}

# Internet Gateways
resource "aws_internet_gateway" "igw_use1" {
  provider = aws.use1
  vpc_id = aws_vpc.vpc_use1.id
}

resource "aws_internet_gateway" "igw_use2" {
  provider = aws.use2
  vpc_id = aws_vpc.vpc_use2.id
}

# Route Tables
resource "aws_route_table" "rt_use1" {
  provider = aws.use1
  vpc_id = aws_vpc.vpc_use1.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw_use1.id
  }
}

resource "aws_route_table" "rt_use2" {
  provider = aws.use2
  vpc_id = aws_vpc.vpc_use2.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw_use2.id
  }
}

# Route Table Associations
resource "aws_route_table_association" "rta_use1" {
  provider = aws.use1
  subnet_id      = aws_subnet.subnet_use1.id
  route_table_id = aws_route_table.rt_use1.id
}

resource "aws_route_table_association" "rta_use2" {
  provider = aws.use2
  subnet_id      = aws_subnet.subnet_use2.id
  route_table_id = aws_route_table.rt_use2.id
}

# Security Groups
resource "aws_security_group" "rds_sg_use1" {
  provider = aws.use1
  name   = "rds-sg-use1"
  vpc_id = aws_vpc.vpc_use1.id

  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["10.1.0.0/16"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "rds_sg_use2" {
  provider = aws.use2
  name   = "rds-sg-use2"
  vpc_id = aws_vpc.vpc_use2.id

  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# DB Subnet Groups
resource "aws_db_subnet_group" "db_subnet_group_use1" {
  provider = aws.use1
  name       = "pgactive-subnet-group-use1"
  subnet_ids = [aws_subnet.subnet_use1.id]
}

resource "aws_db_subnet_group" "db_subnet_group_use2" {
  provider = aws.use2
  name       = "pgactive-subnet-group-use2"
  subnet_ids = [aws_subnet.subnet_use2.id]
}

# Custom Parameter Groups with pgactive enabled
resource "aws_db_parameter_group" "pgactive_pg_use1" {
  provider = aws.use1
  name   = "pgactive-param-group-use1"
  family = "postgres15"

  parameter {
    name  = "rds.enable_pgactive"
    value = "1"
  }

  parameter {
    name  = "rds.custom_dns_resolution"
    value = "1"
  }
}

resource "aws_db_parameter_group" "pgactive_pg_use2" {
  provider = aws.use2
  name   = "pgactive-param-group-use2"
  family = "postgres15"

  parameter {
    name  = "rds.enable_pgactive"
    value = "1"
  }

  parameter {
    name  = "rds.custom_dns_resolution"
    value = "1"
  }
}

# RDS Instances
resource "aws_db_instance" "rds_use1" {
  provider = aws.use1
  identifier        = "pgactive-db-use1"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.t3.micro"
  allocated_storage = 20
  username          = var.db_username
  password          = var.db_password
  db_name           = var.db_name
  vpc_security_group_ids = [aws_security_group.rds_sg_use1.id]
  db_subnet_group_name   = aws_db_subnet_group.db_subnet_group_use1.name
  parameter_group_name   = aws_db_parameter_group.pgactive_pg_use1.name
  skip_final_snapshot    = true
  publicly_accessible    = true
}

resource "aws_db_instance" "rds_use2" {
  provider = aws.use2
  identifier        = "pgactive-db-use2"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.t3.micro"
  allocated_storage = 20
  username          = var.db_username
  password          = var.db_password
  db_name           = var.db_name
  vpc_security_group_ids = [aws_security_group.rds_sg_use2.id]
  db_subnet_group_name   = aws_db_subnet_group.db_subnet_group_use2.name
  parameter_group_name   = aws_db_parameter_group.pgactive_pg_use2.name
  skip_final_snapshot    = true
  publicly_accessible    = true
}
