# Lab 03 — EBS Filesystem Full

## Overview

This lab demonstrates how to troubleshoot a full Linux filesystem hosted on an Amazon Elastic Block Store (EBS) volume.

We deploy an Amazon Linux 2023 EC2 instance and attach a separate 5 GiB `gp3` EBS volume. We create an ext4 filesystem and mount it at `/data`.

After creating a small application configuration file, we intentionally fill the data filesystem with a large disposable test file.

When the filesystem runs out of usable space, new application writes fail with the error:

```text
No space left on device
```

We investigate the issue using Linux commands including `df`, `du`, `find`, `lsblk`, and `df -i`.

After identifying the file responsible for the capacity problem, we remove only the disposable test data, confirm that free space is restored, and verify that the original application file remains intact.

Finally, we safely detach and delete the lab EBS volume and clean up the temporary AWS infrastructure.

**Troubleshooting workflow:**

Provision → Healthy Baseline → Fault Injection → Investigation → Root Cause Analysis → Remediation → Verification → Cleanup

---

## 1. Lab Information

| Item | Configuration |
|---|---|
| Incident ID | INC-EBS-003 |
| Difficulty | Beginner |
| AWS Region | `us-east-1` |
| AWS Services | EC2, EBS, VPC, IAM, Systems Manager |
| Operating System | Amazon Linux 2023 |
| Instance Type | `t3.micro` |
| EBS Volume Type | `gp3` |
| EBS Volume Size | 5 GiB |
| EBS Attachment | `/dev/sdf` |
| Linux Block Device | NVMe device such as `/dev/nvme1n1` |
| Filesystem | ext4 |
| Mount Directory | `/data` |
| Fault Injection | Fill the filesystem using `dd` |
| Expected Error | `No space left on device` |
| Root Cause | Available filesystem space exhausted |
| Remediation | Remove identified disposable test file |
| Verification | Free space restored and writes succeed |
| Data Protection | Preserve important application file |

---

## 2. Production Incident Scenario

### Incident Description

A company runs an application on an Amazon EC2 instance.

The application stores logs, configuration files, and generated data on a separate EBS volume mounted at `/data`.

The application works normally until a large file grows and consumes the available filesystem capacity.

When the application attempts to write new data, Linux returns:

```text
No space left on device
```

The EC2 instance remains running, and AWS reports that the EBS volume is attached with the state `in-use`.

However, the application cannot create new files because the Linux filesystem has reached its capacity limit.

The cloud engineer must determine which filesystem is full, identify the files consuming space, restore available storage, and verify that important application data is preserved.

### Incident Symptoms

- EC2 instance is running.
- EBS volume is attached.
- EBS state remains `in-use`.
- `/data` is mounted.
- Filesystem usage approaches 100%.
- New file creation fails.
- Existing files may remain readable.
- The EC2 root filesystem remains healthy.
- A large disposable file consumes most available capacity.

### Potential Business Impact

- Applications unable to write data.
- Failed log writes.
- Failed report generation.
- Database write failures if data files are affected.
- Application errors or service disruption.
- Increased operational risk.
- Customer-facing failures.

### Objective

Identify the cause of filesystem capacity exhaustion, safely reclaim disk space, and restore application writes without affecting the root filesystem or deleting important data.

---

## 3. AWS Architecture

### Healthy Architecture

```text
                  Amazon EC2
               Amazon Linux 2023
                       |
                       v
                EBS Data Volume
                   gp3 — 5 GiB
                       |
                       v
                Linux NVMe Device
                  /dev/nvme1n1
                       |
                       v
                 ext4 Filesystem
                       |
                       v
                    /data
                       |
                       v
              Application Test Files
                       |
                       v
                Free Space Available
                       |
                       v
                Writes Successful
```

### Broken Architecture

```text
                 Amazon EC2
                      |
                      v
              EBS Volume Attached
                 State: in-use
                      |
                      v
                 ext4 on /data
                      |
                      v
             Large File Created
             lab03-fill.bin
                      |
                      v
              Filesystem Reaches
                  Capacity
                      |
                      v
               No Free Space
                      |
                      v
               Application Write
                      |
                      X
             No Space Left on Device
```

