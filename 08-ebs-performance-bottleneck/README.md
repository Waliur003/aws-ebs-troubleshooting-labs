# Lab 08 — EBS Performance Bottleneck

## Overview

This lab demonstrates how to investigate and troubleshoot slow storage performance on an Amazon EC2 instance using Amazon Elastic Block Store (EBS).

We deploy an Amazon Linux 2023 EC2 instance and attach a dedicated 5 GiB `gp3` EBS volume. We create an ext4 filesystem, mount it at `/data`, and install Linux performance tools.

Using `fio`, we reproduce an intentionally slow application workload by restricting its read throughput to 5 MiB/s.

We then investigate storage performance using `fio`, `iostat`, `lsblk`, AWS CLI, and Amazon CloudWatch.

After identifying the application-level rate limit, we remove the restriction and run the workload again.

We also examine the difference between EBS volume performance limits and EC2 instance EBS bandwidth limits.

An optional exercise demonstrates increasing provisioned gp3 throughput from 125 MiB/s to 250 MiB/s.

Finally, we verify data integrity and remove the temporary AWS infrastructure.

**Troubleshooting workflow:**

Provision → Baseline → Fault Injection → Performance Investigation → Root Cause Analysis → Remediation → Benchmark Comparison → Verification → Cleanup

---

## 1. Lab Information

| Item | Configuration |
|---|---|
| Incident ID | INC-EBS-008 |
| Difficulty | Intermediate–Advanced |
| AWS Region | `us-east-1` |
| AWS Services | EC2, EBS, CloudWatch, VPC, IAM, Systems Manager |
| Operating System | Amazon Linux 2023 |
| Instance Type | `t3.micro` |
| EBS Volume Type | `gp3` |
| EBS Volume Size | 5 GiB |
| Baseline IOPS | 3,000 |
| Baseline Throughput | 125 MiB/s |
| Optional Throughput | 250 MiB/s |
| Filesystem | ext4 |
| Mount Point | `/data` |
| Benchmark Tool | `fio` |
| Monitoring Tool | `iostat` |
| Benchmark File | `/data/fio-test/benchmark.bin` |
| Benchmark File Size | 1 GiB |
| Fault Injection | Apply a 5 MiB/s software rate limit |
| Expected Symptom | Slow sequential disk reads |
| Root Cause | Artificial application-level throughput restriction |
| Remediation | Remove `fio` rate limit |
| Verification | Compare benchmark results and verify original data |

---

## 2. Production Incident Scenario

### Incident Description

A company runs an application on an Amazon EC2 instance.

The application stores and processes files on a separate EBS volume mounted at `/data`.

Users begin reporting slow application responses and increased file-processing times.

The engineering team suspects an EBS storage performance problem.

Initial investigation shows that the application processes data more slowly than expected, despite using an EBS volume with 3,000 provisioned IOPS and 125 MiB/s throughput.

The cloud engineer must investigate the application workload, Linux storage performance, EBS metrics, and EC2 instance limits.

In this controlled lab, the slowdown is deliberately caused by a software throughput restriction.

The engineer identifies the restriction and removes it, then compares results to determine whether the workload performs better.

### Incident Symptoms

- EC2 instance is running.
- EBS volume is attached.
- EBS volume state is `in-use`.
- `/data` is mounted.
- Application storage operations are slow.
- The `fio` benchmark reports approximately 5 MiB/s during the fault.
- The gp3 volume is provisioned for higher throughput.
- Performance may improve after removing the application rate limit.

### Potential Business Impact

- Slow application responses.
- Increased file-processing times.
- Database query delays.
- Growing application queues.
- Longer batch-processing windows.
- Reduced user experience.
- Increased operational costs.
- Inefficient resource utilization.

### Objective

Determine whether slow storage performance is caused by an application limit, EBS provisioned performance, EC2 EBS bandwidth, or another workload constraint.

---

## 3. AWS Architecture

### Healthy Architecture

```text
                    Amazon EC2
                 Amazon Linux 2023
                         |
                         v
                   Application
                         |
                         v
                   Linux ext4
                         |
                         v
                 Mounted at /data
                         |
                         v
                  Amazon EBS gp3
                      5 GiB
                         |
              -----------------------
              |                     |
              v                     v
          3,000 IOPS           125 MiB/s
              |                     |
              -----------------------
                         |
                         v
                  Storage I/O
```

### Broken Architecture

```text
                  EC2 Application
                         |
                         v
                  fio Test Workload
                         |
                         v
                  Software Throttle
                    --rate=5M
                         |
                         v
                  Read Operations
                  Limited to 5 MiB/s
                         |
                         v
                  Slow Performance
                         |
                         v
                  Application Delay
```

### Recovered Architecture

```text
                   Slow Workload
                         |
                         v
                    Investigation
                 fio / iostat / AWS
                         |
                         v
                 Application Rate Cap
                      Identified
                         |
                         v
                 Remove --rate=5M
                         |
                         v
                  Repeat Benchmark
                         |
                         v
                  Compare Results
                         |
                         v
                  Verify Recovery
```

### Infrastructure Configuration

| Resource | Configuration |
|---|---|
| VPC | `10.179.0.0/16` |
| Public Subnet | `10.179.1.0/24` |
| Internet Gateway | Attached |
| Route Table | Public internet route |
| Security Group | No inbound rules |
| EC2 Management | Systems Manager |
| EBS Volume | 5 GiB gp3 |
| Filesystem | ext4 |
| Mount Point | `/data` |

---

## 4. Understanding EBS Performance

