# aws-rds-mysql-guide
<div align="center">

![AWS RDS Explained: Launch, Secure, and Scale a Managed Database](assets/rds-cover.png)

# AWS RDS Explained: Launch, Secure, and Scale a Managed Database

**A hands-on guide to Amazon RDS: launch a private MySQL database, connect from EC2, test Multi-AZ failover, and add a read replica.**

![AWS](https://img.shields.io/badge/AWS-RDS-FF9900?logo=amazonaws&logoColor=white)
![Engine](https://img.shields.io/badge/Engine-MySQL-4479A1?logo=mysql&logoColor=white)
![Level](https://img.shields.io/badge/Level-Beginner-2DD4BF)

`#vishalcloudjourney`

</div>

---

## Table of Contents

- [What Is Amazon RDS?](#what-is-amazon-rds)
- [What You Will Build](#what-you-will-build)
- [Key Concepts](#key-concepts)
- [Prerequisites](#prerequisites)
- [Step 1: Create the Security Groups](#step-1-create-the-security-groups)
- [Step 2: Check the DB Subnet Group](#step-2-check-the-db-subnet-group)
- [Step 3: Create the Database](#step-3-create-the-database)
- [Step 4: Connect from EC2](#step-4-connect-from-ec2)
- [Step 5: Turn On Multi-AZ](#step-5-turn-on-multi-az)
- [Step 6: Test Failover](#step-6-test-failover)
- [Step 7 (Optional): Add a Read Replica](#step-7-optional-add-a-read-replica)
- [Multi-AZ vs Read Replica](#multi-az-vs-read-replica)
- [Security Best Practices](#security-best-practices)
- [Cost Tips](#cost-tips)
- [Troubleshooting](#troubleshooting)
- [Clean Up](#clean-up)
- [Repository Structure](#repository-structure)
- [Connect with Me](#connect-with-me)

---

## What Is Amazon RDS?

Amazon RDS (Relational Database Service) is a managed service for relational databases. You choose the engine and size, and AWS handles the heavy lifting.

**Supported engines:** Amazon Aurora (MySQL and PostgreSQL compatible), MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, and Db2.

| AWS manages | You manage |
|---|---|
| Hardware provisioning and OS patching | Schema design, queries, and indexes |
| Automated backups and point-in-time recovery | Security groups, network placement, and user access |
| Database software patching (maintenance window) | Choosing instance size and storage |
| Multi-AZ failover | Cost control |
| Monitoring through CloudWatch | |

> **Note:** With standard RDS you don't get SSH or OS-level access to the underlying host. That's the trade-off for a managed service.

## What You Will Build

- Security groups where **only your EC2 app server** can reach the database
- A private MySQL RDS instance with encryption and automated backups
- A connection from EC2 to the database
- A Multi-AZ standby and a failover test
- (Optional) A read replica

**Architecture**

```
EC2 app server (ec2-app-sg) → RDS MySQL primary (rds-sg, AZ-a)
                                  ↓ synchronous replication
                              RDS standby (AZ-b), takes over on failure
```

## Key Concepts

| Term | Meaning |
|---|---|
| **DB instance** | The managed database server you pay for |
| **DB subnet group** | The subnets (in at least two AZs) where RDS may place your database |
| **Multi-AZ** | A standby copy in another AZ, kept in sync, that takes over if the primary fails. For **availability**, not read scaling |
| **Read replica** | An asynchronously updated copy used to **scale read traffic** |
| **Automated backups** | Daily backups plus transaction logs, kept 1 to 35 days, enabling point-in-time restore |
| **Snapshot** | A manual backup that stays until you delete it |
| **Endpoint** | The DNS name your app uses to connect. Always use the endpoint, never an IP |

## Prerequisites

- An AWS account with permissions for RDS, EC2, and VPC
- A VPC with subnets in at least two AZs (the default VPC works)
- An EC2 instance (Amazon Linux 2023) to act as the app server, or launch one in Step 4

> **Cost note:** RDS bills while the instance is running, and Multi-AZ roughly doubles the instance cost. Check the current Free Tier terms for your account and follow the [Clean Up](#clean-up) section.

## Step 1: Create the Security Groups

Use two security groups so the database is reachable only from your app server.

| Security group | Attached to | Inbound rule |
|---|---|---|
| `ec2-app-sg` | EC2 app server | SSH (22) from your IP, or use EC2 Instance Connect / Session Manager |
| `rds-sg` | RDS instance | **MySQL/Aurora (3306)** from source **`ec2-app-sg`** (not `0.0.0.0/0`) |

> Never open your database port to the internet.

## Step 2: Check the DB Subnet Group

1. Go to **RDS → Subnet groups**.
2. The default VPC already has a `default` subnet group covering its subnets.
3. For your own VPC, create a subnet group with subnets from **at least two AZs**.

## Step 3: Create the Database

1. Go to **RDS → Databases → Create database**.
2. Choose **Standard create** and engine **MySQL**.
3. Template: **Free tier** or **Dev/Test**, depending on what your account shows.
4. Settings:
   - DB instance identifier: `mydb-rds`
   - Master username: `admin`
   - Credentials: **Managed in AWS Secrets Manager** (recommended), or a strong password you set
5. Instance class: a small burstable class such as `db.t4g.micro` or `db.t3.micro`.
6. Storage: **gp3**, 20 GiB, with **storage autoscaling** enabled and a sensible maximum.
7. Connectivity: your VPC and subnet group, **Public access: No**, security group `rds-sg`.
8. Additional configuration:
   - Initial database name: `appdb`
   - **Automated backups:** enabled, retention 7 days
   - **Encryption:** enabled (must be chosen at creation)
   - **Deletion protection:** on for anything important
9. Click **Create database** and wait for **Available**.
10. Copy the **Endpoint** from the database details.

## Step 4: Connect from EC2

1. Connect to your EC2 instance (launch one with `ec2-app-sg` in the same VPC if needed).
2. Install a MySQL client:

```bash
   sudo dnf install -y mariadb105
```

3. Connect using the RDS endpoint (helper script included in this repo):

```bash
   RDS_ENDPOINT=<your-rds-endpoint> ./scripts/connect.sh
   # or directly:
   mysql -h <your-rds-endpoint> -u admin -p
```

   If you used Secrets Manager, open the secret in the console and choose **Retrieve secret value** for the password.

4. Load the sample data from [`sql/sample.sql`](sql/sample.sql):

```sql
   USE appdb;
   CREATE TABLE visitors (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50));
   INSERT INTO visitors (name) VALUES ('Vishal'), ('AWS Builder');
   SELECT * FROM visitors;
```

## Step 5: Turn On Multi-AZ

1. Select your database and click **Modify**.
2. Under Availability, choose **Create a standby instance (Multi-AZ)**.
3. Choose **Apply immediately** and confirm.

RDS creates a standby in another AZ and keeps it synchronized. Your endpoint stays the same.

## Step 6: Test Failover

1. Select the database, then **Actions → Reboot**.
2. Check **Reboot with failover?** and confirm.
3. Watch the **Events** tab and the database's AZ. The standby is promoted and the AZ changes.
4. Re-run your SQL query. Your data is still there and the **same endpoint** works.

Expect a short interruption (typically a minute or two) while DNS flips to the new primary. Your application should retry failed connections.

## Step 7 (Optional): Add a Read Replica

1. Select the database and choose **Actions → Create read replica**.
2. Give it an identifier such as `mydb-rds-replica`.
3. Once available, point **read-heavy queries** at the replica's endpoint and keep writes on the primary.

Replication is asynchronous, so replicas can lag slightly behind the primary.

## Multi-AZ vs Read Replica

| | Multi-AZ | Read Replica |
|---|---|---|
| Purpose | High availability and failover | Read scaling |
| Replication | Synchronous | Asynchronous |
| Takes traffic normally | No (standby only) | Yes (reads) |
| Automatic failover | Yes | No (manual promotion) |

## Security Best Practices

- [x] Keep **Public access set to No** and place the DB in private subnets
- [x] Allow access only from application security groups
- [x] Use **Secrets Manager** for credentials and rotate them
- [x] Enable **encryption at rest** (KMS) and enforce **SSL/TLS** connections
- [x] Turn on **deletion protection** and automated backups
- [x] Use **IAM** to control who can modify or delete DB instances

## Cost Tips

- Pick the smallest instance that meets your needs and resize later.
- Use Multi-AZ for production, but not for throwaway labs.
- Stop or delete dev databases you aren't using. A stopped RDS instance is automatically started again after 7 days.
- Delete old manual snapshots you no longer need.
- Consider Reserved Instances for steady, long-running databases.

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| Connection times out | `rds-sg` doesn't allow 3306 from your app's SG | Add the inbound rule with `ec2-app-sg` as source |
| Can't connect from your laptop | Database is private (public access: No) | Connect through EC2, a bastion, or Session Manager |
| Access denied for user | Wrong username or password | Re-check credentials or the Secrets Manager value |
| Unknown database | Initial DB name wasn't set | Run `CREATE DATABASE appdb;` |
| Storage full warnings | Autoscaling off or limit reached | Enable storage autoscaling or increase size |

## Clean Up

To avoid charges:

1. Delete the **read replica** first (if you created one)
2. Turn off **deletion protection**, then delete the **DB instance**
3. Decide whether to keep a **final snapshot** (it costs storage) and whether to delete **automated backups**
4. Delete leftover **manual snapshots**
5. Delete the **security groups** and the **EC2 instance** if no longer needed
6. Delete the **Secrets Manager secret** if you created one just for this lab

## Repository Structure

```
aws-rds-mysql-guide/
├── README.md              # This guide
├── assets/
│   └── rds-cover.png      # Cover image
├── sql/
│   └── sample.sql         # Sample table and data for the lab
├── scripts/
│   └── connect.sh         # Helper to connect to RDS from EC2
└── .gitignore
```

## Connect with Me

[![LinkedIn](https://img.shields.io/badge/LinkedIn-vishalpatil--devops--aws-0A66C2?logo=linkedin&logoColor=white)](https://www.linkedin.com/in/vishalpatil-devops-aws/)
[![dev.to](https://img.shields.io/badge/dev.to-vishal__patil__aws-0A0A0A?logo=devdotto&logoColor=white)](https://dev.to/vishal_patil_aws)
[![GitHub](https://img.shields.io/badge/GitHub-VishalPatil--DevOps--AWS-181717?logo=github&logoColor=white)](https://github.com/VishalPatil-DevOps-AWS)

If this guide helped you, give the repo a ⭐ and let's keep learning together.

**By Vishal Patil** | #vishalcloudjourney