### Recovered Architecture

```text
              EBS Filesystem Full
                       |
                       v
                 Investigate
                df / du / find
                       |
                       v
                Identify Large
                Disposable File
                       |
                       v
                Remove Test File
                       |
                       v
                Free Space Returns
                       |
                       v
               Application Writes
                  Successful
```

### Infrastructure Configuration

| Resource | Configuration |
|---|---|
| VPC | `10.174.0.0/16` |
| Public Subnet | `10.174.1.0/24` |
| Internet Gateway | Attached |
| Route | `0.0.0.0/0` to Internet Gateway |
| Security Group | No inbound rules |
| EC2 Access | AWS Systems Manager |
| Data EBS | 5 GiB gp3 |
| Filesystem | ext4 |
| Mount | `/data` |

---

## 4. Understanding Filesystem Capacity

An Amazon EBS volume provides block storage to an EC2 instance.

Inside Linux, a filesystem manages how data is stored on that block device.

Even when the EBS volume is healthy and attached, a filesystem can become full.

### EBS Volume vs. Linux Filesystem

| Layer | Responsibility |
|---|---|
| Amazon EBS | Provides block storage |
| EC2 | Attaches and accesses the storage |
| Linux Block Device | Exposes the storage to the operating system |
| Filesystem | Organizes files and directories |
| Mount Point | Makes filesystem files accessible |
| Application | Reads and writes data |

### Common Reasons a Filesystem Becomes Full

1. Large application logs.
2. Temporary files.
3. Database growth.
4. Backup files.
5. Generated reports.
6. Application caches.
7. Deleted files still held open by processes.
8. Insufficient filesystem capacity.
9. Inode exhaustion caused by too many small files.

### Important Linux Commands

| Command | Purpose |
|---|---|
| `df -h` | Display filesystem capacity |
| `df -i` | Display inode usage |
| `du -sh` | Display file or directory disk usage |
| `du -xhd1` | Inspect usage within one filesystem |
| `find` | Locate large files |
| `lsblk` | Identify block devices |
| `findmnt` | Verify active mounts |
| `lsof +L1` | Identify open deleted files |
| `iostat` | Investigate disk I/O statistics |

### Understanding ENOSPC

`ENOSPC` means the operating system cannot allocate the requested storage resources.

This commonly occurs because:

- The filesystem has no usable free blocks.
- The filesystem has no remaining free inodes.

In this lab, the fault is caused by exhausted filesystem capacity.

---

## 5. Prerequisites and Safety

### Requirements

- AWS account
- AWS CLI v2
- Ubuntu WSL2
- Bash
- EC2 permissions
- EBS permissions
- VPC permissions
- IAM permissions
- Systems Manager access
- Git and GitHub
- Basic Linux commands

### Safety Requirements

**Important: Never intentionally fill the production root filesystem.**

- Use only a disposable EBS data volume.
- Verify `/data` is mounted before generating files.
- Confirm the correct EBS volume ID before formatting.
- Never run `mkfs` on an existing data volume containing important files.
- Do not create test files under `/` or `/home`.
- Keep important test data separate from the disposable fill file.
- Identify files before removing them.
- Avoid commands such as `rm -rf /data/*`.
- Clean up temporary resources after testing.

### Cost Considerations

EC2, EBS volumes, public IPv4 addresses, and applicable network usage can incur AWS charges.

---

## 6. Prepare the AWS Environment

### Step 1 — Configure Environment Variables

Run in Ubuntu WSL2:

```bash
export AWS_REGION="us-east-1"
export PROJECT_TAG="EBS-Troubleshooting-Lab03"

export INSTANCE_NAME="cloud-ebs-lab03-ec2"
export VOLUME_NAME="cloud-ebs-lab03-data"

export ROLE_NAME="cloud-ebs-lab03-ssm-role"
export PROFILE_NAME="cloud-ebs-lab03-ssm-profile"
export SG_NAME="cloud-ebs-lab03-sg"

export AZ_A="us-east-1a"
export VOLUME_SIZE="5"
```