Amazon EBS performance depends on the EBS volume, EC2 instance, workload, and filesystem.

Three important performance measurements are IOPS, throughput, and latency.

### IOPS

IOPS means input/output operations per second.

It measures how many storage read and write requests are completed each second.

For example, if a device completes 3,000 I/O operations in one second:

```text
IOPS = 3,000
```

### Throughput

Throughput measures how much data is transferred per second.

It is commonly expressed in MiB/s.

For example:

```text
Throughput = 125 MiB/s
```

### Latency

Latency measures the time required to complete an I/O operation.

It is often measured in milliseconds.

For example:

```text
Read Latency = 2 ms
```

### Relationship Between IOPS and Throughput

For a workload with a consistent I/O size:

```text
Throughput ≈ IOPS × I/O Size
```

Example:

```text
IOPS = 1,000
I/O Size = 128 KiB

Throughput = 125 MiB/s
```

Actual throughput is also constrained by EBS provisioned settings, EC2 limits, and workload behavior.

### gp3 Baseline Performance

| Property | gp3 Baseline |
|---|---|
| IOPS | 3,000 |
| Throughput | 125 MiB/s |
| Performance Type | Provisioned |
| Burst Credit Dependency | None |
| IOPS and Throughput | Can be provisioned separately within service limits |

### gp3 vs. gp2

| Feature | gp2 | gp3 |
|---|---|---|
| Performance Model | Size-based baseline and bursting | Separately provisioned performance |
| Baseline IOPS | Depends on volume size | 3,000 included |
| Baseline Throughput | Varies by size and conditions | 125 MiB/s included |
| Burst Credits | Applicable to some workloads | Not used |
| Performance Tuning | Often requires increasing size | IOPS and throughput can be adjusted |

### Important EC2 Limitation

Even if an EBS volume supports a particular performance level, the EC2 instance may not be able to deliver that level of sustained performance.

`t3.micro` has relatively limited baseline EBS bandwidth compared with larger instances.

Therefore, increasing a gp3 volume from 125 MiB/s to 250 MiB/s does not guarantee that an application running on `t3.micro` will become faster.

---

## 5. Prerequisites and Safety

### Requirements

- AWS account
- AWS CLI v2
- Ubuntu WSL2
- Bash
- Amazon EC2 permissions
- Amazon EBS permissions
- CloudWatch read permissions
- IAM and VPC permissions
- Systems Manager
- Git and GitHub

### Safety Requirements

- Use a dedicated EBS data volume.
- Do not benchmark the EC2 root filesystem.
- Never run destructive raw-device benchmarks on a production volume.
- Verify `/data` is mounted before testing.
- Use a disposable test file.
- Keep important data separate from the benchmark file.
- Limit benchmark duration.
- Avoid unlimited concurrent I/O workloads.
- Monitor resource costs.
- Clean up all temporary infrastructure.

### Cost Considerations

EC2 instances, EBS volumes, public IPv4 addresses, and provisioned gp3 performance beyond included baseline levels may incur charges.

---

## 6. Prepare the AWS Environment

### Step 1 — Configure Variables

Run in Ubuntu WSL2:

```bash
export AWS_REGION="us-east-1"
export PROJECT_TAG="EBS-Troubleshooting-Lab08"

export INSTANCE_NAME="cloud-ebs-lab08-ec2"
export VOLUME_NAME="cloud-ebs-lab08-data"

export ROLE_NAME="cloud-ebs-lab08-ssm-role"
export PROFILE_NAME="cloud-ebs-lab08-ssm-profile"
export SG_NAME="cloud-ebs-lab08-sg"

export AZ_A="us-east-1a"
export VOLUME_SIZE="5"
export INSTANCE_TYPE="t3.micro"
```

### Step 2 — Verify AWS Access

```bash
aws sts get-caller-identity

aws --version
```

### Step 3 — Check AZ Availability

```bash
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters "Name=instance-type,Values=$INSTANCE_TYPE" \
  --query 'InstanceTypeOfferings[].Location' \
  --output table \
  --region "$AWS_REGION"
```

Use a supported Availability Zone.

### Step 4 — Check Existing Resources

```bash
aws ec2 describe-instances \
  --filters \
    "Name=tag:Project,Values=$PROJECT_TAG" \
    "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name}' \
  --output table \
  --region "$AWS_REGION"

aws ec2 describe-volumes \
  --filters "Name=tag:Project,Values=$PROJECT_TAG" \
  --query 'Volumes[].{ID:VolumeId,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

Avoid duplicate resources.

---

## 7. Create EC2 Infrastructure

### Step 1 — Create VPC

```bash
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block "10.179.0.0/16" \
  --tag-specifications \
    "ResourceType=vpc,Tags=[{Key=Name,Value=cloud-ebs-lab08-vpc},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Vpc.VpcId' \
  --output text \
  --region "$AWS_REGION")
```

Enable DNS:

```bash
aws ec2 modify-vpc-attribute \
  --vpc-id "$VPC_ID" \
  --enable-dns-support \
  --region "$AWS_REGION"

aws ec2 modify-vpc-attribute \
  --vpc-id "$VPC_ID" \
  --enable-dns-hostnames \
  --region "$AWS_REGION"
```

### Step 2 — Create Public Subnet

```bash
export SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block "10.179.1.0/24" \
  --availability-zone "$AZ_A" \
  --tag-specifications \
    "ResourceType=subnet,Tags=[{Key=Name,Value=cloud-ebs-lab08-subnet},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 3 — Create Internet Gateway

