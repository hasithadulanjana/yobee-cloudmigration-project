# Task 4: Optimization and Security
**Yoobee College — AWS Cloud Migration Project**
**Date:** March 2026

---

## 1. Introduction

This report documents the security configurations and cost optimization strategies implemented as part of the Yoobee College AWS cloud migration project. The infrastructure is deployed across seven CloudFormation stacks in the `ap-southeast-2` (Sydney) region, hosting the Learning Management System (LMS) and Faculty Application on AWS.

The migration moves Yoobee's on-premises systems to a cloud architecture that follows AWS Well-Architected Framework principles, with emphasis on the Security and Cost Optimization pillars. This report covers the specific AWS security best practices applied, the CloudWatch monitoring configuration, a cost analysis informed by AWS Trusted Advisor checks, and recommendations for hardening the environment prior to production use.

---

## 2. AWS Security Best Practices Implemented

### 2.1 Network Security

The infrastructure is built on a custom VPC with a strict network segmentation model. EC2 instances and the RDS database are placed in **private subnets** with no direct internet exposure. Only the Application Load Balancer (ALB) resides in public subnets and accepts inbound traffic from the internet.

**Security Groups configured:**

| Security Group | Inbound Rules | Purpose |
|---|---|---|
| ALBSecurityGroup | TCP 80 from 0.0.0.0/0; TCP 443 from 0.0.0.0/0 | Accepts HTTP/HTTPS from public internet |
| LMSSecurityGroup | TCP 80/443 from ALBSecurityGroup; TCP 22 from AllowedAdminCIDR | LMS EC2 instances — web traffic from ALB only; SSH restricted to admin IP |
| FacultyAppSecurityGroup | TCP 80/443 from ALBSecurityGroup; TCP 3389 from AllowedAdminCIDR | Faculty EC2 instances — web traffic from ALB only; RDP restricted to admin IP |
| RDSSecurityGroup | TCP 3306 from LMSSecurityGroup; TCP 3306 from FacultyAppSecurityGroup | MySQL access from application tier only — no external database access |

**Network ACLs** are applied to private subnets as an additional perimeter defense layer. They explicitly allow inbound TCP 80, 443, and ephemeral ports (1024–65535) from within the VPC CIDR, and deny all other inbound traffic by default. This provides defense-in-depth: even if a security group rule were misconfigured, the NACL provides a secondary enforcement point.

No EC2 instance or RDS endpoint is reachable directly from the internet. All internet-bound traffic from private subnets routes through NAT Gateways, which are outbound-only by design.

### 2.2 Identity and Access Management

The principle of least privilege is applied to all IAM roles and groups. Each role grants only the permissions required for its specific workload, and no role carries administrator-level access.

**IAM Roles and Groups:**

| Name | Type | Permissions | Justification |
|---|---|---|---|
| LMSAppRole | IAM Role | S3 read/write to LMS bucket; SSM managed instance; CloudWatch logs | LMS EC2 instances need to serve/store course assets and emit logs |
| FacultyAppRole | IAM Role | S3 read/write to Faculty bucket; SSM managed instance; CloudWatch logs | Faculty EC2 instances need the same scoped access for their own data |
| DBAccessRole | IAM Role | RDS data access; Secrets Manager read | Provides database credentials access without embedding secrets in code |
| StudentCognitoRole | IAM Role | Cognito identity pool authenticated role; S3 read for course materials | Students authenticate via Cognito and read course content only |
| CloudAdminGroup | IAM Group | PowerUser or AdministratorAccess | Cloud engineers managing infrastructure |
| ITAdminGroup | IAM Group | Full access subject to MFA enforcement policy | IT administrators — require MFA before any AWS action is permitted |
| ReadOnlyOpsGroup | IAM Group | ReadOnlyAccess managed policy | Monitoring/operations staff — view metrics and logs without making changes |

**SSM Session Manager** is used on all EC2 instances instead of opening SSH (port 22) or RDP (port 3389) to the internet. The `AmazonSSMManagedInstanceCore` managed policy is attached to both `LMSAppRole` and `FacultyAppRole`. This approach is more secure because:

- No inbound port needs to be open on security groups for administrative access.
- Session activity is logged to CloudWatch and S3, providing a full audit trail.
- Access is controlled by IAM policies rather than SSH key management, reducing the risk of key theft or mismanagement.
- MFA can be enforced at the IAM level for session start.

Note: While SSH (port 22) and RDP (port 3389) are still defined in the LMS and Faculty security groups restricted to `AllowedAdminCIDR`, these are a fallback and should be removed or further restricted in a production hardening exercise.

### 2.3 Data Security

**S3 Security:**
- All buckets use **AES-256 SSE-S3 encryption** at rest, applied automatically to every object on upload.
- All four public access block settings are enabled on every bucket (`BlockPublicAcls`, `IgnorePublicAcls`, `BlockPublicPolicy`, `RestrictPublicBuckets`), preventing any accidental public exposure.
- A **DenyNonSSL** bucket policy denies any S3 request that does not use HTTPS (`aws:SecureTransport: false`), ensuring data in transit is always encrypted.
- **S3 versioning** is enabled to protect against accidental deletion or overwrite of course materials.
- **Access logging** is enabled on all buckets, with logs stored in a dedicated logging bucket for audit purposes.

**EBS and Compute:**
- All EBS volumes are encrypted using the default AWS-managed KMS key (`gp3` type, 50 GB for Linux LMS instances and 80 GB for Windows Faculty instances).
- No credentials, passwords, or API keys are stored in CloudFormation templates or EC2 UserData. Database credentials are managed via GitHub Actions Secrets and passed as CloudFormation parameters at deployment time.

**RDS Security:**
- The RDS MySQL instance is deployed in a **private subnet group** with no public accessibility.
- Access is exclusively through `RDSSecurityGroup`, which only permits MySQL connections from the application-tier security groups.
- RDS **error, general, and slowquery logs** are exported to CloudWatch Logs for audit and performance monitoring.

### 2.4 MFA Setup

The **EnforceMFA** IAM policy is attached to `ITAdminGroup` and uses a conditional Deny to enforce multi-factor authentication before any AWS action can be taken. The policy logic is:

- **Deny all actions** (`*`) when the condition `aws:MultiFactorAuthPresent` is `false`, **except** for the following IAM actions required to set up MFA itself:
  - `iam:CreateVirtualMFADevice`
  - `iam:EnableMFADevice`
  - `iam:GetUser`
  - `iam:ListMFADevices`
  - `sts:GetSessionToken`

This means an IT admin who logs in without MFA can only enroll their MFA device — they cannot perform any other AWS action until they authenticate with MFA. Once MFA is present in the session, all permissions granted to the group become active.

**Root account:** Enabling MFA on the AWS root account should be the first manual action after account setup. The root account bypasses all IAM policies and should never be used for day-to-day operations. AWS recommends locking the root account with a hardware MFA device.

### 2.5 Screenshots to Take for Security

The following screenshots should be captured to evidence the security configuration:

1. **IAM > Roles** — Show the roles list with `LMSAppRole`, `FacultyAppRole`, `DBAccessRole`, and `StudentCognitoRole` visible.
2. **IAM > Roles > LMSAppRole** — Show the Trust Relationship tab (trusted entity: EC2) and the Permissions tab (S3, SSM, CloudWatch policies).
3. **IAM > User groups** — Show the groups list with `CloudAdminGroup`, `ITAdminGroup`, `ReadOnlyOpsGroup` visible.
4. **IAM > User groups > ITAdminGroup > Permissions** — Show the `EnforceMFA` inline or managed policy attached to the group.
5. **VPC > Security Groups** — Show the list of all security groups with their names (ALBSecurityGroup, LMSSecurityGroup, FacultyAppSecurityGroup, RDSSecurityGroup).
6. **VPC > Security Groups > RDSSecurityGroup > Inbound rules** — Show that only MySQL/Aurora port 3306 is allowed, sourced from LMSSecurityGroup and FacultyAppSecurityGroup.
7. **S3 > yobee-lms-assets > Permissions > Block public access** — Show all four checkboxes enabled.
8. **S3 > yobee-lms-assets > Permissions > Bucket policy** — Show the `DenyNonSSL` policy statement.