### Step 2 — Verify AWS Credentials

```bash
aws sts get-caller-identity
aws --version
```

### Step 3 — Check Instance Type Availability

```bash
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters "Name=instance-type,Values=t3.micro" \
  --query 'InstanceTypeOfferings[].Location' \
  --output table \
  --region "$AWS_REGION"
```

Choose a supported Availability Zone.

### Step 4 — Check Existing Lab Resources

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

Avoid creating duplicate resources.

---

## 7. Create EC2 Infrastructure

### Step 1 — Create VPC

```bash
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block "10.174.0.0/16" \
  --tag-specifications \
  "ResourceType=vpc,Tags=[{Key=Name,Value=cloud-ebs-lab03-vpc},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --cidr-block "10.174.1.0/24" \
  --availability-zone "$AZ_A" \
  --tag-specifications \
  "ResourceType=subnet,Tags=[{Key=Name,Value=cloud-ebs-lab03-subnet},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 3 — Create Internet Gateway

```bash
export IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications \
  "ResourceType=internet-gateway,Tags=[{Key=Name,Value=cloud-ebs-lab03-igw},{Key=Project,Value=$PROJECT_TAG}]" \
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
  "ResourceType=route-table,Tags=[{Key=Name,Value=cloud-ebs-lab03-rt},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --region "$AWS_REGION")