```bash
export IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications \
    "ResourceType=internet-gateway,Tags=[{Key=Name,Value=cloud-ebs-lab08-igw},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'InternetGateway.InternetGatewayId' \
  --output text \
  --region "$AWS_REGION")

aws ec2 attach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID" \
  --region "$AWS_REGION"
```

### Step 4 — Create Route Table

```bash
export RT_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
    "ResourceType=route-table,Tags=[{Key=Name,Value=cloud-ebs-lab08-rt},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'RouteTable.RouteTableId' \
  --output text \
  --region "$AWS_REGION")

aws ec2 create-route \
  --route-table-id "$RT_ID" \
  --destination-cidr-block "0.0.0.0/0" \
  --gateway-id "$IGW_ID" \
  --region "$AWS_REGION"

export RT_ASSOC_ID=$(aws ec2 associate-route-table \
  --route-table-id "$RT_ID" \
  --subnet-id "$SUBNET_ID" \
  --query AssociationId \
  --output text \
  --region "$AWS_REGION"
```

### Step 5 — Create Security Group

```bash
export SG_ID=$(aws ec2 create-security-group \
  --group-name "$SG_NAME" \
  --description "Lab08 EBS performance investigation" \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
    "ResourceType=security-group,Tags=[{Key=Name,Value=$SG_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query GroupId \
  --output text \
  --region "$AWS_REGION")
```

No inbound rules are required for Systems Manager.

### Step 6 — Create IAM Trust Policy

Save as `ec2-trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Step 7 — Create IAM Role

Verify the intended IAM role and profile names are unused.

```bash
aws iam create-role \
  --role-name "$ROLE_NAME" \
  --assume-role-policy-document file://ec2-trust-policy.json \
  --tags Key=Project,Value="$PROJECT_TAG"

aws iam attach-role-policy \
  --role-name "$ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam create-instance-profile \
  --instance-profile-name "$PROFILE_NAME" \
  --tags Key=Project,Value="$PROJECT_TAG"

aws iam add-role-to-instance-profile \
  --instance-profile-name "$PROFILE_NAME" \
  --role-name "$ROLE_NAME"
```

Allow IAM changes to propagate.

### Step 8 — Retrieve Amazon Linux AMI

```bash
export AMI_ID=$(aws ssm get-parameter \
  --name "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64" \
  --query Parameter.Value \
  --output text \
  --region "$AWS_REGION")
```

### Step 9 — Launch EC2

```bash
export INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type "$INSTANCE_TYPE" \
  --iam-instance-profile "Name=$PROFILE_NAME" \
  --network-interfaces \
    "DeviceIndex=0,SubnetId=$SUBNET_ID,Groups=$SG_ID,AssociatePublicIpAddress=true" \
  --metadata-options "HttpTokens=required,HttpEndpoint=enabled" \
  --count 1 \
  --tag-specifications \
    "ResourceType=instance,Tags=[{Key=Name,Value=$INSTANCE_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Instances[0].InstanceId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 10 — Wait for EC2

```bash
aws ec2 wait instance-running \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait instance-status-ok \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

### Step 11 — Retrieve Instance AZ

```bash
export INSTANCE_AZ=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' \
  --output text \
  --region "$AWS_REGION")

echo "$INSTANCE_AZ"
```

---

## 8. Create the gp3 EBS Volume

### Step 1 — Create EBS

```bash
export VOLUME_ID=$(aws ec2 create-volume \
  --availability-zone "$INSTANCE_AZ" \
  --size "$VOLUME_SIZE" \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125 \
  --tag-specifications \
    "ResourceType=volume,Tags=[{Key=Name,Value=$VOLUME_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query VolumeId \
  --output text \
  --region "$AWS_REGION")
```

### Step 2 — Wait Until Available

```bash
aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 3 — Verify EBS Performance Settings

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,Type:VolumeType,Size:Size,IOPS:Iops,Throughput:Throughput,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
Volume Type: gp3
Size: 5 GiB
IOPS: 3000
Throughput: 125 MiB/s
State: available
```

### Step 4 — Attach EBS

```bash
aws ec2 attach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdf \
  --region "$AWS_REGION"

aws ec2 wait volume-in-use \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 5 — Verify Attachment

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].Attachments' \
  --output json \
  --region "$AWS_REGION"
```

Expected: volume attached to the correct instance.

---

## 9. Prepare Linux Filesystem and Tools

### Step 1 — Connect Through Systems Manager

```bash
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
  --query 'InstanceInformationList[].{ID:InstanceId,Status:PingStatus}' \
  --output table \
  --region "$AWS_REGION"
```

Once the instance is online:

```bash
aws ssm start-session \
  --target "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

The following Linux commands run inside EC2.

### Step 2 — Install Benchmark Tools

```bash
sudo dnf install -y fio sysstat
```

Verify:

```bash
fio --version

iostat -V
```

### Step 3 — Identify the EBS Device

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,SERIAL
```

Verify the 5 GiB disk by comparing its NVMe serial with the AWS volume ID.

For example:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

After verification:

```bash
export DATA_DEVICE="/dev/nvme1n1"
export DATA_DISK="$(basename "$DATA_DEVICE")"
```

Replace the example device name if necessary.

### Step 4 — Inspect the Empty Volume

```bash
lsblk -f

sudo blkid "$DATA_DEVICE" || true

