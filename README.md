# Oracle Free 23ai on EC2 (Docker) with EBS and Sample Schemas

This guide deploys Oracle Database Free 23ai in Docker on EC2 with persistent EBS storage and installs the HR and Customer Orders sample schemas. It also provides a separate template to create a client VPC and VPC peering for private access.

References:
- Oracle Sample Schemas (HR, CO): `https://github.com/oracle-samples/db-sample-schemas`

## What gets created
- EC2 with Docker, auto-assigned public IP, SG allowing SSH (22) and Oracle listener (1521)
- EBS volume, mounted at `/opt/oracle/oradata`, mapped into container for persistence
- Docker container `sharvan-kumar-afe-oracle-free` using `gvenzl/oracle-free:latest`
- Automated install of HR and CO schemas into `FREEPDB1` with sample data
- Uses existing VPC and subnet (no new networking resources created)

## Prerequisites
- AWS account with permissions to create EC2/IAM/EBS
- Existing EC2 key pair name
- Existing VPC with public subnet and Internet Gateway
- Docker daemon access (automatically configured)

## Step 1: Find Your VPC and Subnet IDs

### Option A: Using AWS Console
1. Go to **VPC Console** → **VPCs**
2. Note your VPC ID (e.g., `vpc-12345678`)
3. Go to **Subnets** → Find a public subnet (one with route to IGW)
4. Note the Subnet ID (e.g., `subnet-12345678`)

### Option B: Using AWS CLI
```bash
# List VPCs
aws ec2 describe-vpcs --query 'Vpcs[*].[VpcId,Tags[?Key==`Name`].Value|[0]]' --output table

# List public subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-12345678" --query 'Subnets[*].[SubnetId,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' --output table
```

## Step 2: Deploy Main Stack (Oracle Database)

1. **Open AWS CloudFormation** → Create stack with new resources
2. **Upload template**: `main-oracle-free23_withexistingVPC.yaml`
3. **Set parameters**:
   - `KeyPairName`: Your existing EC2 key pair name
   - `VpcId`: Your existing VPC ID (e.g., `vpc-12345678`)
   - `SubnetId`: Your existing public subnet ID (e.g., `subnet-12345678`)
   - `AllowedSSHLocation`: Your IP/CIDR (e.g., `203.0.113.10/32`) or `0.0.0.0/0` for demo
   - `AllowedDBClientCidr`: Your IP/CIDR or `0.0.0.0/0` for demo
   - `OraclePassword`: Strong password (12+ characters)
   - `InstanceType`: `t3.large` (default)
   - `VolumeSizeGiB`: `200` (default)
4. **Create stack** and wait for completion (~15-20 minutes)
5. **Note the outputs**:
   - `InstancePublicIp`: Public IP for database access
   - `InstancePrivateIp`: Private IP for VPC peering
   - `SecurityGroupId`: For VPC peering setup

## Step 3: Connect to the Database

### From Your Local Machine
- **Host**: `InstancePublicIp` (from CloudFormation outputs)
- **Port**: `1521`
- **Service**: `FREEPDB1`
- **Users**: 
  - `system/<your-password>` (admin access)
  - `hr/hr` (HR schema access)
  - `co/co` (CO schema access)

### SSH to EC2 Instance
```bash
ssh -i <your-key.pem> ec2-user@<InstancePublicIp>
```

### Docker Commands (from EC2)
```bash
# Check container status
docker ps

# View Oracle logs
docker logs sharvan-kumar-afe-oracle-free

# Connect to container bash
docker exec -it sharvan-kumar-afe-oracle-free bash

# Connect to database
docker exec -it sharvan-kumar-afe-oracle-free sqlplus system/<password>@localhost:1521/FREEPDB1
```

## Restrict access to only your client IP later
- Update the main stack with a narrower `AllowedDBClientCidr` (e.g., `203.0.113.10/32`).
- Alternatively, edit the security group inbound rule for TCP 1521 to your client IP/CIDR.

## VPC Peering (private access from VPC B)
Use `vpc-b-peering.yaml` as a separate stack.
1. From the main stack Outputs, collect:
   - `VpcId` (as VpcAId)
   - `PublicRouteTableId` (as VpcAPublicRouteTableId)
   - `VpcCidr` (the value you used; default `10.20.0.0/16`) as VpcACidr
   - `SecurityGroupId` (as DBSecurityGroupId)
2. Create stack with `vpc-b-peering.yaml` with parameters:
   - VpcAId, VpcACidr, VpcAPublicRouteTableId, DBSecurityGroupId
   - VpcBCidr (default `10.30.0.0/16`) and `PublicSubnetCidrB` (default `10.30.0.0/24`)
3. After completion:
   - Test from an instance in VPC B to `InstancePrivateIp:1521` (service `FREEPDB1`).

## Data persistence
- Data is on EBS mounted at `/opt/oracle/oradata` and bound into the container (`/opt/oracle/oradata:/opt/oracle/oradata`). DB state survives container/instance restarts.

