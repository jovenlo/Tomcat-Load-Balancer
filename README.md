# Tomcat + JBoss + AWS ALB — CloudFormation Deployment

Automated AWS infrastructure setup using a single CloudFormation template.  
Deploys **Tomcat** and **JBoss (WildFly)** on separate EC2 instances behind an **Application Load Balancer**.

---

## Architecture Overview

```
Internet
    │
    ▼
Application Load Balancer (port 80)
    │
    ├── 50% ──▶ LinuxServer1 — Tomcat 10.1.42 (port 8080)
    │
    └── 50% ──▶ LinuxServer2 — JBoss WildFly 32 (port 9090)
```

---

## What Gets Created

| Resource | Details |
|---|---|
| VPC | 10.0.0.0/16 — created automatically |
| Public Subnet 1 | 10.0.1.0/24 — AZ1 (for Tomcat server) |
| Public Subnet 2 | 10.0.2.0/24 — AZ2 (for JBoss server) |
| Internet Gateway | Attached to VPC with route table |
| LinuxServer1 | Java 21 + Tomcat 10.1.42 on port 8080 |
| LinuxServer2 | Java 21 + JBoss WildFly 32 on port 9090 |
| Security Group (ALB) | Allows port 80 from internet |
| Security Group (EC2) | Allows port 8080/9090 from ALB, port 22 from anywhere |
| Target Group (Tomcat) | Health check on port 8080 |
| Target Group (JBoss) | Health check on port 9090 |
| ALB | Internet-facing, splits traffic 50/50 |

---

## How It Works (Flow)

1. **CloudFormation reads the template** and provisions all resources in order
2. **VPC + Subnets + IGW** are created first — no existing network needed
3. **Security Groups** are configured — ALB is public; EC2 only accepts traffic from ALB
4. **Two EC2 instances launch** — each runs a UserData script on first boot:
   - LinuxServer1 installs Java 21 + Tomcat, starts on port 8080
   - LinuxServer2 installs Java 21 + JBoss WildFly, starts on port 9090
5. **Two Target Groups** are created — one per server, each on its own port
6. **ALB is created** and its listener on port 80 forwards traffic equally to both target groups
7. **Outputs tab** shows the ALB DNS URL — open it in a browser to access the app

---

## Prerequisites

- An AWS account with permissions to create VPC, EC2, IAM, and ELB resources
- An existing **EC2 Key Pair** in the region you are deploying to

---

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `KeyPairName` | *(required)* | Name of your existing EC2 Key Pair for SSH |
| `InstanceType` | `t3.micro` | EC2 size — `t3.micro`, `t3.small`, or `t3.medium` |

> **AMI, VPC, and Subnets are all handled automatically** — you do not need to provide them.

---

## Deployment Steps

### Option 1 — AWS Console

1. Go to **AWS Console → CloudFormation → Create Stack**
2. Select **Upload a template file** and upload `tomcat-jboss-alb.yaml`
3. Enter a stack name (e.g. `tomcat-jboss-stack`)
4. Fill in **KeyPairName** — select your key pair from the dropdown
5. Leave **InstanceType** as `t3.micro` or change if needed
6. Click through and hit **Create Stack**
7. Wait ~5 minutes for status to show `CREATE_COMPLETE`
8. Go to the **Outputs** tab and open the **ALBDNSName** URL

### Option 2 — AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name tomcat-jboss-stack \
  --template-body file://tomcat-jboss-alb.yaml \
  --parameters ParameterKey=KeyPairName,ParameterValue=YOUR_KEY_PAIR \
  --capabilities CAPABILITY_IAM
```

---

## Outputs

| Output | Description |
|---|---|
| `ALBDNSName` | Public URL to access the app via ALB |
| `LinuxServer1TomcatId` | EC2 Instance ID of the Tomcat server |
| `LinuxServer2JBossId` | EC2 Instance ID of the JBoss server |
| `VPCId` | ID of the VPC created by this stack |

---

## Software Versions

| Software | Version |
|---|---|
| Java | 21 (Amazon Corretto) |
| Tomcat | 10.1.42 |
| JBoss (WildFly) | 32.0.0.Final |
| OS | Amazon Linux 2023 (latest, auto-resolved) |

---

## SSH Access

```bash
# Connect to Tomcat server
ssh -i your-key.pem ec2-user@<LinuxServer1-Public-IP>

# Connect to JBoss server
ssh -i your-key.pem ec2-user@<LinuxServer2-Public-IP>

# Check UserData logs on either server
cat /var/log/userdata.log
```

---

## Cleanup

To delete all resources created by this stack:

```bash
aws cloudformation delete-stack --stack-name tomcat-jboss-stack
```

Or go to **CloudFormation → Select your stack → Delete**.

> ⚠️ This will permanently delete the VPC, EC2 instances, ALB, and all associated resources.

---

## Files

```
.
├── tomcat-jboss-alb.yaml   # CloudFormation template
└── README.md               # This file
```

---

## License

MIT