sudo file -s "$DATA_DEVICE"
```

Expected: blank EBS data volume.

### Step 5 — Create ext4

**Destructive command: Only run on the verified blank Lab 08 data disk.**

```bash
sudo mkfs.ext4 "$DATA_DEVICE"
```

### Step 6 — Mount the Volume

```bash
sudo mkdir -p /data

sudo mount "$DATA_DEVICE" /data

sudo chown ec2-user:ec2-user /data
```

Verify:

```bash
findmnt /data

df -hT /data
```

### Step 7 — Create Benchmark Directory

```bash
sudo mkdir -p /data/fio-test

sudo chown ec2-user:ec2-user /data/fio-test
```

### Step 8 — Create Important Test Data

```bash
echo "EBS Lab 08 - Important Application Data" \
  | sudo tee /data/important-data.txt
```

Record checksum:

```bash
sha256sum /data/important-data.txt
```

### Step 9 — Establish Healthy Baseline

```bash
df -h /

df -h /data

findmnt /data

iostat -dx 1 5
```

Expected: `/data` is mounted on the dedicated EBS volume and has enough free space for benchmarking.

---

## 10. Create the Benchmark Workload

### Step 1 — Verify Mount Safety

```bash
mountpoint -q /data || {
  echo "ERROR: /data is not mounted"
  exit 1
}
```

Verify the source:

```bash
findmnt -n -o SOURCE,FSTYPE,TARGET /data
```

Confirm the source matches the verified Lab 08 data device.

### Step 2 — Prepare a 1 GiB Test File

```bash
sudo -u ec2-user fio \
  --name=prepare \
  --filename=/data/fio-test/benchmark.bin \
  --size=1G \
  --rw=write \
  --bs=1M \
  --ioengine=sync \
  --direct=1 \
  --group_reporting
```

This writes a disposable benchmark file.

The original application test file remains separate.

### Step 3 — Verify Benchmark File

```bash
ls -lh /data/fio-test/

df -hT /data
```

Expected: a 1 GiB benchmark file and available filesystem space.

---

## 11. Fault Injection — Artificial Throughput Limit

### Step 1 — Run the Throttled Workload

```bash
sudo -u ec2-user fio \
  --name=slow-application \
  --filename=/data/fio-test/benchmark.bin \
  --size=1G \
  --rw=read \
  --bs=1M \
  --ioengine=libaio \
  --iodepth=4 \
  --direct=1 \
  --rate=5M \
  --runtime=30 \
  --time_based \
  --group_reporting
```

The setting:

```text
--rate=5M
```

limits the benchmark workload to approximately 5 MiB/s.

### Step 2 — Inspect the Results

Find the `READ` throughput summary.

Illustrative output:

```text
READ: bw=5120KiB/s
```

This is approximately 5 MiB/s.

The exact result may differ depending on the environment.

### Step 3 — Investigate Device Activity

While the workload is running, use another EC2 session:

```bash
iostat -dx 1 10
```

Inspect the dedicated EBS device.

### Step 4 — Verify EBS Provisioned Performance

From WSL2:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{IOPS:Iops,Throughput:Throughput,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
IOPS: 3000
Throughput: 125
State: in-use
```

### Fault Summary

| Check | Expected Finding |
|---|---|
| EC2 | Running |
| EBS | Attached |
| gp3 IOPS | 3,000 |
| gp3 Throughput | 125 MiB/s |
| Linux Filesystem | Mounted |
| Application Benchmark | Rate limited |
| Benchmark Throughput | Approximately 5 MiB/s |
| Root Cause | Explicit software rate limit |

**Important:** This test simulates application-level throttling. It does not intentionally change EBS volume performance or prove that an AWS EBS limit was reached.

---

## 12. Troubleshooting Investigation

### Check 1 — Inspect Benchmark Configuration

Review the command used for the slow test.

The important setting is:

```text
--rate=5M
```

Finding: the benchmark is explicitly restricted.

### Check 2 — Inspect Linux Disk Performance

```bash
iostat -dx 1 5
```

Relevant metrics:

| Metric | Meaning |
|---|---|
| `r/s` | Reads per second |
| `w/s` | Writes per second |
| `r_await` | Average read completion time |
| `w_await` | Average write completion time |
| `aqu-sz` | Average outstanding I/O requests |
| `%util` | Device active-time percentage |

High `%util` does not necessarily mean a modern NVMe volume has reached its maximum performance.

### Check 3 — Inspect Provisioned EBS IOPS

From WSL2:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].Iops' \
  --output text \
  --region "$AWS_REGION"
```

Expected: `3000`.

### Check 4 — Inspect Provisioned Throughput

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].Throughput' \
  --output text \
  --region "$AWS_REGION"
```

Expected: `125` MiB/s.

### Check 5 — Check EC2 EBS Limits

```bash
aws ec2 describe-instance-types \
  --instance-types "$INSTANCE_TYPE" \
  --query 'InstanceTypes[0].{Type:InstanceType,EBS:EbsInfo.EbsOptimizedInfo}' \
  --output json \
  --region "$AWS_REGION"
```

Finding: the EC2 instance has its own EBS throughput and IOPS limits.

These can be lower than the provisioned capability of the attached volumes.

### Check 6 — Inspect CPU and Memory

Inside EC2:

```bash
uptime

free -h

top -b -n 1 | head -20
```

Check whether CPU pressure or memory exhaustion might affect the application.

### Check 7 — Inspect Filesystem Capacity

```bash
df -hT /data

df -i /data
```

Finding: the filesystem should have adequate free space.

### Investigation Summary