---

## 3. AWS CloudWatch Monitoring Configuration

### 3.1 Monitoring Architecture

CloudWatch serves as the central observability platform for the Yoobee infrastructure. Metrics are collected from four AWS services: EC2, RDS, ALB, and Auto Scaling. The CloudWatch Agent is installed on EC2 instances via the instance `UserData` script at launch time, enabling collection of OS-level metrics (memory, disk utilisation) that are not available from the default EC2 hypervisor metrics.

All alarms are connected to a single **SNS topic** (`YoobeeAlarmsTopic`) which delivers email notifications to the administration team. When any alarm transitions to the `ALARM` state, an email is sent immediately, enabling rapid incident response.

### 3.2 Alarms Configured

| Alarm Name | Metric | Threshold | Period | Action |
|---|---|---|---|---|
| LMS1CPUAlarm | EC2 CPUUtilization (LMS instance 1) | > 80% | 2 × 5 min | SNS email notification |
| LMS2CPUAlarm | EC2 CPUUtilization (LMS instance 2) | > 80% | 2 × 5 min | SNS email notification |
| LMS1StatusCheckAlarm | EC2 StatusCheckFailed (LMS instance 1) | >= 1 failure | 2 × 1 min | SNS email notification |
| Faculty1CPUAlarm | EC2 CPUUtilization (Faculty instance 1) | > 80% | 2 × 5 min | SNS email notification |
| Faculty2CPUAlarm | EC2 CPUUtilization (Faculty instance 2) | > 80% | 2 × 5 min | SNS email notification |
| RDSConnectionAlarm | RDS DatabaseConnections | > 80 connections | 1 × 5 min | SNS email notification |
| RDSStorageAlarm | RDS FreeStorageSpace | < 5 GB | 1 × 5 min | SNS email notification |
| ALB5xxAlarm | ALB HTTPCode_Target_5XX_Count | > 10 errors | 1 × 5 min | SNS email notification |

The two-period requirement on CPU alarms prevents transient spikes from generating false positive alerts — the CPU must be above threshold for at least 10 consecutive minutes before an alert fires.

### 3.3 CloudWatch Dashboard

The **YobeeMigration-Infrastructure** dashboard provides a single-pane view of infrastructure health with four widgets:

1. **EC2 CPU Utilisation** — Line graph showing CPUUtilization for all four EC2 instances (LMS1, LMS2, Faculty1, Faculty2).
2. **RDS Metrics** — Combined view of `DatabaseConnections` and `FreeStorageSpace` for the RDS instance.
3. **ALB Requests and Errors** — Shows `RequestCount`, `HTTPCode_Target_5XX_Count`, and `HTTPCode_Target_4XX_Count` from the load balancer.
4. **ASG Instance Count** — Tracks `GroupInServiceInstances` from the LMS Auto Scaling Group, showing how the group scales in response to load.

### 3.4 Screenshots to Take for CloudWatch

1. **CloudWatch > Alarms > All alarms** — Show the full list of 8 alarms, ideally in `OK` or `INSUFFICIENT_DATA` state, confirming all alarms are created.
2. **CloudWatch > Alarms > [Any alarm]** — Click into one alarm (e.g., `LMS1CPUAlarm`) and show the details panel: metric name, threshold value, evaluation period, and the configured SNS action.
3. **CloudWatch > Dashboards > YobeeMigration-Infrastructure** — Show the dashboard with all four widgets rendered.
4. **CloudWatch > Log groups** — Show `/yoobee/lms` and `/yoobee/faculty` log groups present (requires EC2 instances to be running with CloudWatch Agent active).

---

## 4. Cost Optimization

### 4.1 Cost Optimization Strategies Implemented

Several cost controls are built directly into the CloudFormation architecture:

- **t3.micro instance type** for all EC2 instances — eligible for 750 free hours per month under the AWS Free Tier for the first 12 months of account use.
- **RDS db.t3.micro** — eligible for 750 free hours per month under Free Tier, covering a single-AZ instance running continuously.
- **Auto Scaling with minimum capacity of 0** on the LMS Auto Scaling Group — allows the group to scale to zero instances during off-hours (nights and weekends), eliminating EC2 compute costs when the LMS is not in use.
- **S3 Lifecycle policies** — non-current object versions are transitioned to `STANDARD_IA` storage class after 30 days and expired after 90 days, reducing long-term storage costs for versioned course materials.
- **Single-AZ RDS deployment** — Multi-AZ is disabled for this academic environment, avoiding the 2× cost of a standby replica. Multi-AZ should be re-evaluated before production launch.
- **gp3 EBS volumes** — gp3 provides 3,000 IOPS baseline at no extra cost and is approximately 20% cheaper per GB than gp2.

### 4.2 AWS Trusted Advisor Analysis (Theoretical)

The AWS account uses Free Tier support, which limits Trusted Advisor access to six core checks. Full recommendations require Business or Enterprise Support plans. The following analysis is based on the known Trusted Advisor check categories and applied to the deployed architecture:

**Cost Optimization checks:**

- **Idle EC2 Instances:** Auto Scaling to zero is already implemented for the LMS tier, directly addressing this concern for off-hours periods.
- **Underutilized EBS Volumes:** gp3 volumes are already used, which is the most cost-effective general-purpose option. No provisioned IOPS are configured, avoiding unnecessary IOPS charges.
- **Reserved Instance Recommendations:** After the 12-month Free Tier period, purchasing 1-year Reserved Instances for the always-on Faculty application servers could yield approximately 40% savings compared to On-Demand pricing.
- **NAT Gateway Optimisation:** The two NAT Gateways represent the largest single ongoing cost (~$86/month combined). Deploying **VPC Gateway Endpoints for S3** (free of charge) would eliminate all NAT Gateway data processing charges for S3 API calls from private subnets, which is a significant portion of traffic in an LMS environment.

**Security checks (Free Tier — 6 core checks):**

| Trusted Advisor Check | Status | Notes |
|---|---|---|
| S3 Bucket Permissions | PASS | All public access blocked on all buckets |
| Security Groups — Unrestricted Access | REVIEW | `AllowedAdminCIDR` should be narrowed from 0.0.0.0/0 to the college's actual IP range |
| IAM Use | PASS | IAM roles and groups are configured; root account not used for operations |
| MFA on Root Account | ACTION REQUIRED | Must be enabled manually — not automatable via CloudFormation |
| EBS Public Snapshots | PASS | No snapshots are set to public |
| RDS Public Snapshots | PASS | RDS is private and no public snapshots exist |

**Performance checks (Business Support required):**

- EBS Provisioned IOPS utilisation — worth reviewing after sustained load testing.
- CloudFront CDN for course materials — recommended as a future enhancement to reduce ALB load and improve student download speeds globally.

**Fault Tolerance checks (Business Support required):**

- Multi-AZ deployment is currently disabled. For production, enabling Multi-AZ on the RDS instance is strongly recommended.
- S3 versioning: PASS — enabled on all buckets.

### 4.3 Cost Reduction Recommendations

1. **VPC Gateway Endpoint for S3** — Deploy a free S3 Gateway Endpoint to route all S3 traffic from private subnets directly, bypassing the NAT Gateway. This can eliminate a significant portion of the ~$86/month NAT Gateway data charge.
2. **Reserved Instances** — After the Free Tier period expires, purchase 1-year Standard Reserved Instances for the Faculty App servers (which run continuously), achieving approximately 40% savings.
3. **Auto Scaling schedule** — Configure a scheduled scaling action to set the LMS ASG minimum to 0 after 10 PM and back to 1 at 7 AM on weekdays, and to 0 all weekend. This directly eliminates EC2 costs during unused hours.
4. **S3 Intelligent-Tiering** — For course material objects with unpredictable access patterns (archived courses vs. current semester), S3 Intelligent-Tiering automatically moves objects between access tiers, potentially reducing storage costs by 30–50% on infrequently accessed content.
5. **AWS Billing Alarms** — Set up a CloudWatch billing alarm at a $50/month threshold to detect unexpected cost increases early.
6. **Right-sizing review** — After 4 weeks of production load, review CPU and memory utilisation in CloudWatch. If average utilisation is consistently below 20%, the workload may be a candidate for downsizing or consolidation.

