To deploy **PostgreSQL** on an EKS cluster using the **AWS Controllers for Kubernetes (ACK)**, follow the steps below. This guide assumes you already have an EKS cluster (`eksctl-Three-Tier-K8s-EKS-Cluster`) set up and configured.

---

### **Step 1: Prerequisites**
1. **EKS Cluster**:
   - Ensure your EKS cluster is running and accessible.
   - Verify access to the cluster:
     ```bash
     kubectl get nodes
     ```

2. **Install Required Tools**:
   - **AWS CLI**: Configured with the necessary permissions.
   - **kubectl**: Configured to access your EKS cluster.
   - **Helm**: For installing the ACK controller.
   - **eksctl**: For managing the EKS cluster (optional).

3. **IAM Permissions**:
   - Ensure the IAM user or role has permissions to create and manage RDS instances.

---

### **Step 2: Install the ACK RDS Controller**
1. **Add the ACK Helm Repository**:
   ```bash
   helm repo add aws-controllers https://aws-controllers-k8s.github.io/charts
   helm repo update
   ```

2. **Install the ACK RDS Controller**:
   ```bash
   export AWS_REGION=us-east-1  # Replace with your region
   export ACK_SYSTEM_NAMESPACE=ack-system

   helm install --create-namespace -n $ACK_SYSTEM_NAMESPACE ack-rds-controller \
     aws-controllers/ack-rds-controller \
     --set aws.region=$AWS_REGION
   ```

3. **Verify Installation**:
   Check if the ACK RDS controller pod is running:
   ```bash
   kubectl get pods -n $ACK_SYSTEM_NAMESPACE
   ```

---

### **Step 3: Configure IAM Roles for Service Accounts (IRSA)**
1. **Create an IAM Policy**:
   Create an IAM policy for the ACK RDS controller to manage RDS resources:
   ```bash
   cat <<EOF > ack-rds-policy.json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Action": [
                   "rds:*",
                   "ec2:DescribeSubnets",
                   "ec2:DescribeVpcs",
                   "ec2:DescribeSecurityGroups"
               ],
               "Resource": "*"
           }
       ]
   }
   EOF

   aws iam create-policy --policy-name ACKRDSPolicy --policy-document file://ack-rds-policy.json
   ```

2. **Create an IAM Role for the Service Account**:
   Replace `<your-account-id>` and `<your-cluster-name>` with your AWS account ID and EKS cluster name:
   ```bash
   eksctl create iamserviceaccount \
       --name ack-rds-controller \
       --namespace $ACK_SYSTEM_NAMESPACE \
       --cluster <your-cluster-name> \
       --attach-policy-arn arn:aws:iam::<your-account-id>:policy/ACKRDSPolicy \
       --approve
   ```

3. **Verify the Service Account**:
   Ensure the service account is created and annotated with the IAM role:
   ```bash
   kubectl describe serviceaccount ack-rds-controller -n $ACK_SYSTEM_NAMESPACE
   ```

---

### **Step 4: Deploy PostgreSQL Using ACK**
1. **Create a Namespace for PostgreSQL**:
   ```bash
   kubectl create namespace postgres-ns
   ```

2. **Create a Kubernetes Secret for PostgreSQL Credentials**:
   Replace `<username>` and `<password>` with your desired PostgreSQL credentials:
   ```bash
   kubectl create secret generic postgres-credentials \
       --namespace postgres-ns \
       --from-literal=username=<username> \
       --from-literal=password=<password>
   ```

3. **Deploy PostgreSQL**:
   Create a YAML file (`postgresql.yaml`) for the PostgreSQL RDS instance:
   ```yaml
   apiVersion: rds.services.k8s.aws/v1alpha1
   kind: DBInstance
   metadata:
     name: postgresql-instance
     namespace: postgres-ns
   spec:
     dbInstanceIdentifier: postgresql-instance
     dbInstanceClass: db.t3.micro
     engine: postgres
     engineVersion: "13.4"
     masterUsername:
       secretKeyRef:
         name: postgres-credentials
         key: username
     masterUserPassword:
       secretKeyRef:
         name: postgres-credentials
         key: password
     allocatedStorage: 20
     publiclyAccessible: true
     dbSubnetGroupName: default
     vpcSecurityGroupIDs:
       - sg-xxxxxxxx  # Replace with your security group ID
   ```