| Component | Finding |
|---|---|
| EC2 | Running |
| EBS Type | gp3 |
| EBS IOPS | 3,000 |
| Provisioned Throughput | 125 MiB/s |
| Filesystem | ext4 |
| Benchmark | Sequential read |
| I/O Size | 1 MiB |
| Fault Setting | `--rate=5M` |
| Root Cause | Application-level throttle |
| EC2 Limitation | Must be considered during performance analysis |

---

## 13. Investigate CloudWatch EBS Metrics

Amazon CloudWatch provides EBS volume metrics for monitoring storage activity and performance.

### Step 1 — List EBS Metrics

From WSL2:

```bash
aws cloudwatch list-metrics \
  --namespace AWS/EBS \
  --dimensions "Name=VolumeId,Value=$VOLUME_ID" \
  --region "$AWS_REGION"
```

Metrics may not appear immediately if the volume was recently created or has not published datapoints yet.

### Step 2 — Define Monitoring Window

```bash
export END_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

export START_TIME=$(date -u -d "30 minutes ago" +"%Y-%m-%dT%H:%M:%SZ")
```

### Step 3 — Check VolumeQueueLength

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeQueueLength \
  --dimensions "Name=VolumeId,Value=$VOLUME_ID" \
  --start-time "$START_TIME" \
  --end-time "$END_TIME" \
  --period 60 \
  --statistics Average Maximum \
  --region "$AWS_REGION" \
  --output json
```

A growing queue accompanied by high latency can indicate insufficient effective storage capacity.

Queue length alone does not identify which layer is responsible.

### Step 4 — Check VolumeReadOps

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions "Name=VolumeId,Value=$VOLUME_ID" \
  --start-time "$START_TIME" \
  --end-time "$END_TIME" \
  --period 60 \
  --statistics Sum \
  --region "$AWS_REGION" \
  --output json
```

Calculate read IOPS:

```text
Read IOPS = VolumeReadOps Sum / 60
```

For example:

```text
60,000 operations / 60 seconds = 1,000 IOPS
```

### Step 5 — Check VolumeReadBytes

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadBytes \
  --dimensions "Name=VolumeId,Value=$VOLUME_ID" \
  --start-time "$START_TIME" \
  --end-time "$END_TIME" \
  --period 60 \
  --statistics Sum \
  --region "$AWS_REGION" \
  --output json
```

Calculate throughput:

```text
MiB/s = VolumeReadBytes Sum / (60 × 1,048,576)
```

### Step 6 — Inspect IOPS Exceeded Check

On supported Nitro-based EBS attachments:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeIOPSExceededCheck \
  --dimensions "Name=VolumeId,Value=$VOLUME_ID" \
  --start-time "$START_TIME" \
  --end-time "$END_TIME" \
  --period 60 \
  --statistics Maximum \
  --region "$AWS_REGION" \
  --output json
```

A returned value of `1` indicates that volume IOPS demand exceeded provisioned performance during that interval.

A returned value of `0` indicates no exceedance was reported in that interval.

### Step 7 — Inspect Throughput Exceeded Check

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeThroughputExceededCheck \
  --dimensions "Name=VolumeId,Value=$VOLUME_ID" \
  --start-time "$START_TIME" \
  --end-time "$END_TIME" \
  --period 60 \
  --statistics Maximum \
  --region "$AWS_REGION" \
  --output json
```

A value of `1` can indicate the volume's provisioned throughput was exceeded.

Empty datapoints do not prove that the volume was healthy. Metric availability depends on supported volume and instance configurations and published activity.

### Step 8 — Inspect Read Latency

Where supported:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeAvgReadLatency \
  --dimensions "Name=VolumeId,Value=$VOLUME_ID" \
  --start-time "$START_TIME" \
  --end-time "$END_TIME" \
  --period 60 \
  --statistics Average \
  --region "$AWS_REGION" \
  --output json
```

Compare latency with throughput, IOPS, queue length, and application response times.

### CloudWatch Investigation Summary

| Metric | Purpose |
|---|---|
| `VolumeReadOps` | Read operation count |
| `VolumeWriteOps` | Write operation count |
| `VolumeReadBytes` | Bytes read |
| `VolumeWriteBytes` | Bytes written |
| `VolumeQueueLength` | Outstanding I/O requests |
| `VolumeIOPSExceededCheck` | Detects exceeded provisioned IOPS on supported configurations |
| `VolumeThroughputExceededCheck` | Detects exceeded provisioned throughput on supported configurations |
| `VolumeAvgReadLatency` | Average read latency on supported configurations |
| `VolumeAvgWriteLatency` | Average write latency on supported configurations |

---

## 14. Root Cause Analysis

### Root Cause

The controlled slowdown was caused by the `fio` workload being configured with an artificial read throughput limit of 5 MiB/s.

The EBS volume was not intentionally misconfigured.

The configured gp3 throughput remained 125 MiB/s.

### Technical Explanation

The application was not generating storage requests faster than the configured software rate limit.

The storage subsystem could not deliver more application throughput than the workload requested.

This is an important distinction between workload-generated demand and the provisioned capacity of the underlying storage.

### Root Cause Flow

```text
Application Workload
        |
        v
fio --rate=5M
        |
        v
Software Limits I/O Requests
        |
        v
Observed Read Throughput ~5 MiB/s
        |
        v
Slow Workload
        |
        v
Application-Level Bottleneck Identified
```

### Five Whys

