# Task 3: Practical Implementation — Virtual Machines and Storage Services

**Assessment:** Yoobee College — AWS Cloud Migration Project
**Student:** Hasitha
**Region:** ap-southeast-2 (Sydney)
**Deployment Method:** AWS CloudFormation via GitHub Actions CI/CD Pipeline

---

## 1. Introduction

This document details the practical implementation of cloud infrastructure for the Yoobee College Learning Management System (LMS) migration to AWS. The solution provisions virtual machine instances, object storage, and load balancing services to support the LMS and Faculty portal workloads.

All infrastructure was defined as code using AWS CloudFormation templates and deployed automatically through a GitHub Actions pipeline. This Infrastructure-as-Code (IaC) approach ensures reproducibility, version control, and consistent deployment across environments. The compute stack provisions four EC2 instances across two availability zones, while the storage stack configures a secure, versioned S3 bucket. An Application Load Balancer routes traffic between the LMS and Faculty server fleets, and an Auto Scaling Group provides elastic capacity for the LMS tier.

---

## 2. EC2 Instance Configuration

### 2.1 Linux EC2 Instances (LMS Servers)

Two Linux instances — **LMSServer1** and **LMSServer2** — serve the Yoobee LMS application.

| Parameter | Value |
|---|---|
| Instance type | t3.micro |
| AMI | Amazon Linux 2 (latest, ap-southeast-2) |
| Region | ap-southeast-2 (Sydney) |
| Placement | Private subnets — one instance per AZ (ap-southeast-2a, ap-southeast-2b) |
| EBS root volume | 50 GB, gp3, AES-256 encrypted |
| IAM instance profile | LMSAppRole — grants SSM Session Manager, CloudWatch Logs/Metrics, and S3 read/write access |
| Key pair | Configured for emergency break-glass SSH access |

**Security group rules (LMSSecurityGroup):**
- Inbound port 80 (HTTP) — source: ALBSecurityGroup only
- Inbound port 443 (HTTPS) — source: ALBSecurityGroup only
- Inbound port 22 (SSH) — source: admin CIDR block only
- Outbound: all traffic permitted

**UserData bootstrap script** runs on first launch and performs the following:
1. Updates all system packages via `yum update -y`
2. Installs `httpd` (Apache) and `php` for the LMS application runtime
3. Installs and configures the Amazon CloudWatch agent to ship system and application logs to CloudWatch Logs
4. Enables and starts the `httpd` service

### 2.2 Windows EC2 Instances (Faculty Servers)

Two Windows instances — **FacultyServer1** and **FacultyServer2** — host the Yoobee Faculty portal via IIS.

| Parameter | Value |
|---|---|
| Instance type | t3.micro |
| AMI | Windows Server 2022 Base (latest, ap-southeast-2) |
| Region | ap-southeast-2 (Sydney) |
| Placement | Private subnets — one instance per AZ (ap-southeast-2a, ap-southeast-2b) |
| EBS root volume | 80 GB, gp3, AES-256 encrypted |
| IAM instance profile | FacultyAppRole — grants SSM Session Manager, CloudWatch Logs/Metrics, and S3 read-only access |

**Security group rules (FacultySecurityGroup):**
- Inbound port 80 (HTTP) — source: ALBSecurityGroup only
- Inbound port 443 (HTTPS) — source: ALBSecurityGroup only
- Inbound port 3389 (RDP) — source: admin CIDR block only
- Outbound: all traffic permitted

**UserData bootstrap script** (PowerShell) runs via EC2Launch on first boot and performs the following:
1. Installs the `Web-Server` Windows feature (IIS)
2. Installs `Web-Asp-Net45` for ASP.NET 4.5 application support
3. Installs the Amazon CloudWatch agent MSI and configures it to forward Windows Event Logs and IIS logs to CloudWatch Logs

### 2.3 Screenshots to Take — EC2 Instances

Follow these steps in the **AWS Console → EC2** service:

1. Navigate to **EC2 → Instances**. Take a screenshot showing all four instances — LMSServer1, LMSServer2, FacultyServer1, FacultyServer2 — with their **Name**, **Instance type** (t3.micro), and **Instance state** (Running) visible in the table.

2. Click **LMSServer1**. On the **Details** tab, take a screenshot showing the Instance ID, instance type, VPC ID, Subnet ID, and Security groups fields.