```

### Step 5 — Create Security Group

```bash
export SG_ID=$(aws ec2 create-security-group \
  --group-name "$SG_NAME" \
  --description "EBS Lab03 filesystem capacity testing" \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
  "ResourceType=security-group,Tags=[{Key=Name,Value=$SG_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query GroupId \
  --output text \
  --region "$AWS_REGION")
```

No inbound rules are needed for Systems Manager access.

### Step 6 — Create IAM Trust Policy

Create `ec2-trust-policy.json`:

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

### Step 7 — Configure IAM Role

Check whether the role and instance profile already exist.

Create the dedicated resources:

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

Allow IAM propagation before launching EC2.

### Step 8 — Retrieve Amazon Linux 2023 AMI

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
  --instance-type t3.micro \
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

### Step 11 — Retrieve Instance Availability Zone

```bash
export INSTANCE_AZ=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' \
  --output text \
  --region "$AWS_REGION")

echo "$INSTANCE_AZ"
```

---

## 8. Create and Attach EBS Volume

### Step 1 — Create EBS Volume

```bash
export VOLUME_ID=$(aws ec2 create-volume \
  --availability-zone "$INSTANCE_AZ" \
  --size "$VOLUME_SIZE" \
  --volume-type gp3 \
  --tag-specifications \
  "ResourceType=volume,Tags=[{Key=Name,Value=$VOLUME_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'VolumeId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 2 — Wait for Availability

```bash
aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 3 — Attach Volume

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

### Step 4 — Verify Attachment

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{State:State,Size:Size,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Expected: `in-use` with an attachment to the correct EC2 instance.

---

## 9. Prepare Linux Filesystem

### Step 1 — Connect to EC2

Verify Systems Manager status:

```bash
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
  --query 'InstanceInformationList[].{Instance:InstanceId,Status:PingStatus}' \
  --output table \
  --region "$AWS_REGION"
```

Connect:

```bash
aws ssm start-session \
  --target "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

The following Linux commands run inside EC2.

### Step 2 — Identify EBS Device

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,SERIAL
```

An example device might be:

```text
/dev/nvme1n1
```

Identify it using the NVMe serial number or:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

Verify the device corresponds to the dedicated Lab 03 EBS volume.

Set the actual device path:

```bash
export DATA_DEVICE="/dev/nvme1n1"
```

Replace the example path if necessary.

### Step 3 — Check for Existing Filesystem

```bash
lsblk -f

sudo blkid "$DATA_DEVICE" || true

sudo file -s "$DATA_DEVICE"
```

Expected: a new blank volume without a filesystem.

### Step 4 — Create ext4 Filesystem

**Destructive operation: Run only on the verified empty lab data volume.**

```bash
sudo mkfs.ext4 "$DATA_DEVICE"
```

### Step 5 — Mount the Filesystem

```bash
sudo mkdir -p /data

sudo mount "$DATA_DEVICE" /data

sudo chown ec2-user:ec2-user /data
```

Verify:

```bash
findmnt /data

df -hT /data

lsblk -f
```

Expected: a roughly 5 GiB ext4 filesystem mounted at `/data`.

### Step 6 — Create Important Test Data

```bash
echo "Important application configuration - preserve this file" \
  > /data/important-config.txt
```

Verify:

```bash
cat /data/important-config.txt

sha256sum /data/important-config.txt
```

Record the checksum for later verification.

### Step 7 — Establish Healthy Baseline

```bash
df -h /data

df -i /data

du -sh /data

touch /data/write-test.txt
```

Expected: available storage and successful file creation.

---

## 10. Fault Injection — Fill the Filesystem

### Step 1 — Verify Mount Safety

**Critical:** Verify `/data` is the separate 5 GiB EBS filesystem, not a directory on the root disk.

```bash
mountpoint -q /data || {
  echo "ERROR: /data is not mounted"
  exit 1
}
```

Inspect the mount:

```bash
findmnt -n -o SOURCE,FSTYPE,TARGET /data

df -hT /data
```

Confirm the source is the verified Lab 03 data device.

### Step 2 — Generate a Large Test File

```bash
dd if=/dev/zero \
  of=/data/lab03-fill.bin \
  bs=16M \
  count=320 \
  status=progress
```

The command attempts to write 5 GiB of data to the dedicated filesystem.

Because filesystem metadata and reserved blocks consume capacity, the write is expected to fail before completing.

Expected:

```text
dd: error writing '/data/lab03-fill.bin': No space left on device
```

### Step 3 — Verify Disk Usage

```bash
df -hT /data
```

Expected: filesystem usage near 100%, with little or no usable free space.

### Step 4 — Reproduce Application Write Failure

```bash
echo "New application record" > /data/new-record.txt
```

Expected:

```text
No space left on device
```

### Step 5 — Verify Root Filesystem

```bash
df -h /

df -h /data
```

Expected: `/data` is full while the EC2 root filesystem remains healthy.

### Fault Summary

| Check | Expected Finding |
|---|---|
| EC2 | Running |
| EBS | Attached |
| EBS State | In-use |
| `/data` | Mounted |
| Filesystem | ext4 |
| `/data` Usage | Near 100% |
| Application Write | Fails |
| Root Filesystem | Normal |
| Important File | Preserved |
| Root Cause | Large disposable file consumed capacity |

---

## 11. Troubleshooting Investigation

### Check 1 — Inspect Filesystem Capacity

```bash
df -hT
```

Finding: `/data` has no usable free space.

### Check 2 — Check Inode Usage

```bash
df -i /data
```

Finding: inode exhaustion is not expected in this lab.

### Check 3 — Inspect Directory Usage

```bash
sudo du -xhd1 /data
```

Finding: `/data` contains a large amount of allocated data.

### Check 4 — Identify Large Files

```bash
sudo du -ahx /data | sort -h | tail -20
```

Expected: `lab03-fill.bin` appears among the largest entries.

### Check 5 — Search for Large Files

```bash
sudo find /data -xdev -type f -size +100M -ls
```

Expected: the disposable fill file is identified.

### Check 6 — Inspect File Size

```bash
ls -lh /data/lab03-fill.bin

du -h /data/lab03-fill.bin
```

Finding: the fill file consumes most of the available space.

### Check 7 — Verify Important File

```bash
cat /data/important-config.txt

sha256sum /data/important-config.txt
```

Finding: the original application data remains readable.

### Check 8 — Verify AWS Volume Size

From a separate WSL2 terminal:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,State:State,Size:Size,Type:VolumeType}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
State: in-use
Size: 5 GiB
Type: gp3
```

### Investigation Summary

| Investigation | Finding |
|---|---|
| EC2 | Running |
| EBS Attachment | Correct |
| Volume Size | 5 GiB |
| Filesystem | ext4 |
| Mount | `/data` |
| Free Space | Exhausted |
| Inode Usage | Not exhausted |
| Largest File | `lab03-fill.bin` |
| Root Filesystem | Healthy |
| Root Cause | Large test file consumed free space |

---

## 12. Root Cause Analysis

### Root Cause

The 5 GiB EBS-backed ext4 filesystem reached its usable storage capacity after a large test file consumed almost all available space.

When the application attempted to create another file, Linux returned `ENOSPC`.

The EBS volume itself was still attached and functioning properly.

### Technical Explanation

The filesystem maintains a limited number of data blocks.

When those blocks are exhausted, the operating system cannot allocate additional space for new application data.

The application fails to write even though EC2 remains running and the EBS volume state remains `in-use`.

### Root Cause Flow

```text
EBS Data Volume Attached
           |
           v
Filesystem Mounted at /data
           |
           v
Large Disposable File Created
           |
           v
Free Space Exhausted
           |
           v
Application Attempts New Write
           |
           v
ENOSPC: No Space Left on Device
```

### Five Whys

1. **Why did the application write fail?** The filesystem had no usable free space.
2. **Why was the filesystem full?** A large file consumed the available capacity.
3. **Why did the file grow beyond available space?** The write process did not stop before capacity was exhausted.
4. **Why wasn't the issue detected earlier?** No filesystem usage threshold prevented the test from reaching capacity.
5. **How can this be prevented?** Monitor disk usage, manage file retention, and provision sufficient capacity.

---

## 13. Remediation — Recover Free Space

### Step 1 — Identify the Disposable File

```bash
ls -lh /data/lab03-fill.bin
```

Verify that this is the file created during the lab.

### Step 2 — Remove Only the Test File

```bash
rm -- /data/lab03-fill.bin
```

Do not remove unknown application files.

### Step 3 — Verify Free Space

```bash
df -hT /data
```

Expected: filesystem usage drops significantly.

### Step 4 — Verify New File Creation

```bash
echo "Application write successful after recovery" \
  > /data/new-record.txt
```

Verify:

```bash
cat /data/new-record.txt
```

Expected:

```text
Application write successful after recovery
```

### Step 5 — Verify Original Data Integrity

```bash
cat /data/important-config.txt

sha256sum /data/important-config.txt
```

Compare the checksum against the original value.

Expected: the original file remains unchanged.

### Step 6 — Check for Deleted Open Files

If free space is not restored after deleting a file, investigate whether an application still holds it open:

```bash
sudo lsof +L1
```

Install `lsof` if necessary.

A deleted file may continue consuming disk space until the process closes it.

Do not restart production services without checking their impact.

### Remediation Summary

| Check | During Fault | After Remediation |
|---|---|---|
| EBS State | In-use | In-use |
| Filesystem | ext4 | ext4 |
| Disk Usage | Near 100% | Reduced |
| Free Space | Exhausted | Available |
| Application Writes | Failed | Successful |
| Original File | Present | Present |
| Data Integrity | Preserved | Verified |

---

## 14. Additional Production Troubleshooting

### Investigate Disk Usage

```bash
df -hT

df -i

sudo du -xhd1 /data
```

### Identify the 10 Largest Files

```bash
sudo find /data -xdev -type f -printf '%s %p\n' \
  | sort -nr \
  | head -10 \
  | numfmt --field=1 --to=iec
```

### Simple Disk Usage Warning

```bash
df -P /data | awk 'NR==2 {
  usage=$5
  gsub("%","",usage)

  if (usage >= 80)
    print "WARNING: /data usage is " usage "%"
  else
    print "OK: /data usage is " usage "%"
}'
```

### Common Disk-Full Problems

| Problem | Investigation | Typical Remediation |
|---|---|---|
| Large application logs | `du`, `find` | Configure appropriate log retention |
| Temporary files | `find`, `du` | Remove verified disposable files |
| Inode exhaustion | `df -i` | Investigate excessive small files |
| Deleted open files | `lsof +L1` | Close responsible file descriptors safely |
| Wrong mount point | `findmnt`, `df` | Correct filesystem mount |
| Insufficient volume capacity | AWS CLI, `lsblk` | Expand EBS and filesystem |
| Rapid data growth | Monitoring and application logs | Address growth and capacity planning |

### CloudWatch Monitoring

Amazon EBS provides volume-level metrics, but those metrics do not directly report the percentage of free space inside a Linux filesystem.

For filesystem usage monitoring on EC2, use the Amazon CloudWatch Agent to collect disk utilization metrics.

Useful alarms include:

- Disk utilization above 80%.
- Disk utilization above 90%.
- Low available storage.
- Unexpected rapid storage growth.

---

## 15. Production Best Practices

### Filesystem Troubleshooting Checklist

1. Confirm the affected application and filesystem.
2. Inspect filesystem usage with `df -h`.
3. Check inode availability with `df -i`.
4. Identify the mount point using `findmnt`.
5. Locate large files and directories.
6. Review application logs and temporary files.
7. Determine which data is safe to remove.
8. Preserve important application data.
9. Reclaim space safely.
10. Verify successful application writes.
11. Monitor capacity to prevent recurrence.

### Prevention

- Monitor filesystem utilization.
- Configure storage alerts.
- Apply appropriate log retention.
- Avoid storing unlimited temporary files.
- Use dedicated data volumes.
- Maintain application backups.
- Use EBS snapshots where appropriate.
- Review storage growth trends.
- Expand volume capacity when necessary.
- Document recovery procedures.

### Security Considerations

- Avoid unnecessary public SSH access.
- Use Systems Manager for administrative access.
- Apply least-privilege IAM.
- Restrict access to sensitive application files.
- Avoid broad deletion commands.
- Verify the correct EBS volume before destructive operations.

---

## 16. Cleanup

**Warning:** These commands delete the disposable EBS data volume, terminate the lab EC2 instance, and remove its dedicated infrastructure.

### Step 1 — Unmount the Data Filesystem

Inside EC2:

```bash
cd /

findmnt /data

sudo fuser -vm /data
```

After confirming no required process is using the filesystem:

```bash
sudo umount /data
```

Verify:

```bash
findmnt /data
```

Exit Systems Manager:

```bash
exit
```

### Step 2 — Verify AWS Resource Ownership

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

Confirm the project tag is `EBS-Troubleshooting-Lab03`.

### Step 3 — Detach the Data Volume

```bash
aws ec2 detach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 4 — Delete the Disposable Volume

```bash
aws ec2 delete-volume \
  --volume-id "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 5 — Verify Root EBS Deletion Settings

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].{RootDevice:RootDeviceName,BlockDevices:BlockDeviceMappings}' \
  --output json \
  --region "$AWS_REGION"
```

Confirm the root EBS volume is configured to be deleted when the instance terminates.

### Step 6 — Terminate EC2

```bash
aws ec2 terminate-instances \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait instance-terminated \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

### Step 7 — Delete IAM Resources

Verify these are dedicated Lab 03 resources:

```bash
aws iam list-role-tags \
  --role-name "$ROLE_NAME"

aws iam get-instance-profile \
  --instance-profile-name "$PROFILE_NAME"
```

Delete:

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

### Step 8 — Verify Network Interfaces

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'NetworkInterfaces[].{ID:NetworkInterfaceId,Status:Status}' \
  --output table \
  --region "$AWS_REGION"
```

Wait until the lab's network interfaces are removed.

### Step 9 — Delete Networking

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

### Step 10 — Verify Cleanup

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

Expected: empty arrays after cleanup finishes.

Also verify that the instance, root EBS volume, IAM role, and instance profile are removed.

### Cleanup Summary

| Resource | Expected Result |
|---|---|
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

## 17. Interview Questions and Answers

### Q1. What causes `No space left on device` in Linux?

The filesystem may have no free data blocks or no available inodes.

### Q2. What does `df -h` show?

It displays filesystem capacity, used space, available space, and usage percentage.

### Q3. What does `du -sh` show?

It shows the disk space used by a file or directory in human-readable format.

### Q4. What is the difference between `df` and `du`?

`df` reports filesystem-level usage. `du` measures the space occupied by files and directories.

### Q5. What does `df -i` show?

It reports filesystem inode usage and availability.

### Q6. Can an EBS volume remain `in-use` when the filesystem is full?

Yes. EBS attachment state is independent of free space within the Linux filesystem.

### Q7. Why might deleting a file not immediately free disk space?

A process may still hold the deleted file open.

### Q8. How do you find large files in Linux?

Use `find`, `du`, `sort`, and `ls -lh`.

### Q9. Should you delete every file when a filesystem becomes full?

No. Identify the cause and remove only files that are safe to delete.

### Q10. How can filesystem usage be monitored in AWS?

Use the CloudWatch Agent to collect Linux disk usage metrics and configure alarms.

### Q11. What if deleting temporary files is insufficient?

Consider expanding the EBS volume and then growing the partition or filesystem as required.

### Q12. How would you explain this lab in an interview?

I deployed an EC2 instance and attached a separate 5 GiB EBS data volume.

I created an ext4 filesystem and mounted it at `/data`.

After creating an important application test file, I intentionally filled the filesystem using `dd`.

New application writes failed with `No space left on device`.

I investigated filesystem capacity, inode usage, and large files using `df`, `du`, and `find`.

After identifying the disposable test file, I removed it safely, confirmed that free space returned, and verified successful application writes.

I also checked that the original application file remained unchanged.

---

## 18. Repository Structure

```text
03-filesystem-full/
├── README.md
└── ec2-trust-policy.json
```

### File Descriptions

| File | Purpose |
|---|---|
| `README.md` | Complete filesystem-full troubleshooting documentation |
| `ec2-trust-policy.json` | IAM trust policy for Systems Manager access |

Do not commit credentials, private keys, or sensitive data.

---

## 19. Lessons Learned

This lab demonstrates that an attached and healthy Amazon EBS volume can still experience application failures when its Linux filesystem becomes full.

The incident was not caused by an EBS attachment problem or EC2 infrastructure failure.

Instead, a large file consumed nearly all available filesystem capacity.

Using `df`, `du`, `find`, and `df -i`, we identified the affected filesystem and determined the reason new writes were failing.

We safely removed the disposable file, restored free space, and verified that application writes worked again.

We also learned the importance of protecting the root filesystem when testing storage incidents and preserving important files during cleanup.

**Key Lesson:** When an application reports `No space left on device`, investigate Linux filesystem capacity and inode usage before assuming there is an AWS EBS hardware or attachment problem.

---

## 20. Skills Demonstrated

- Amazon EBS provisioning
- EBS attachment
- EC2 storage administration
- Linux ext4 filesystem management
- Filesystem capacity troubleshooting
- Disk usage analysis
- Inode usage investigation
- `df`
- `du`
- `find`
- `lsblk`
- `findmnt`
- `dd`
- Safe file cleanup
- Data integrity verification
- AWS CLI v2
- AWS Systems Manager
- Root cause analysis
- Storage monitoring fundamentals
- AWS infrastructure cleanup

## Final Outcome

| Item | Expected Result |
|---|---|
| Incident | INC-EBS-003 |
| Problem | EBS-backed filesystem full |
| Root Cause | Large disposable file consumes space |
| EC2 | Running |
| EBS | In-use |
| Filesystem | ext4 |
| Mount | `/data` |
| Fault Result | No space left on device |
| Investigation | `df`, `du`, `find`, `df -i` |
| Remediation | Remove disposable test file |
| Final Disk Usage | Reduced |
| Application Writes | Successful |
| Important Data | Preserved |
| Cleanup | Temporary resources removed |
| Expected Incident Status | Resolved |

**Key Takeaway:** Amazon EBS provides storage capacity, but Linux manages filesystem usage. Effective troubleshooting requires checking both AWS volume information and in-guest filesystem utilization before making changes.