1. **Why was throughput low?** The benchmark was restricted to approximately 5 MiB/s.
2. **Why was the workload restricted?** It used the `--rate=5M` option.
3. **Why didn't the EBS volume deliver 125 MiB/s?** The workload did not request that level of throughput.
4. **Why is checking AWS limits alone insufficient?** Provisioned storage capacity does not prove that an application is demanding or achieving it.
5. **How can the problem be prevented?** Investigate application workload configuration before increasing storage performance.

### Prevention

- Monitor application-level throughput.
- Monitor EBS volume performance.
- Check EC2 EBS bandwidth limits.
- Review rate limits and concurrency settings.
- Compare benchmark results under controlled conditions.
- Avoid purchasing additional IOPS or throughput without identifying the true bottleneck.

---

## 15. Remediation — Remove the Rate Limit

### Step 1 — Run the Unthrottled Workload

Inside EC2:

```bash
sudo -u ec2-user fio \
  --name=recovered-application \
  --filename=/data/fio-test/benchmark.bin \
  --size=1G \
  --rw=read \
  --bs=1M \
  --ioengine=libaio \
  --iodepth=4 \
  --direct=1 \
  --runtime=30 \
  --time_based \
  --group_reporting
```

The important change is removing:

```text
--rate=5M
```

### Step 2 — Compare Performance

Record the actual benchmark results.

| Measurement | During Fault | After Fix |
|---|---|---|
| Workload | Sequential read | Sequential read |
| Block Size | 1 MiB | 1 MiB |
| I/O Depth | 4 | 4 |
| Runtime | 30 seconds | 30 seconds |
| Software Limit | 5 MiB/s | Removed |
| Read Throughput | Approximately 5 MiB/s | Record actual output |
| Expected Result | Restricted | Improved if other capacity is available |

### Step 3 — Inspect Disk Performance

```bash
iostat -dx 1 10
```

Compare the storage device's throughput, queue length, and latency.

### Step 4 — Verify Original Data

```bash
cat /data/important-data.txt

sha256sum /data/important-data.txt
```

Compare the checksum with the original value.

### Remediation Summary

| Check | Before | After |
|---|---|---|
| Application Rate Limit | 5 MiB/s | Removed |
| EBS Volume | gp3 | gp3 |
| EBS Provisioned IOPS | 3,000 | 3,000 |
| EBS Provisioned Throughput | 125 MiB/s | 125 MiB/s |
| Benchmark Throughput | Throttled | Re-measured |
| Filesystem | ext4 | ext4 |
| Application Data | Preserved | Verified |

---

## 16. Optional Advanced Exercise — Modify gp3 Throughput

This optional exercise demonstrates Amazon EBS Elastic Volumes performance configuration.

### Step 1 — Verify Current Throughput

From WSL2:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{IOPS:Iops,Throughput:Throughput,Type:VolumeType}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
IOPS: 3000
Throughput: 125
Type: gp3
```

### Step 2 — Increase Throughput to 250 MiB/s

```bash
aws ec2 modify-volume \
  --volume-id "$VOLUME_ID" \
  --throughput 250 \
  --region "$AWS_REGION"
```

### Step 3 — Monitor Modification

```bash
aws ec2 describe-volumes-modifications \
  --volume-ids "$VOLUME_ID" \
  --query 'VolumesModifications[].{Original:OriginalThroughput,Target:TargetThroughput,State:ModificationState,Progress:Progress}' \
  --output table \
  --region "$AWS_REGION"
```

Wait until the modification reaches `optimizing` or `completed`.

### Step 4 — Verify Provisioned Throughput

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{IOPS:Iops,Throughput:Throughput,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

Expected: 250 MiB/s provisioned throughput.

### Step 5 — Repeat Benchmark

Inside EC2:

```bash
sudo -u ec2-user fio \
  --name=post-ebs-modification \
  --filename=/data/fio-test/benchmark.bin \
  --size=1G \
  --rw=read \
  --bs=1M \
  --ioengine=libaio \
  --iodepth=4 \
  --direct=1 \
  --runtime=30 \
  --time_based \
  --group_reporting
```

### Step 6 — Interpret the Result

If performance improves, the volume's new configuration may have provided additional effective capacity.

If performance does not improve, investigate the EC2 EBS bandwidth limit, workload characteristics, and other resource constraints.

**Important:** A successful volume modification does not guarantee increased application performance.

### Production Remediation Guide

| Root Cause | Possible Remediation |
|---|---|
| Application rate limit | Remove or adjust rate limit |
| Insufficient gp3 IOPS | Increase provisioned IOPS if justified |
| Insufficient gp3 throughput | Increase provisioned throughput |
| EC2 bandwidth limit | Choose an instance with higher EBS bandwidth |
| High I/O contention | Optimize workload concurrency |
| Poor access pattern | Optimize read/write sizes and access patterns |
| Filesystem nearly full | Safely reclaim or expand space |
| Snapshot initialization | Account for initialization or use supported initialization features |

---

## 17. Final Verification

### AWS Verification

From WSL2:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{State:State,Size:Size,IOPS:Iops,Throughput:Throughput}' \
  --output table \
  --region "$AWS_REGION"
```

### Linux Verification

Inside EC2:

```bash
lsblk -f

findmnt /data

df -hT /data

cat /data/important-data.txt

sha256sum /data/important-data.txt
```

### Performance Comparison Table

Record the actual results from your benchmark runs.