### 4.4 Estimated Monthly Cost Breakdown (ap-southeast-2)

| Service | Configuration | Est. Monthly Cost (Free Tier) | Est. Monthly Cost (Post Free Tier) |
|---|---|---|---|
| EC2 (4 × t3.micro) | 730 hrs/month each | ~$0.00 | ~$30.16 |
| RDS (db.t3.micro, Single-AZ) | 730 hrs/month | ~$0.00 | ~$14.60 |
| Application Load Balancer | 1 ALB, low traffic | ~$16.43 | ~$16.43 |
| NAT Gateways (×2) | ~$0.059/hr each + data | ~$86.12 | ~$86.12 |
| S3 Storage | <5 GB, standard class | ~$0.58 | ~$0.58 |
| EBS Volumes (4 × gp3) | 50 GB + 80 GB each type | ~$0.00 (Free Tier 30 GB) | ~$14.40 |
| CloudWatch | 8 alarms, 1 dashboard | ~$3.20 | ~$3.20 |
| **Total (approximate)** | | **~$106/month** | **~$165/month** |

> **Note:** NAT Gateways are the dominant ongoing cost driver. Deploying a VPC S3 Gateway Endpoint (free) and reviewing whether both NAT Gateways are required could reduce monthly costs by $40–$60.

---

## 5. Security Recommendations for Production

Before promoting the Yoobee infrastructure to a production environment serving live students, the following additional controls should be implemented:

1. **Restrict `AllowedAdminCIDR`** — Replace the current `0.0.0.0/0` default with the Yoobee College office IP range. This limits SSH/RDP access to known physical locations even as a secondary control behind SSM.
2. **Enable MFA on the AWS root account** — This is the single highest-priority manual action. Use a hardware MFA token (YubiKey) rather than a virtual authenticator for maximum security.
3. **Enable AWS GuardDuty** — GuardDuty provides continuous threat detection by analysing VPC Flow Logs, DNS logs, and CloudTrail events for anomalous behaviour such as port scanning, unusual API calls, or communication with known malicious IP addresses.
4. **Enable AWS Config** — Config continuously records the state of AWS resources and evaluates them against compliance rules (e.g., ensuring all S3 buckets have encryption enabled, all security groups are compliant). This is essential for ongoing governance.
5. **Enable AWS CloudTrail** — CloudTrail records every API call made to the AWS account, providing a complete audit trail for security investigations and compliance reporting. Logs should be stored in a dedicated, write-protected S3 bucket.
6. **Deploy AWS WAF** — Attaching AWS WAF to the ALB adds a web application firewall that can block common attack patterns (SQL injection, XSS, OWASP Top 10) before they reach the application tier.
7. **Enable RDS Multi-AZ** — For a production LMS serving assessed student coursework, database high availability is critical. Multi-AZ provides automatic failover with typically less than 60 seconds of downtime during a failure event.
8. **Rotate IAM access keys** — Any long-term IAM access keys (used by GitHub Actions or other CI/CD tools) should be rotated every 90 days and monitored via IAM credential reports.

---

## 6. Conclusion

The Yoobee College AWS migration has been architected with a defence-in-depth security posture, applying multiple layers of control: network segmentation via VPC and private subnets, access control via security groups and NACLs, identity management via scoped IAM roles and MFA-enforced groups, and data protection via encryption at rest and in transit.

CloudWatch monitoring provides operational visibility across EC2, RDS, and ALB with automated alerting through SNS, ensuring the operations team is notified promptly of performance degradation or resource exhaustion.

Cost optimisation is addressed through Free Tier–eligible instance sizing, Auto Scaling to zero for non-production hours, S3 lifecycle policies, and the strategic use of gp3 storage. The primary cost driver post-Free Tier is the NAT Gateway pair, which can be substantially reduced through the implementation of VPC Gateway Endpoints for S3.

The recommendations in Section 5 outline the path from this functional academic deployment to a hardened, production-grade environment aligned with AWS security best practices and the AWS Well-Architected Framework.