## Step 4: Verify Sample Schemas

### Check Container Status
```bash
# SSH to EC2 instance
ssh -i <your-key.pem> ec2-user@<InstancePublicIp>

# Check container logs
docker logs sharvan-kumar-afe-oracle-free | tail -n 50
```

### Verify HR Schema
```bash
# Connect as HR user and check tables
docker exec -it sharvan-kumar-afe-oracle-free sqlplus hr/hr@localhost:1521/FREEPDB1 << EOF
SELECT table_name FROM user_tables ORDER BY table_name;
SELECT COUNT(*) FROM employees;
SELECT COUNT(*) FROM departments;
EXIT;
EOF
```

### Verify CO Schema
```bash
# Connect as CO user and check tables
docker exec -it sharvan-kumar-afe-oracle-free sqlplus co/co@localhost:1521/FREEPDB1 << EOF
SELECT table_name FROM user_tables ORDER BY table_name;
SELECT COUNT(*) FROM customers;
SELECT COUNT(*) FROM orders;
EXIT;
EOF
```

### Check Schema Installation from SYSTEM
```bash
# Connect as SYSTEM and verify schemas
docker exec -it sharvan-kumar-afe-oracle-free sqlplus system/<password>@localhost:1521/FREEPDB1 << EOF
SELECT 'HR Tables: ' || COUNT(*) FROM dba_tables WHERE owner = 'HR';
SELECT 'CO Tables: ' || COUNT(*) FROM dba_tables WHERE owner = 'CO';
SELECT table_name FROM dba_tables WHERE owner = 'HR' ORDER BY table_name;
EXIT;
EOF
```

## Teardown (graceful delete)
- Delete the VPC B + peering stack first (if created).
- Then delete the main DB stack. CloudFormation deletes the EIP, EC2, SG, and VPC. The EBS data volume is set to DeleteOnTermination=true (default in template). If you want to retain data, set that flag off before deletion.

## Security notes
- For demos, defaults allow broad access. For stricter setups:
  - Set `AllowedSSHLocation` and `AllowedDBClientCidr` to specific /32 client IPs.
  - Prefer private-only DB access via VPC peering.

## Troubleshooting
- DB readiness: The user data waits up to ~30 minutes for `DATABASE IS READY TO USE!`.
- Schema install logs: see `docker logs oracle-free`. You can re-run schema scripts inside the container under `/tmp/schemas`.

## SSH access and key generation

You must provide a Key Pair name when launching the main stack. Use one of the following approaches:

### Option A: Create key pair in AWS Console (quickest)
1. In EC2 Console → Key Pairs → Create key pair.
2. Type: RSA (or ED25519), Name: e.g., `oracle-free23-key`.
3. Download the private key file (`.pem`). Keep it safe.
4. Use this name for the `KeyPairName` parameter when deploying the main stack.

### Option B: Generate locally and import public key
- macOS/Linux:
  - Generate ED25519 key (recommended):
    ```bash
    ssh-keygen -t ed25519 -C "oracle-free23" -f ~/.ssh/oracle-free23
    ```
    This creates `~/.ssh/oracle-free23` (private) and `~/.ssh/oracle-free23.pub` (public).
  - Copy contents of `~/.ssh/oracle-free23.pub`.
  - In EC2 Console → Key Pairs → Import key pair. Give it a name (e.g., `oracle-free23-key`) and paste the public key.
  - Use that name for `KeyPairName` when launching the main stack.
- Windows (PowerShell with OpenSSH):
  ```powershell
  ssh-keygen -t ed25519 -C "oracle-free23" -f $env:USERPROFILE\.ssh\oracle-free23
  ```
  Then import `oracle-free23.pub` into EC2 as above.
- Windows (PuTTY):
  - Use PuTTYgen → Generate an RSA or ED25519 key → Save private key (`.ppk`) and copy the OpenSSH formatted public key.
  - Import the public key to EC2 Key Pairs. For PuTTY connections, load the `.ppk` in PuTTY.

### Connect over SSH
- Find `InstancePublicIp` in the main stack Outputs.
- macOS/Linux:
  ```bash
  chmod 600 ~/.ssh/oracle-free23.pem   # if you used a .pem from console
  ssh -i ~/.ssh/oracle-free23.pem ec2-user@<InstancePublicIp>
  ```
  If you imported your own key, use the corresponding private key path (e.g., `~/.ssh/oracle-free23`).
- Windows (PowerShell OpenSSH):
  ```powershell
  ssh -i $env:USERPROFILE\.ssh\oracle-free23 ec2-user@<InstancePublicIp>
  ```
- Windows (PuTTY):
  - Host Name: `<InstancePublicIp>`
  - User: `ec2-user`
  - Auth → Private key file: select your `.ppk` from PuTTYgen

Notes
- Username is `ec2-user` (Amazon Linux 2023 AMI).
- Ensure your local network/firewall allows outbound TCP 22.
- The security group must allow your client IP in `AllowedSSHLocation` (default is open for demo; restrict for security).