| Measurement | Throttled | Unthrottled | Optional gp3 Modified |
|---|---|---|---|
| EBS Provisioned IOPS | 3,000 | 3,000 | 3,000 |
| EBS Provisioned Throughput | 125 MiB/s | 125 MiB/s | 250 MiB/s |
| Application Rate Limit | 5 MiB/s | None | None |
| Measured Throughput | Record result | Record result | Record result |
| Read Latency | Record result | Record result | Record result |
| Queue Length | Record result | Record result | Record result |
| Data Integrity | Verify | Verify | Verify |

### Success Criteria

- EC2 instance provisioned.
- 5 GiB gp3 volume attached.
- ext4 filesystem mounted at `/data`.
- `fio` and `sysstat` installed.
- Benchmark test file created.
- Software rate limit reproduced.
- Throttled throughput measured.
- Linux device metrics investigated.
- EBS provisioned performance verified.
- EC2 EBS bandwidth limit checked.
- CloudWatch metrics reviewed.
- Software throttle removed.
- Benchmark repeated.
- Results compared.
- Original data preserved.
- Optional gp3 throughput modification tested.
- Temporary resources cleaned up.

---

## 18. Production Best Practices

### EBS Performance Troubleshooting Checklist

1. Confirm the affected application.
2. Identify the correct EBS volume.
3. Inspect the volume type.
4. Review provisioned IOPS.
5. Review provisioned throughput.
6. Check EC2 instance EBS bandwidth.
7. Inspect application workload limits.
8. Measure storage latency and throughput.
9. Check I/O queue length.
10. Review CloudWatch metrics.
11. Check filesystem usage.
12. Examine CPU and memory pressure.
13. Change only the identified bottleneck.
14. Repeat measurements under comparable conditions.
15. Record performance before and after remediation.

### Performance Best Practices

- Use gp3 for flexible general-purpose SSD performance.
- Monitor IOPS and throughput independently.
- Understand the application's I/O size.
- Use representative benchmarking workloads.
- Avoid benchmarking production volumes with destructive tests.
- Compare workload demand with instance and volume limits.
- Use CloudWatch alarms where appropriate.
- Review performance trends before increasing provisioned capacity.

### Security Considerations

- Use least-privilege IAM.
- Restrict access to EBS operations.
- Use Systems Manager when appropriate.
- Protect application files.
- Avoid storing credentials in benchmark files.
- Do not benchmark production block devices without approval.
- Verify resource ownership before cleanup.

---

## 19. Cleanup

**Warning:** Cleanup permanently deletes the disposable benchmark data, EBS volume, EC2 instance, and supporting AWS infrastructure.

### Step 1 — Remove the Benchmark File

Inside EC2:

```bash
ls -lh /data/fio-test/

cat /data/important-data.txt
```

Remove only the test file:

```bash
sudo rm -f /data/fio-test/benchmark.bin

sudo rmdir /data/fio-test
```

### Step 2 — Unmount the Data Volume

```bash
cd /

sudo fuser -vm /data

sync

sudo umount /data
```

Verify:

```bash
findmnt /data
```

Exit EC2:

```bash
exit
```

### Step 3 — Verify AWS Ownership

From WSL2:

```bash
aws sts get-caller-identity

aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,State:State,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"

aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"
```

Confirm the project tag is `EBS-Troubleshooting-Lab08`.

### Step 4 — Detach EBS

```bash
aws ec2 detach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 5 — Delete Data Volume

```bash
aws ec2 delete-volume \
  --volume-id "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 6 — Verify Root Volume Settings

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].{RootDevice:RootDeviceName,BlockDevices:BlockDeviceMappings}' \
  --output json \
  --region "$AWS_REGION"
```

Verify the root volume is configured for deletion on termination.

### Step 7 — Terminate EC2

```bash
aws ec2 terminate-instances \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait instance-terminated \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

### Step 8 — Delete IAM Resources

Verify the IAM resources belong only to Lab 08.

```bash
aws iam list-role-tags \
  --role-name "$ROLE_NAME"

aws iam get-instance-profile \
  --instance-profile-name "$PROFILE_NAME"
```

Remove:

```bash
aws iam detach-role-policy \
  --role-name "$ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam remove-role-from-instance-profile \
  --instance-profile-name "$PROFILE_NAME" \
  --role-name "$ROLE_NAME"

aws iam delete-instance-profile \
  --instance-profile-name "$PROFILE_NAME"

aws iam delete-role \
  --role-name "$ROLE_NAME"
```

### Step 9 — Check Network Interfaces

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'NetworkInterfaces[].{ID:NetworkInterfaceId,Status:Status}' \
  --output table \
  --region "$AWS_REGION"
```

Wait until the instance's network interfaces have been removed.

### Step 10 — Delete Networking

```bash
aws ec2 delete-security-group \
  --group-id "$SG_ID" \
  --region "$AWS_REGION"

aws ec2 disassociate-route-table \
  --association-id "$RT_ASSOC_ID" \
  --region "$AWS_REGION"

aws ec2 delete-route-table \
  --route-table-id "$RT_ID" \
  --region "$AWS_REGION"

aws ec2 delete-subnet \
  --subnet-id "$SUBNET_ID" \
  --region "$AWS_REGION"

aws ec2 detach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID" \
  --region "$AWS_REGION"

aws ec2 delete-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --region "$AWS_REGION"

aws ec2 delete-vpc \
  --vpc-id "$VPC_ID" \
  --region "$AWS_REGION"
```

### Step 11 — Verify Cleanup

```bash
aws ec2 describe-volumes \
  --filters "Name=tag:Project,Values=$PROJECT_TAG" \
  --query 'Volumes[].VolumeId' \
  --output json \
  --region "$AWS_REGION"