4. **Apply the Manifest**:
   Deploy the PostgreSQL instance:
   ```bash
   kubectl apply -f postgresql.yaml
   ```

5. **Verify the Deployment**:
   Check the status of the RDS instance:
   ```bash
   kubectl get dbinstance -n postgres-ns
   ```

---

### **Step 5: Access PostgreSQL**
1. **Get the Endpoint**:
   Retrieve the endpoint of the PostgreSQL instance:
   ```bash
   kubectl get dbinstance postgresql-instance -n postgres-ns -o jsonpath='{.status.endpointAddress}'
   ```

2. **Connect to PostgreSQL**:
   Use the endpoint, username, and password to connect to the PostgreSQL database:
   ```bash
   psql -h <endpoint> -U <username> -d postgres
   ```

---

### **Step 6: Clean Up**
1. **Delete the PostgreSQL Instance**:
   ```bash
   kubectl delete dbinstance postgresql-instance -n postgres-ns
   ```

2. **Uninstall the ACK RDS Controller**:
   ```bash
   helm uninstall ack-rds-controller -n $ACK_SYSTEM_NAMESPACE
   ```

3. **Delete the IAM Role and Policy**:
   ```bash
   eksctl delete iamserviceaccount --name ack-rds-controller --namespace $ACK_SYSTEM_NAMESPACE --cluster <your-cluster-name>
   aws iam delete-policy --policy-arn arn:aws:iam::<your-account-id>:policy/ACKRDSPolicy
   ```

---

### **Troubleshooting**
- **RDS Instance Not Created**:
  - Check the logs of the ACK RDS controller:
    ```bash
    kubectl logs -n $ACK_SYSTEM_NAMESPACE -l app.kubernetes.io/instance=ack-rds-controller
    ```
  - Ensure the IAM role has the correct permissions.

- **Connection Issues**:
  - Verify the security group allows inbound traffic on port `5432`.
  - Ensure the PostgreSQL instance is publicly accessible (if required).


To resolve the "InvalidSubnet: No default subnet detected in VPC" error and successfully deploy your PostgreSQL instance using the AWS Controllers for Kubernetes (ACK) in your Amazon EKS cluster, you'll need to create a **DB Subnet Group** that includes your existing subnets. This subnet group will inform Amazon RDS about which subnets within your VPC it can utilize for the database instance.

**Steps to Create a DB Subnet Group and Deploy PostgreSQL:**

1. **Identify Your Subnets:**
   Based on the information you've provided, you have the following subnets:
   - **Private Subnet in us-east-1f:**
     - Subnet ID: `subnet-079948fd64452326e`
     - CIDR: `192.168.64.0/19`
   - **Private Subnet in us-east-1c:**
     - Subnet ID: `subnet-053d9e501c1107821`
     - CIDR: `192.168.96.0/19`

   These subnets are in different Availability Zones (AZs), which is a requirement for creating a DB Subnet Group.

2. **Create a DB Subnet Group:**
   A DB Subnet Group is a collection of subnets that RDS uses to allocate IP addresses for your DB instances in a VPC. You can create this group using the AWS Management Console, AWS CLI, or directly through Kubernetes manifests with ACK.

   - **Using AWS CLI:**
     Ensure you have the AWS CLI installed and configured with the necessary permissions.

     ```bash
     aws rds create-db-subnet-group \
       --db-subnet-group-name my-db-subnet-group \
       --db-subnet-group-description "My DB Subnet Group for EKS" \
       --subnet-ids subnet-079948fd64452326e subnet-053d9e501c1107821
     ```

     This command creates a DB Subnet Group named `my-db-subnet-group` that includes your specified subnets.

   - **Using ACK with Kubernetes Manifest:**
     If you prefer managing AWS resources through Kubernetes, you can define the DB Subnet Group using a Kubernetes manifest.

     ```yaml
     apiVersion: rds.services.k8s.aws/v1alpha1
     kind: DBSubnetGroup
     metadata:
       name: my-db-subnet-group
     spec:
       name: my-db-subnet-group
       description: "My DB Subnet Group for EKS"
       subnetIDs:
         - subnet-079948fd64452326e
         - subnet-053d9e501c1107821
     ```

     Save this manifest as `db-subnet-group.yaml` and apply it using:

     ```bash
     kubectl apply -f db-subnet-group.yaml
     ```

     This approach leverages the ACK RDS controller to manage the creation of the DB Subnet Group.

