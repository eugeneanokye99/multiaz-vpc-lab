# Multi-AZ Highly Available VPC – CloudFormation Lab

## Architecture Overview

```
Region
├── VPC (10.0.0.0/16)
│   ├── Availability Zone 1
│   │   ├── Public Subnet  (10.0.1.0/24)   ← Web Instance 1 + NAT GW 1
│   │   └── Private Subnet (10.0.11.0/24)  ← App Instance 1
│   └── Availability Zone 2
│       ├── Public Subnet  (10.0.2.0/24)   ← Web Instance 2 + NAT GW 2
│       └── Private Subnet (10.0.12.0/24)  ← App Instance 2
│
├── Internet Gateway (attached to VPC)
├── EIP 1 → NAT Gateway 1 (AZ1)
└── EIP 2 → NAT Gateway 2 (AZ2)
```

### Route Tables

| Route Table | Subnet(s) | 0.0.0.0/0 Target |
|---|---|---|
| Public RT | Public AZ1 + AZ2 | Internet Gateway |
| Private RT AZ1 | Private AZ1 | NAT Gateway 1 |
| Private RT AZ2 | Private AZ2 | NAT Gateway 2 |

---

## Files

| File | Purpose |
|---|---|
| `vpc-multiaz-stack.yaml` | Single CloudFormation template – deploys everything |
| `README.md` | This file – deployment guide, validation steps, design notes |

---

## Deployment

### Prerequisites
- AWS CLI configured (`aws configure`)
- Sufficient IAM permissions (EC2, VPC, IAM, CloudFormation, SSM)

### Deploy via AWS CLI

```bash
aws cloudformation deploy \
  --template-file vpc-multiaz-stack.yaml \
  --stack-name multiaz-vpc-lab \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    StudentName="Your Full Name" \
    LabName="Multi-AZ VPC Lab" \
    ProjectTag="MultiAZ-VPC-Lab"
```

### Deploy via AWS Console
1. Open **CloudFormation → Stacks → Create Stack**
2. Upload `vpc-multiaz-stack.yaml`
3. Fill in Parameters (StudentName, LabName, etc.)
4. Acknowledge IAM capability checkbox
5. Click **Create Stack** and wait ~5 minutes

### Get Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name multiaz-vpc-lab \
  --query "Stacks[0].Outputs" \
  --output table
```

---

## Validation Checklist

### 1. Apache Web Servers (browser test)
- Copy `WebInstance1PublicIP` and `WebInstance2PublicIP` from stack Outputs
- Visit `http://<WebInstance1PublicIP>` → should show your name + lab name
- Visit `http://<WebInstance2PublicIP>` → should show your name + lab name

### 2. Session Manager Access
```bash
# Connect to any instance
aws ssm start-session --target <INSTANCE_ID> --region <YOUR_REGION>
```
Or use the AWS Console: **EC2 → Instances → Select instance → Connect → Session Manager**

### 3. Public ↔ Private Ping Test
From a **Web Instance** (Session Manager session):
```bash
# Ping the private app instances (use their private IPs from EC2 console)
ping -c 4 <AppInstance1PrivateIP>
ping -c 4 <AppInstance2PrivateIP>
```

From an **App Instance** (Session Manager session):
```bash
ping -c 4 <WebInstance1PrivateIP>
ping -c 4 <WebInstance2PrivateIP>
```

### 4. Outbound Internet from Private Instances
```bash
# traceroute – first hop should be the NAT Gateway EIP
traceroute 8.8.8.8

# ping public internet
ping -c 4 google.com

# install a package (proves full egress works)
sudo yum install -y telnet
```

---

## Security Design Decisions

### No SSH Access
- No key pair is associated with any instance
- Security groups have **no inbound port 22 rules**
- All access is via **AWS Systems Manager Session Manager** (TLS over HTTPS)

### Least-Privilege Security Groups

| SG | Inbound | Outbound |
|---|---|---|
| Web-SG | TCP 80 from 0.0.0.0/0, ICMP from VPC CIDR | All |
| App-SG | ICMP from VPC CIDR only | All |

### IAM Role
- Single role with `AmazonSSMManagedInstanceCore` managed policy
- Grants SSM Agent permission to register with SSM, stream sessions, and use Parameter Store
- No other permissions – minimum required for management

---

## NAT Gateway Architecture Q&A

### What is the AWS Regional NAT Gateway?
AWS introduced a **Regional NAT Gateway** (sometimes called a "public NAT gateway" or referred to in the context of its HA behaviour): by default, every NAT Gateway you create is already AZ-scoped – it sits in one subnet inside one AZ and serves traffic from that AZ. AWS guarantees high availability *within* that AZ via redundant infrastructure, but the gateway itself does not span AZs.

### How does it differ from the two AZ-scoped NAT Gateways used here?

| Aspect | Two AZ-scoped NAT GWs (this lab) | Single NAT GW (cost-optimised) |
|---|---|---|
| **AZ failure** | Each AZ is self-sufficient; no egress loss if one AZ fails | Private instances in the NAT GW's AZ survive; other AZs lose egress |
| **Cross-AZ charges** | None – traffic stays in its AZ | Traffic from other AZs crosses AZ boundaries (billed) |
| **Cost** | Two EIPs + two NAT GW hours | One EIP + one NAT GW hour |
| **Fault isolation** | Full – each AZ independent | Partial – single point of failure for other AZs |

### How could a single NAT Gateway replace the two in this architecture?
You would:
1. Delete `NatGateway2`, `EIP2`, `PrivateRouteTable2`
2. Change `PrivateRoute2` to point at `NatGateway1`

**Trade-off:** Private instances in AZ2 would route through the NAT Gateway in AZ1, incurring cross-AZ data transfer fees and creating an AZ-dependency: if AZ1 is impaired, AZ2 private instances lose outbound internet access. This is acceptable for dev/test workloads where cost matters more than strict HA.

---

## Tagging Convention

All resources are tagged with:
- `Name` = `{ProjectTag}-{ResourceDescription}` (e.g. `MultiAZ-VPC-Lab-Web-AZ1`)
- `Project` = value of the `ProjectTag` parameter
- `Tier` = `Web` | `App` | `Public` | `Private` (where applicable)
- `AZ` = `AZ1` | `AZ2` (where applicable)

---

## Clean Up

```bash
aws cloudformation delete-stack --stack-name multiaz-vpc-lab
```

> **Note:** The stack deletion will also release the Elastic IPs and terminate all EC2 instances.