aws ec2 describe-vpcs \
  --filters "Name=tag:Project,Values=$PROJECT_TAG" \
  --query 'Vpcs[].VpcId' \
  --output json \
  --region "$AWS_REGION"
```

Expected: empty arrays after cleanup completes.

Also verify the EC2 instance, root EBS volume, IAM role, and instance profile are gone.

### Cleanup Summary

| Resource | Expected Result |
|---|---|
| Benchmark File | Deleted |
| EBS Data Volume | Deleted |
| EC2 Instance | Terminated |
| Root EBS Volume | Deleted |
| IAM Role | Deleted |
| Instance Profile | Deleted |
| Security Group | Deleted |
| Route Table | Deleted |
| Public Subnet | Deleted |
| Internet Gateway | Deleted |
| Lab VPC | Deleted |
| Default VPC | Unchanged |

---

## 20. Interview Questions and Answers

### Q1. What are IOPS?

IOPS measures the number of storage input/output operations completed per second.

### Q2. What is throughput?

Throughput measures the amount of data transferred per second.

### Q3. What is latency?

Latency measures the time required to complete an I/O request.

### Q4. What is the difference between IOPS and throughput?

IOPS measures the number of operations. Throughput measures how much data is transferred.

### Q5. What are the baseline gp3 performance settings?

3,000 IOPS and 125 MiB/s.

### Q6. Does gp3 use burst credits?

No. gp3 performance does not rely on gp2-style I/O burst credits.

### Q7. Which Linux tools help investigate storage performance?

Tools include `fio`, `iostat`, `lsblk`, `df`, and `top`.

### Q8. What does high EBS queue length suggest?

I/O demand may be exceeding effective service capacity, although queue length must be interpreted alongside latency and throughput.

### Q9. Can EC2 limit EBS performance?

Yes. EC2 instances have their own EBS bandwidth and IOPS limits.

### Q10. Why might increasing gp3 throughput fail to improve performance?

Because the bottleneck may be the EC2 instance, application configuration, or another resource instead of the EBS volume.

### Q11. What does `fio --direct=1` do?

It requests direct I/O, reducing the influence of normal filesystem page caching on the benchmark.

### Q12. How would you explain this lab in an interview?

I deployed an Amazon Linux EC2 instance and attached a dedicated gp3 EBS volume.

I created an ext4 filesystem and installed `fio` and `sysstat`.

I intentionally restricted a benchmark workload to 5 MiB/s and reproduced slow storage throughput.

I investigated the workload using `fio`, `iostat`, AWS CLI, and CloudWatch.

I compared the benchmark results with the gp3 volume's provisioned IOPS and throughput and reviewed the EC2 instance's EBS bandwidth limits.

After identifying the application-level rate limit as the controlled root cause, I removed the restriction and repeated the benchmark.

I also practiced increasing gp3 provisioned throughput and learned why higher EBS settings do not always translate into better application performance.

---

## 21. Repository Structure

```text
08-ebs-performance-bottleneck/
├── README.md
└── ec2-trust-policy.json
```

### File Descriptions

| File | Purpose |
|---|---|
| `README.md` | Complete EBS performance troubleshooting documentation |
| `ec2-trust-policy.json` | EC2 trust policy for Systems Manager |

Do not commit AWS credentials, private keys, benchmark data, or sensitive application files.

---

## 22. Lessons Learned

This lab demonstrates that slow storage performance does not always mean an Amazon EBS volume is underprovisioned.

The application workload itself can limit the throughput it requests.

In this exercise, the performance restriction came from a software rate limit configured in `fio`.

We used Linux performance tools and AWS metrics to investigate the slowdown.

We also learned how to distinguish IOPS, throughput, latency, and queue length.

An important finding is that EBS volume performance and EC2 instance EBS bandwidth must be considered together.

Simply increasing gp3 throughput may not improve application performance when another resource is limiting the workload.

**Key Lesson:** Investigate application workload behavior, EC2 limits, and EBS metrics before increasing storage performance. The most effective fix addresses the actual bottleneck.

---

## 23. Skills Demonstrated

- Amazon EBS provisioning
- gp3 IOPS and throughput configuration
- AWS CLI v2
- Linux ext4 filesystem management
- Storage benchmarking with `fio`
- Storage monitoring with `iostat`
- CloudWatch EBS metrics
- Throughput analysis
- IOPS analysis
- Latency investigation
- I/O queue analysis
- EC2 EBS bandwidth evaluation
- Application rate-limit troubleshooting
- EBS Elastic Volumes
- Performance verification
- SHA-256 data verification
- Root cause analysis
- Safe AWS resource cleanup

## Final Outcome

| Item | Expected Result |
|---|---|
| Incident | INC-EBS-008 |
| Problem | Slow EBS-backed application workload |
| EBS Type | gp3 |
| EBS Size | 5 GiB |
| Provisioned IOPS | 3,000 |
| Baseline Throughput | 125 MiB/s |
| Fault Injection | 5 MiB/s software rate limit |
| Root Cause | Application-level throttling |
| Investigation | `fio`, `iostat`, AWS CLI, CloudWatch |
| Remediation | Remove `--rate=5M` |
| Benchmark Verification | Compare measured results |
| Optional Modification | 250 MiB/s provisioned throughput |
| Original Data | Preserved |
| Cleanup | Temporary resources removed |
| Expected Incident Status | Resolved |

**Key Takeaway:** Effective EBS performance troubleshooting requires measuring the entire storage path. Always distinguish application-level throttling, EBS volume limits, and EC2 instance limits before making performance changes.