3. Click **LMSServer2**. Repeat the same screenshot of the Details tab.

4. Click **FacultyServer1**. Take a screenshot of the Details tab showing Instance ID, instance type, VPC, subnet, and security groups.

5. Click **FacultyServer2**. Repeat the same screenshot of the Details tab.

6. Return to the Instances list. Select **LMSServer1**, then click the **Security** tab. Take a screenshot showing the inbound security group rules — ports 80 and 22 with their source CIDRs or security group references.

7. With **LMSServer1** selected, click the **Storage** tab. Take a screenshot showing the root EBS volume — confirm it shows **50 GiB**, **gp3**, and that encryption is **Enabled**.

---

## 3. S3 Bucket Configuration

### 3.1 Bucket Setup

| Parameter | Value |
|---|---|
| Bucket name | yobee-lms-assets-{AWS Account ID} |
| Region | ap-southeast-2 (Sydney) |
| Versioning | Enabled |
| Access logging | Enabled — logs delivered to `yobee-access-logs` bucket |

### 3.2 Security Policies Applied

**Block Public Access:** All four Block Public Access settings are enabled, preventing any public ACL or bucket policy from exposing objects.

**Default Encryption:** Server-Side Encryption with Amazon S3-managed keys (SSE-S3 / AES-256) is set as the default. All objects written to the bucket are automatically encrypted at rest.

**Bucket Policy — DenyNonSSL:** A bucket policy denies all S3 actions where the request condition `aws:SecureTransport` is `false`. This ensures all data in transit is encrypted via HTTPS/TLS and that unencrypted HTTP requests to the bucket are rejected.

**CORS Configuration:** A CORS rule permits `GET` requests originating from `https://*.yoobee.ac.nz`, allowing the LMS front-end hosted on the Yoobee domain to retrieve assets directly from S3.

### 3.3 Lifecycle Rules

A lifecycle rule named `VersionManagement` manages object versions to control storage costs:

| Action | Condition |
|---|---|
| Transition noncurrent versions to STANDARD_IA | After 90 days |
| Permanently expire (delete) noncurrent versions | After 365 days |

This retains recent versions in Standard storage for recovery purposes while moving older versions to infrequent-access storage, and ultimately deleting versions that are over a year old.

### 3.4 Screenshots to Take — S3

Follow these steps in the **AWS Console → S3** service:

1. Navigate to **S3 → Buckets**. Take a screenshot of the bucket list showing both **yobee-lms-assets-{accountid}** and **yobee-access-logs** buckets.

2. Click the **yobee-lms-assets** bucket. On the **Properties** tab, scroll to the **Bucket Versioning** section. Take a screenshot confirming the status is **Enabled**.

3. Still on the **Properties** tab, scroll to **Default encryption**. Take a screenshot showing encryption type **SSE-S3 (AES-256)**.

4. Click the **Permissions** tab. Take a screenshot of the **Block public access** section showing all four settings ticked/enabled.

5. Still on the **Permissions** tab, scroll to **Bucket policy**. Take a screenshot of the JSON policy showing the `DenyNonSSL` statement with the `aws:SecureTransport: false` condition.

6. Click the **Management** tab. Take a screenshot of the **Lifecycle rules** section showing the `VersionManagement` rule with its transition and expiration actions.

---

## 4. Elastic Load Balancer Setup

### 4.1 ALB Configuration

| Parameter | Value |
|---|---|
| Name | YobeeMigration-ALB |
| Type | Application Load Balancer |
| Scheme | Internet-facing |
| Subnets | PublicSubnet1 (ap-southeast-2a), PublicSubnet2 (ap-southeast-2b) |
| Security group | ALBSecurityGroup — inbound ports 80 and 443 from 0.0.0.0/0 |
| Access logs | Enabled — delivered to `yobee-access-logs` S3 bucket |

### 4.2 Target Groups

**LMS Target Group (LMS-TG):**
- Protocol: HTTP, Port: 80
- Health check path: `/health`
- Registered targets: LMSServer1, LMSServer2

**Faculty Target Group (Faculty-TG):**
- Protocol: HTTP, Port: 80
- Health check path: `/`
- Registered targets: FacultyServer1, FacultyServer2

### 4.3 Listener Rules

The ALB has a single HTTP listener on port 80 with two forwarding rules:

| Priority | Condition | Action |
|---|---|---|
| 10 | Path pattern: `/faculty/*` | Forward to Faculty-TG |
| Default | (all other requests) | Forward to LMS-TG |

This routing configuration ensures that requests to `/faculty/` paths are directed to the Windows/IIS Faculty servers, while all other traffic is handled by the Linux/Apache LMS servers.

### 4.4 Auto Scaling Integration

An Auto Scaling Group — **LMSAutoScalingGroup** — is associated with the LMS Target Group. As the ASG launches new instances, they are automatically registered with the target group and begin receiving traffic once health checks pass.

| Parameter | Value |
|---|---|
| Minimum capacity | 0 instances |
| Maximum capacity | 6 instances |
| Scaling policy | Target tracking — CPU utilisation target: 70% |

When average CPU utilisation across the LMS fleet exceeds 70%, the ASG launches additional instances. When load drops, it terminates surplus instances to reduce cost.

### 4.5 Screenshots to Take — Load Balancer

Follow these steps in the **AWS Console → EC2** service:

1. Navigate to **EC2 → Load Balancers**. Take a screenshot showing **YobeeMigration-ALB** with **State: Active** and its public DNS name visible.

2. Click **YobeeMigration-ALB**. On the **Description** tab, take a screenshot showing the scheme (internet-facing), availability zones, and security groups.

3. Navigate to **EC2 → Target Groups**. Take a screenshot of the target group list showing both **LMS-TG** and **Faculty-TG**.

4. Click **LMS-TG**, then select the **Targets** tab. Take a screenshot showing LMSServer1 and LMSServer2 registered as targets with their health check status.

5. Return to **EC2 → Load Balancers**, click the ALB, and select the **Listeners** tab. Take a screenshot showing the port 80 HTTP listener.

6. Click **View/edit rules** for the port 80 listener. Take a screenshot showing both routing rules — the `/faculty/*` path rule forwarding to Faculty-TG, and the default rule forwarding to LMS-TG.

7. Navigate to **EC2 → Auto Scaling Groups**. Take a screenshot of **LMSAutoScalingGroup** showing the Min (0), Max (6), and Desired capacity values, and the associated target group.

---

## 5. Deployment Pipeline

### 5.1 GitHub Actions Workflow

All infrastructure is deployed through a GitHub Actions CI/CD pipeline defined in the repository. The workflow is triggered on every push to the `main` branch or manually via workflow dispatch.

**Deployment sequence** (stacks deployed in dependency order):

| Step | Stack | Purpose |
|---|---|---|
| 1 | network | VPC, subnets, route tables, internet gateway |
| 2 | storage | S3 buckets and policies |
| 3 | IAM | Roles and instance profiles |
| 4 | database | RDS or other data tier resources |
| 5 | compute | EC2 instances (all four servers) |
| 6 | ALB | Load balancer, target groups, listener rules |
| 7 | monitoring | CloudWatch dashboards and alarms |

A separate **delete job** tears down all stacks in reverse order when triggered, ensuring resources with dependencies are removed cleanly.

Before deployment, the pipeline runs `aws cloudformation validate-template` against all seven templates to catch syntax and structural errors before any resources are created.

### 5.2 Screenshots to Take — Deployment Pipeline

1. Open the **GitHub repository → Actions** tab. Take a screenshot of the most recent successful workflow run, showing all deployment steps with green checkmarks.

2. Navigate to **AWS Console → CloudFormation → Stacks**. Take a screenshot showing all seven stacks with a **CREATE_COMPLETE** (or **UPDATE_COMPLETE**) status.

---

## 6. Conclusion

This implementation successfully provisions the core compute and storage infrastructure required for Yoobee College's LMS migration to AWS. Four EC2 instances — two Linux (Apache/PHP) and two Windows (IIS) — are deployed into private subnets across two availability zones, providing fault tolerance and workload separation between the LMS and Faculty portal. The S3 bucket is secured with encryption at rest, enforced HTTPS, versioning for data recovery, and automated lifecycle management to control costs.

The Application Load Balancer provides a single internet-facing entry point, routing traffic intelligently between the two server fleets based on URL path, while the Auto Scaling Group ensures the LMS tier can scale elastically between zero and six instances in response to CPU demand. All infrastructure is defined in CloudFormation templates and deployed through an automated GitHub Actions pipeline, meeting Yoobee's requirements for repeatable, auditable, and version-controlled cloud infrastructure.