3. **Modify Your PostgreSQL DBInstance Manifest:**
   After creating the DB Subnet Group, update your existing `DBInstance` manifest to reference this subnet group.

   ```yaml
   apiVersion: rds.services.k8s.aws/v1alpha1
   kind: DBInstance
   metadata:
     name: eksrdslab
   spec:
     allocatedStorage: 20
     dbInstanceClass: db.t4g.micro
     dbInstanceIdentifier: eksrdslab
     engine: postgres
     engineVersion: "14"
     masterUsername: "postgres"
     masterUserPassword:
       namespace: default
       name: eksrdslab-password
       key: password
     dbSubnetGroupName: my-db-subnet-group
   ```

   Ensure the `dbSubnetGroupName` field matches the name of the DB Subnet Group you created.

4. **Apply the Updated DBInstance Manifest:**
   Deploy the updated `DBInstance` manifest to your EKS cluster:

   ```bash
   kubectl apply -f your-dbinstance-manifest.yaml
   ```

   Replace `your-dbinstance-manifest.yaml` with the actual filename of your manifest.

5. **Verify the Deployment:**
   Monitor the status of your DBInstance to ensure it's created successfully:

   ```bash
   kubectl describe dbinstance eksrdslab
   ```

   Look for a `Status` condition indicating that the instance is `available`.

By following these steps, you configure your RDS instance to utilize the specified subnets within your VPC, resolving the subnet-related error and ensuring proper deployment within your EKS environment. 

Deploying PostgreSQL within Amazon Elastic Kubernetes Service (EKS) pods offers several advantages over traditional methods:

**1. Cost Efficiency and Flexibility:**
Operating PostgreSQL on EKS can lead to significant cost savings. By utilizing Kubernetes' orchestration capabilities, resources can be optimized, and expenses associated with managed services like Amazon RDS can be reduced. This approach provides greater control over configurations and scaling, allowing for tailored solutions that align with specific workload requirements. citeturn0search2

**2. Enhanced Scalability and Automation:**
Kubernetes excels at automating deployment, scaling, and management of containerized applications. Running PostgreSQL on EKS leverages these features, enabling seamless scaling and efficient management of database instances. Tools like PGO (Postgres Operator) further streamline these processes, facilitating automated backups, failover, and maintenance tasks. citeturn0search0

**3. Unified Infrastructure Management:**
Integrating PostgreSQL into your EKS environment consolidates your infrastructure, allowing for consistent management of both application and database components. This unification simplifies operations, reduces the complexity associated with managing disparate systems, and fosters a more cohesive development and deployment pipeline.

**4. Avoidance of Vendor Lock-In:**
Deploying PostgreSQL on EKS offers flexibility in terms of infrastructure choices, potentially reducing dependency on a single cloud provider. This approach can be advantageous for organizations seeking to maintain portability and avoid vendor lock-in. citeturn0search7

**5. Advanced Monitoring and Customization:**
Running PostgreSQL within EKS allows for the integration of sophisticated monitoring and logging tools. Operators often provide metrics and logging capabilities, enabling proactive management of database performance and health. This setup also permits extensive customization to meet unique application requirements. citeturn0search0

However, it's important to note that managing PostgreSQL on EKS requires a certain level of expertise in Kubernetes and database administration. While this approach offers greater control and potential cost benefits, it also entails responsibilities such as handling backups, updates, and ensuring high availability—tasks typically managed by services like Amazon RDS. Therefore, organizations should carefully assess their operational capabilities and specific needs before opting for this deployment strategy. 
