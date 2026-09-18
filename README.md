# PMBRS-CloudOps-Lab

A deliberate, hands-on lab built to close specific, identified gaps toward the AWS Certified CloudOps Engineer – Associate (SOA-C03) certification — reusing and rebuilding on the existing PMBRS application and infrastructure (see [`PMBRS-MES-LAB`](https://github.com/AbdulKhatib/PMBRS-MES-LAB)) rather than starting from an unrelated toy project.

Where PMBRS proved end-to-end deployment engineering on a single, simple AWS setup (default VPC, one app server, one database server, manual security-group management), this lab exists to rebuild that same system on real operational AWS patterns: custom networking, monitoring, high availability, automation, and security — the specific domains SOA-C03 tests and PMBRS did not yet cover.

## Why this exists

A gap analysis against the SOA-C03 exam domains, done honestly against what PMBRS actually built, found:

- **Strong:** reliability/recovery discipline (real EBS snapshot/restore test), deployment automation (real CI/CD pipeline with rollback)
- **Weak or absent:** IAM beyond a single broad-access account, CloudWatch/CloudTrail, custom VPC design, NAT/VPC endpoints, Route 53, Auto Scaling/load balancing, Infrastructure as Code

This lab is structured to close each gap in the order the exam weights them, using PMBRS as the running example throughout rather than disconnected mini-exercises.

## How this lab is different from PMBRS

PMBRS optimized for shipping a working system and documenting what broke. This lab optimizes for exam-relevant depth and diagnostic fluency — the goal isn't a portfolio narrative, it's being able to walk into a scenario question (or a real production incident) and correctly reason: which layer blocked this, and why.

## Relationship to PMBRS

This is not a new application. The PMBRS three-tier app (Windows Server, IIS, Flask, Oracle) is the constant — what changes around it is the AWS infrastructure it runs on and the operational practices applied to it. Where PMBRS used the default VPC, a single instance per tier, manual security groups, and no AWS-native monitoring, this lab rebuilds those specific layers to be exam-representative, while leaving the application layer itself alone.

Some PMBRS instances or their replacements may be terminated and rebuilt during Stage 2 and Stage 5 (VPC migration, CloudFormation rebuild) — this is expected and is itself part of the learning (recovery from real infrastructure change, not just simulated failure).

## In Scope

- Custom VPC design: public/private subnets, route tables, internet gateway, NAT gateway
- Security Group, NACL, Windows Firewall, and route table failure isolation (deliberately breaking each layer independently)
- VPC Endpoints (S3 via PrivateLink, at minimum)
- Route 53: hosted zones, record types, routing policies, DNS troubleshooting
- CloudWatch: custom metrics (memory, via the CloudWatch agent), alarms, SNS notification
- CloudTrail: enabled, read, understood
- VPC Flow Logs and ELB access logs: enabled and interpreted during troubleshooting scenarios
- Auto Scaling Group + Application Load Balancer in front of the app tier (PMBRS moves from single-instance to APP-A/APP-B behind a load balancer)
- A second, full instance-loss disaster recovery test (PMBRS's original DR test only proved volume-level recovery; this closes that explicitly acknowledged gap)
- AWS CLI used routinely instead of console-only work from Stage 5 onward
- AWS Systems Manager (Session Manager, at minimum, as a safer RDP alternative)
- A CloudFormation template recreating the Stage 2 network and instances, torn down and rebuilt at least once
- IAM: dedicated user with MFA, custom least-privilege policies (hand-written JSON, tested with a deliberate negative case), an instance role replacing any stored credentials
- KMS and Secrets Manager: at minimum, migrating PMBRS's existing `config.py`-based credential storage to Secrets Manager as a concrete before/after comparison
