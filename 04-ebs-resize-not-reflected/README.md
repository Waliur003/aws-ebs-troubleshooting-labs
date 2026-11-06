# Lab 04 — EBS Resize Not Reflected in Linux

## Overview

This lab demonstrates how to troubleshoot an Amazon EBS volume whose capacity has been increased in AWS but whose Linux filesystem still reports its original size.

We deploy an Amazon Linux 2023 EC2 instance with a separate 5 GiB `gp3` EBS data volume. We create an ext4 filesystem directly on the EBS block device and mount it at `/data`.

After creating an application test file, we take a recovery snapshot and increase the volume's capacity from 5 GiB to 10 GiB using AWS CLI.

AWS reports the new size, and Linux recognizes the larger block device. However, the mounted ext4 filesystem continues reporting approximately 5 GiB.

We investigate the mismatch using AWS CLI, `lsblk`, `df`, `blkid`, `blockdev`, and `tune2fs`.

Once the root cause is identified, we expand the existing ext4 filesystem using `resize2fs`, verify the additional capacity, and confirm that the original application data remains unchanged.

Finally, we safely delete the temporary EBS volume, snapshot, EC2 instance, and dedicated AWS infrastructure.

**Troubleshooting workflow:**

Provision → Healthy Baseline → Snapshot → Fault Injection → Investigation → Root Cause Analysis → Remediation → Verification → Cleanup

---

## 1. Lab Information

| Item | Configuration |
|---|---|
| Incident ID | INC-EBS-004 |
| Difficulty | Intermediate |
| AWS Region | `us-east-1` |
| AWS Services | EC2, EBS, EBS Snapshots, VPC, IAM, Systems Manager |
| Operating System | Amazon Linux 2023 |
| Instance Type | `t3.micro` |
| EBS Volume Type | `gp3` |
| Initial EBS Size | 5 GiB |
| Expanded EBS Size | 10 GiB |
| AWS Attachment | `/dev/sdf` |
| Linux Block Device | NVMe device such as `/dev/nvme1n1` |
| Filesystem | ext4 |
| Partition Table | None on the data volume |
| Mount Point | `/data` |
| Fault Injection | Expand EBS without growing ext4 |
| Primary Symptom | Linux filesystem still shows approximately 5 GiB |
| Root Cause | ext4 filesystem not expanded |
| Remediation | `resize2fs` |
| Data Protection | Pre-resize EBS snapshot |
| Verification | New filesystem capacity and unchanged file checksum |

---

## 2. Production Incident Scenario

### Incident Description

A company runs an application on Amazon EC2 that stores its files on an EBS data volume mounted at `/data`.

The original EBS volume has a capacity of 5 GiB.

As the application grows, the available storage becomes insufficient. A cloud engineer increases the volume capacity from 5 GiB to 10 GiB.

AWS accepts the modification and reports the new provisioned size.

However, Linux still reports approximately 5 GiB of filesystem capacity when the engineer runs:

```bash
df -hT /data
```

The application can still access its original files, but it cannot use the additional provisioned storage capacity.

The engineer investigates the EBS volume, Linux block device, filesystem type, and partition structure.

The investigation reveals that the EBS block device was expanded successfully but the ext4 filesystem was not.

The engineer must extend the filesystem without formatting the volume or losing application data.

### Incident Symptoms

- EC2 instance is running.
- EBS volume remains attached.
- AWS reports 10 GiB of EBS capacity.
- EBS modification reaches `optimizing` or `completed`.
- Linux recognizes a 10 GiB block device.
- Mounted ext4 filesystem still reports approximately 5 GiB.
- Original application files remain accessible.
- Newly provisioned space is not yet usable by the filesystem.

### Potential Business Impact

- Application storage capacity remains constrained.
- Engineers may believe the AWS resize failed.
- Applications may continue encountering low-space conditions.
- Storage upgrades may not deliver the expected benefit.
- Operational incidents may take longer to resolve.
- Unnecessary additional volumes may be provisioned.

### Objective

Identify the difference between the provisioned EBS volume capacity and usable Linux filesystem capacity, expand the filesystem safely, and verify data integrity.

---

## 3. AWS Architecture

### Initial Architecture

```text
                    Amazon EC2
                 Amazon Linux 2023
                          |
                          v
                   EBS gp3 Volume
                       5 GiB
                          |
                          v
                  Linux NVMe Device
                       5 GiB
                          |
                          v
                   ext4 Filesystem
                       ~5 GiB
                          |
                          v
                      /data
                          |
                          v
                 Application Data
```

### Broken Architecture

```text
                     AWS EBS
                 Volume Resized
                  5 GiB -> 10 GiB
                          |
                          v
                  Linux Block Device
                       10 GiB
                          |
                          X
              Filesystem Not Expanded
                          |
                          v
                  ext4 Filesystem
                       ~5 GiB
                          |
                          v
                       /data
                          |
                          v
              Extra Space Not Available
```

### Recovered Architecture

```text
                  AWS EBS Volume
                       10 GiB
                          |
                          v
                  Linux Block Device
                       10 GiB
                          |
                          v
                  sudo resize2fs
                          |
                          v
                 ext4 Filesystem
                       ~10 GiB
                          |
                          v
                       /data
                          |
                          v
              Extra Capacity Available
                          |
                          v
               Original Data Preserved
```

### Resource Configuration

| Resource | Configuration |
|---|---|
| VPC | `10.175.0.0/16` |
| Public Subnet | `10.175.1.0/24` |
| Internet Gateway | Attached |
| Public Route | `0.0.0.0/0` through IGW |
| EC2 Security Group | No inbound rules |
| EC2 Access | Systems Manager |
| Initial Data Volume | 5 GiB gp3 |
| Expanded Data Volume | 10 GiB gp3 |
| Filesystem | ext4 directly on disk |
| Mount | `/data` |
| Recovery Backup | EBS snapshot |

---

## 4. Understanding EBS Volume Resizing

Amazon EBS provides block storage for EC2 instances.

An EBS volume has a provisioned capacity, while Linux filesystems maintain their own structures describing how much storage can be used.

Expanding the EBS volume does not automatically expand all layers above it.

### Storage Layers

| Layer | Purpose |
|---|---|
| Amazon EBS | Provides block storage capacity |
| EC2 Attachment | Connects the EBS volume to the instance |
| Linux Block Device | Represents the available disk capacity |
| Partition | Divides a disk into logical regions, if used |
| Filesystem | Organizes files and manages usable space |
| Mount Point | Exposes a filesystem to applications |

### EBS Size vs. Filesystem Size

Before modification:

```text
EBS Volume:       5 GiB
Linux Disk:       5 GiB
ext4 Filesystem: ~5 GiB
```

After EBS modification but before filesystem expansion:

```text
EBS Volume:      10 GiB
Linux Disk:      10 GiB
ext4 Filesystem: ~5 GiB
```

After remediation:

```text
EBS Volume:      10 GiB
Linux Disk:      10 GiB
ext4 Filesystem: ~10 GiB
```

The exact filesystem capacity is slightly lower because of ext4 metadata and reserved space.

### Important Commands

| Command | Purpose |
|---|---|
| `aws ec2 describe-volumes` | Check AWS EBS capacity |
| `aws ec2 modify-volume` | Modify EBS capacity |
| `aws ec2 describe-volumes-modifications` | Check modification progress |
| `lsblk` | Inspect Linux block devices and partitions |
| `df -hT` | Inspect mounted filesystem capacity |
| `blkid` | Identify filesystem type and UUID |
| `blockdev --getsize64` | Display raw block device size |
| `tune2fs -l` | Inspect ext4 metadata |
| `resize2fs` | Extend an ext4 filesystem |
| `growpart` | Expand a partition when required |
| `xfs_growfs` | Extend a mounted XFS filesystem |

### EBS Modification States

| State | Meaning |
|---|---|
| `modifying` | The volume modification is underway |
| `optimizing` | New settings are available while optimization continues |
| `completed` | The modification has completed |
| `failed` | The requested modification did not succeed |

Wait until the modification reaches `optimizing` or `completed` before extending the filesystem.

---

## 5. Prerequisites and Safety

### Requirements

- AWS account
- AWS CLI v2
- Ubuntu WSL2
- Bash
- Amazon EC2 permissions
- Amazon EBS permissions
- Snapshot permissions
- VPC permissions
- IAM permissions
- AWS Systems Manager
- Git and GitHub

### Safety Requirements

**Important: Resizing and filesystem commands must target the correct storage device.**

- Use a dedicated EBS data volume.
- Never modify the EC2 root filesystem accidentally.
- Verify the EBS volume ID.
- Verify the Linux NVMe device using its serial number.
- Take a snapshot before resizing.
- Check the filesystem type before running a resize command.
- Never use `mkfs` to expand an existing filesystem.
- Do not run `growpart` unless a partition actually exists.
- Record and compare checksums for important data.
- Verify resource ownership before cleanup.

### Cost Considerations

EBS volumes, snapshots, EC2 instances, public IPv4 addresses, and related resources may incur AWS charges.

Increasing the volume from 5 GiB to 10 GiB increases provisioned storage capacity and associated charges.

---

## 6. Prepare the AWS Environment

### Step 1 — Configure Environment Variables

Run in Ubuntu WSL2:

```bash
export AWS_REGION="us-east-1"
export PROJECT_TAG="EBS-Troubleshooting-Lab04"

export INSTANCE_NAME="cloud-ebs-lab04-ec2"
export VOLUME_NAME="cloud-ebs-lab04-data"

export ROLE_NAME="cloud-ebs-lab04-ssm-role"
export PROFILE_NAME="cloud-ebs-lab04-ssm-profile"
export SG_NAME="cloud-ebs-lab04-sg"

export AZ_A="us-east-1a"
export INITIAL_SIZE="5"
export NEW_SIZE="10"
```

### Step 2 — Verify AWS Identity

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

Select a supported Availability Zone if necessary.

### Step 4 — Check Existing Resources

```bash
aws ec2 describe-instances \
  --filters \
    "Name=tag:Project,Values=$PROJECT_TAG" \
    "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query 'Reservations[].Instances[].{Instance:InstanceId,State:State.Name}' \
  --output table \
  --region "$AWS_REGION"

aws ec2 describe-volumes \
  --filters "Name=tag:Project,Values=$PROJECT_TAG" \
  --query 'Volumes[].{Volume:VolumeId,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

Avoid duplicate infrastructure.

---

## 7. Create EC2 Infrastructure

### Step 1 — Create VPC

```bash
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block "10.175.0.0/16" \
  --tag-specifications \
  "ResourceType=vpc,Tags=[{Key=Name,Value=cloud-ebs-lab04-vpc},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --cidr-block "10.175.1.0/24" \
  --availability-zone "$AZ_A" \
  --tag-specifications \
  "ResourceType=subnet,Tags=[{Key=Name,Value=cloud-ebs-lab04-subnet},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 3 — Create Internet Gateway

```bash
export IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications \
  "ResourceType=internet-gateway,Tags=[{Key=Name,Value=cloud-ebs-lab04-igw},{Key=Project,Value=$PROJECT_TAG}]" \
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
  "ResourceType=route-table,Tags=[{Key=Name,Value=cloud-ebs-lab04-rt},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --description "Lab04 EBS filesystem resize" \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
  "ResourceType=security-group,Tags=[{Key=Name,Value=$SG_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query GroupId \
  --output text \
  --region "$AWS_REGION")
```

No inbound permissions are required because Systems Manager provides remote access.

### Step 6 — Create IAM Trust Policy

Save the following as `ec2-trust-policy.json`:

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

### Step 7 — Create IAM Role and Profile

Verify that the intended names are unused before creating them.

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

### Step 11 — Identify Instance AZ

```bash
export INSTANCE_AZ=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' \
  --output text \
  --region "$AWS_REGION")

echo "$INSTANCE_AZ"
```

---

## 8. Create and Attach the EBS Volume

### Step 1 — Create Initial 5 GiB Volume

```bash
export VOLUME_ID=$(aws ec2 create-volume \
  --availability-zone "$INSTANCE_AZ" \
  --size "$INITIAL_SIZE" \
  --volume-type gp3 \
  --tag-specifications \
  "ResourceType=volume,Tags=[{Key=Name,Value=$VOLUME_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'VolumeId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 2 — Wait for Volume Availability

```bash
aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 3 — Verify Volume

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[].{ID:VolumeId,Size:Size,State:State,AZ:AvailabilityZone}' \
  --output table \
  --region "$AWS_REGION"
```

Expected: 5 GiB, `available`, and in the same AZ as EC2.

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

Verify:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{State:State,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Expected: `in-use` and attached to the intended instance.

---

## 9. Create the ext4 Filesystem

### Step 1 — Connect Through Systems Manager

```bash
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
  --query 'InstanceInformationList[].{Instance:InstanceId,Status:PingStatus}' \
  --output table \
  --region "$AWS_REGION"
```

Connect once the instance is online:

```bash
aws ssm start-session \
  --target "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

The following Linux commands run inside the EC2 instance.

### Step 2 — Identify the EBS Device

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,SERIAL
```

Possible output:

```text
NAME          SIZE TYPE FSTYPE MOUNTPOINTS
nvme0n1         8G disk
nvme0n1p1       8G part xfs    /
nvme1n1         5G disk
```

Identify the correct EBS volume using the NVMe serial or:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

After verification:

```bash
export DATA_DEVICE="/dev/nvme1n1"
```

Replace the example device with your actual lab volume.

**Never select the EC2 root disk.**

### Step 3 — Check Existing Filesystem Information

```bash
lsblk -f

sudo blkid "$DATA_DEVICE" || true

sudo file -s "$DATA_DEVICE"
```

Confirm it is the new blank data disk.

### Step 4 — Create ext4

**Destructive command: Only run on the verified empty Lab 04 volume.**

```bash
sudo mkfs.ext4 "$DATA_DEVICE"
```

This lab deliberately uses an ext4 filesystem directly on the whole disk, without partitions.

### Step 5 — Mount at `/data`

```bash
sudo mkdir -p /data

sudo mount "$DATA_DEVICE" /data
```

Verify:

```bash
findmnt /data

df -hT /data

lsblk -f
```

Expected: approximately 5 GiB of ext4 capacity mounted at `/data`.

### Step 6 — Create Important Application Data

```bash
echo "EBS Lab 04 - Important Data - Do Not Lose" \
  | sudo tee /data/important-data.txt
```

Verify:

```bash
cat /data/important-data.txt

sha256sum /data/important-data.txt
```

Record the original checksum.

### Step 7 — Establish Healthy Baseline

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS

df -hT /data

findmnt /data
```

Expected:

| Check | Result |
|---|---|
| EBS Size | 5 GiB |
| Block Device | 5 GiB |
| Filesystem | Approximately 5 GiB |
| Mount | `/data` |
| Application Data | Accessible |

---

## 10. Create a Pre-Resize Snapshot

Before changing storage capacity, create a recovery snapshot.

### Step 1 — Flush Filesystem Writes

Inside EC2:

```bash
sync
```

For production databases, coordinate application-consistent backups. A filesystem flush alone does not guarantee a database-consistent snapshot.

Exit EC2:

```bash
exit
```

### Step 2 — Create Snapshot

From Ubuntu WSL2:

```bash
export SNAPSHOT_ID=$(aws ec2 create-snapshot \
  --volume-id "$VOLUME_ID" \
  --description "Lab04 pre-resize snapshot" \
  --tag-specifications \
  "ResourceType=snapshot,Tags=[{Key=Name,Value=cloud-ebs-lab04-before-resize},{Key=Project,Value=$PROJECT_TAG}]" \
  --query SnapshotId \
  --output text \
  --region "$AWS_REGION")

echo "$SNAPSHOT_ID"
```

### Step 3 — Wait for Snapshot Completion

```bash
aws ec2 wait snapshot-completed \
  --snapshot-ids "$SNAPSHOT_ID" \
  --region "$AWS_REGION"
```

### Step 4 — Verify Snapshot

```bash
aws ec2 describe-snapshots \
  --snapshot-ids "$SNAPSHOT_ID" \
  --query 'Snapshots[].{ID:SnapshotId,State:State,VolumeSize:VolumeSize}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
State: completed
Volume Size: 5 GiB
```

The snapshot provides a recovery point in case the resize or filesystem modification encounters problems.

---

## 11. Fault Injection — EBS Resize Without Filesystem Expansion

### Step 1 — Verify Original Volume Size

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,Size:Size,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

Expected: 5 GiB.

### Step 2 — Increase EBS to 10 GiB

```bash
aws ec2 modify-volume \
  --volume-id "$VOLUME_ID" \
  --size "$NEW_SIZE" \
  --region "$AWS_REGION"
```

This is the intentional fault injection.

We enlarge the EBS volume but do not yet expand its ext4 filesystem.

### Step 3 — Inspect Volume Modification

```bash
aws ec2 describe-volumes-modifications \
  --volume-ids "$VOLUME_ID" \
  --query 'VolumesModifications[].{Volume:VolumeId,Original:OriginalSize,Target:TargetSize,State:ModificationState,Progress:Progress}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
Original: 5
Target: 10
State: optimizing or completed
```

Do not proceed with filesystem expansion while the modification remains in the `modifying` state.

### Step 4 — Verify AWS EBS Capacity

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,Size:Size,State:State,Type:VolumeType}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
Size: 10
State: in-use
Type: gp3
```

### Step 5 — Reconnect to EC2

```bash
aws ssm start-session \
  --target "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

### Step 6 — Inspect Linux Disk Size

Inside EC2:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
```

Expected once the new block device capacity is recognized:

```text
nvme1n1  10G  ext4  /data
```

### Step 7 — Inspect Filesystem Size

```bash
df -hT /data
```

Expected: approximately 5 GiB.

### Step 8 — Compare the Device and Filesystem

```bash
sudo blockdev --getsize64 "$DATA_DEVICE"

df -B1 /data
```

The raw block device should report approximately 10 GiB, while ext4 reports its original capacity.

### Fault Summary

| Check | Finding |
|---|---|
| EC2 | Running |
| EBS Attachment | Correct |
| EBS Size | 10 GiB |
| Linux Block Device | 10 GiB |
| ext4 Filesystem | Approximately 5 GiB |
| Mount | `/data` |
| Application Data | Accessible |
| Additional Capacity | Not yet usable |
| Root Cause | Filesystem not expanded |

---

## 12. Troubleshooting Investigation

### Check 1 — Inspect AWS Volume Capacity

From WSL2:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,Size:Size,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

Finding: EBS reports 10 GiB.

### Check 2 — Inspect Modification State

```bash
aws ec2 describe-volumes-modifications \
  --volume-ids "$VOLUME_ID" \
  --query 'VolumesModifications[].{State:ModificationState,Progress:Progress}' \
  --output table \
  --region "$AWS_REGION"
```

Finding: the modification has reached `optimizing` or `completed`.

### Check 3 — Inspect Block Device

Inside EC2:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Finding: data disk is approximately 10 GiB.

### Check 4 — Inspect Filesystem Capacity

```bash
df -hT /data
```

Finding: ext4 filesystem still reports approximately 5 GiB.

### Check 5 — Inspect Filesystem Type

```bash
sudo blkid "$DATA_DEVICE"
```

Finding: the filesystem is ext4.

### Check 6 — Inspect Partition Layout

```bash
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS
```

Finding: the data filesystem is created directly on the disk, without a partition table.

Therefore, this lab does not require `growpart`.

### Check 7 — Inspect ext4 Metadata

```bash
sudo tune2fs -l "$DATA_DEVICE" \
  | grep -E 'Block count|Block size|Filesystem state'
```

Finding: ext4 metadata still reflects the original filesystem capacity.

### Check 8 — Verify Important Data

```bash
cat /data/important-data.txt

sha256sum /data/important-data.txt
```

Finding: the original application file remains accessible.

### Investigation Summary

| Component | Finding |
|---|---|
| AWS EBS | 10 GiB |
| EBS Modification | Optimizing or completed |
| Linux Disk | 10 GiB |
| Partition | None |
| Filesystem Type | ext4 |
| Filesystem Size | Approximately 5 GiB |
| Data | Preserved |
| Root Cause | ext4 not extended |

---

## 13. Root Cause Analysis

### Root Cause

The Amazon EBS volume was successfully expanded from 5 GiB to 10 GiB.

However, the existing ext4 filesystem still used its original filesystem metadata and block count.

The filesystem had not been extended to use the additional available space on the enlarged block device.

### Technical Explanation

EBS controls the underlying provisioned disk capacity.

Linux recognizes the larger block device after the EBS modification becomes effective.

But the filesystem maintains its own capacity metadata.

Until that metadata is updated, applications cannot use the extra blocks.

### Root Cause Flow

```text
EBS Volume: 5 GiB
        |
        v
AWS ModifyVolume
        |
        v
EBS Volume: 10 GiB
        |
        v
Linux Device: 10 GiB
        |
        X
ext4 Filesystem Still ~5 GiB
        |
        v
Additional Capacity Not Usable
```

### Five Whys

1. **Why does `/data` still show 5 GiB?** The ext4 filesystem has not been expanded.
2. **Why didn't ext4 expand automatically?** EBS volume modification and Linux filesystem expansion are separate operations.
3. **Why does `lsblk` show 10 GiB?** Linux recognizes the larger block device.
4. **Why are the original files still present?** The filesystem remains intact and mounted.
5. **How is the issue fixed?** Extend the existing filesystem using the correct filesystem-specific command.

### Prevention

Use a complete storage expansion procedure that verifies all relevant layers:

```text
AWS EBS Capacity
        |
        v
Linux Block Device Capacity
        |
        v
Partition Capacity (if present)
        |
        v
Filesystem Capacity
        |
        v
Application Verification
```

---

## 14. Remediation — Expand the ext4 Filesystem

### Step 1 — Verify the Correct Device

Inside EC2:

```bash
echo "$DATA_DEVICE"

lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,SERIAL

findmnt /data
```

Confirm that the selected block device is the dedicated Lab 04 EBS volume.

Verify the disk reports approximately 10 GiB and the filesystem is ext4.

### Step 2 — Extend ext4

```bash
sudo resize2fs "$DATA_DEVICE"
```

Because this filesystem occupies the entire block device, partition expansion is unnecessary.

The filesystem can be extended online while mounted.

**Do not run `mkfs.ext4` again.** Formatting the device is not part of the remediation.

### Step 3 — Verify Filesystem Capacity

```bash
df -hT /data
```

Expected: approximately 10 GiB.

### Step 4 — Verify Device Layout

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
```

Expected: 10 GiB data disk mounted at `/data`.

### Step 5 — Verify Original Data

```bash
cat /data/important-data.txt

sha256sum /data/important-data.txt
```

The checksum should match the original value recorded before resizing.

### Step 6 — Verify New File Writes

```bash
echo "Filesystem successfully expanded to 10 GiB" \
  | sudo tee /data/resize-test.txt
```

Read it:

```bash
cat /data/resize-test.txt
```

### Step 7 — Check System Messages

```bash
sudo dmesg | tail -30
```

Investigate any unexpected filesystem or block device errors.

### Remediation Summary

| Check | During Incident | After Remediation |
|---|---|---|
| EBS Size | 10 GiB | 10 GiB |
| Linux Disk | 10 GiB | 10 GiB |
| ext4 Filesystem | ~5 GiB | ~10 GiB |
| Mount Point | `/data` | `/data` |
| Original Data | Present | Present |
| Data Integrity | Preserved | Verified |
| New Capacity | Unavailable | Available |
| Incident | Active | Resolved |

---

## 15. Resizing Partitioned Volumes and XFS

This lab deliberately uses ext4 directly on an unpartitioned data disk.

Many production volumes use partitions or XFS.

### Unpartitioned ext4

```text
/dev/nvme1n1
    |
    └── ext4 → /data
```

Example remediation:

```bash
sudo resize2fs /dev/nvme1n1
```

### Partitioned ext4

```text
/dev/nvme1n1
    |
    └── /dev/nvme1n1p1
              |
              └── ext4 → /data
```

After verifying the correct disk and partition, an example procedure is:

```bash
sudo growpart /dev/nvme1n1 1

sudo resize2fs /dev/nvme1n1p1
```

`growpart` extends the partition.

`resize2fs` extends the filesystem within the larger partition.

**Do not run `growpart` in the unpartitioned Lab 04 environment.**

### XFS

For a mounted XFS filesystem, an example command is:

```bash
sudo xfs_growfs /data
```

Use the correct tool for the filesystem type.

### Common Resize Problems

| Problem | Possible Cause | Investigation |
|---|---|---|
| Linux disk still reports old size | EBS modification not ready | Inspect modification state |
| `df` reports old size | Filesystem not expanded | Run the appropriate grow command |
| `resize2fs` reports bad magic number | Wrong device or filesystem type | Check `blkid` |
| `resize2fs` has nothing to do | Already expanded or device still small | Compare disk and filesystem |
| `growpart` cannot find partition | No partition exists | Inspect `lsblk` |
| XFS resize fails | Wrong command or mount state | Check `df -T` and `findmnt` |
| Application data unavailable | Incorrect mount or data issue | Stop and investigate before changing storage |

---

## 16. Final Verification

### AWS Verification

From WSL2:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,Size:Size,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

Expected: 10 GiB, `in-use`.

### Linux Verification

Inside EC2:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS

df -hT /data

findmnt /data

cat /data/important-data.txt

sha256sum /data/important-data.txt
```

### Verification Matrix

| Check | Initial | During Incident | Recovered |
|---|---|---|---|
| EBS Size | 5 GiB | 10 GiB | 10 GiB |
| Linux Disk | 5 GiB | 10 GiB | 10 GiB |
| ext4 Size | ~5 GiB | ~5 GiB | ~10 GiB |
| `/data` | Mounted | Mounted | Mounted |
| Original File | Present | Present | Present |
| File Checksum | Valid | Valid | Valid |
| New Capacity | No | No | Yes |
| Resize Operation | Not started | In progress or effective | Filesystem expanded |

### Success Criteria

- EC2 instance created.
- 5 GiB EBS data volume attached.
- Correct Linux device identified.
- ext4 filesystem created and mounted.
- Important test file created.
- Original checksum recorded.
- Pre-resize snapshot completed.
- AWS volume expanded to 10 GiB.
- Filesystem size mismatch reproduced.
- Root cause identified.
- ext4 expanded using `resize2fs`.
- New filesystem capacity verified.
- Original data preserved.
- Temporary AWS resources cleaned up.

---

## 17. Production Best Practices

### EBS Resize Troubleshooting Checklist

1. Confirm the AWS account and instance.
2. Identify the correct EBS volume ID.
3. Verify the volume and instance relationship.
4. Inspect the current EBS size.
5. Create a recovery snapshot.
6. Submit the EBS modification.
7. Check modification state.
8. Verify the new Linux block device size.
9. Identify the filesystem type.
10. Inspect the partition structure.
11. Expand partitions only when necessary.
12. Grow the filesystem using the correct tool.
13. Verify usable capacity.
14. Verify important data integrity.
15. Review system and application logs.

### Prevention

- Use infrastructure as code to manage EBS capacity.
- Document complete filesystem resize procedures.
- Monitor filesystem usage.
- Set storage capacity alarms.
- Create recovery snapshots before risky changes.
- Verify Linux filesystem capacity after every volume modification.
- Avoid modifying the wrong disk.
- Maintain tested backup and recovery procedures.

### Security Considerations

- Use least-privilege IAM permissions.
- Use Systems Manager for controlled administration.
- Keep EC2 security groups restrictive.
- Enable EBS encryption where required.
- Restrict snapshot access.
- Protect sensitive application data.
- Verify resource ownership before deletion.

---

## 18. Cleanup

**Warning:** Cleanup permanently deletes the disposable Lab 04 EBS data volume, recovery snapshot, EC2 instance, and supporting AWS resources.

### Step 1 — Unmount the Data Filesystem

Inside EC2:

```bash
cd /

findmnt /data

sudo fuser -vm /data
```

Confirm that no required processes are using `/data`.

Unmount safely:

```bash
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

### Step 2 — Verify AWS Resource Ownership

From Ubuntu WSL2:

```bash
aws sts get-caller-identity

aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,State:State,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"

aws ec2 describe-snapshots \
  --snapshot-ids "$SNAPSHOT_ID" \
  --query 'Snapshots[].{ID:SnapshotId,State:State,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"

aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"
```

Confirm the expected project tag before proceeding.

### Step 3 — Detach EBS

```bash
aws ec2 detach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 4 — Delete Data Volume

```bash
aws ec2 delete-volume \
  --volume-id "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 5 — Delete Recovery Snapshot

Only delete the snapshot after confirming it is not needed:

```bash
aws ec2 delete-snapshot \
  --snapshot-id "$SNAPSHOT_ID" \
  --region "$AWS_REGION"
```

### Step 6 — Inspect Root Volume Termination Setting

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].{RootDevice:RootDeviceName,BlockDevices:BlockDeviceMappings}' \
  --output json \
  --region "$AWS_REGION"
```

Confirm the root EBS volume is configured with `DeleteOnTermination=true`.

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

Verify the role and profile are dedicated to Lab 04 and not shared.

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

### Step 9 — Verify Network Interface Cleanup

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'NetworkInterfaces[].{ID:NetworkInterfaceId,Status:Status}' \
  --output table \
  --region "$AWS_REGION"
```

Wait until the instance's ENI has been removed.

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

aws ec2 describe-snapshots \
  --owner-ids self \
  --filters "Name=tag:Project,Values=$PROJECT_TAG" \
  --query 'Snapshots[].SnapshotId' \
  --output json \
  --region "$AWS_REGION"

aws ec2 describe-vpcs \
  --filters "Name=tag:Project,Values=$PROJECT_TAG" \
  --query 'Vpcs[].VpcId' \
  --output json \
  --region "$AWS_REGION"
```

Expected: empty arrays after deletion completes.

Also verify that the lab instance, root EBS volume, IAM role, and instance profile have been removed.

### Cleanup Summary

| Resource | Expected Result |
|---|---|
| 10 GiB Data Volume | Deleted |
| Pre-resize Snapshot | Deleted |
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

## 19. Interview Questions and Answers

### Q1. Why isn't an EBS resize automatically reflected in Linux?

Because expanding the underlying EBS block device does not automatically expand an existing filesystem.

### Q2. How do you inspect EBS volume capacity?

Use `aws ec2 describe-volumes`.

### Q3. How do you check an EBS modification?

Use `aws ec2 describe-volumes-modifications`.

### Q4. How do you check Linux disk capacity?

Use `lsblk` or `blockdev`.

### Q5. How do you check filesystem capacity?

Use `df -hT`.

### Q6. Which command expands ext4?

`resize2fs`.

### Q7. Which command expands XFS?

`xfs_growfs`.

### Q8. When is `growpart` needed?

When a filesystem is located inside a partition that has not yet been extended to use the larger block device.

### Q9. Can an EBS data volume be resized while EC2 is running?

Yes, supported EBS volumes can be expanded using Elastic Volumes without stopping the instance.

### Q10. Can `modify-volume` shrink an existing EBS volume?

No. The operation supports increasing volume size but not directly decreasing it.

### Q11. Why should you create a snapshot before resizing?

To preserve a recovery point before changing storage configuration.

### Q12. How would you explain this lab in an interview?

I deployed an EC2 instance with a separate 5 GiB gp3 EBS volume.

I created an ext4 filesystem and mounted it at `/data`.

After saving application test data and creating a recovery snapshot, I expanded the EBS volume to 10 GiB using AWS CLI.

AWS reported the new capacity, but Linux still showed the old filesystem size.

I investigated the mismatch using `describe-volumes`, `describe-volumes-modifications`, `lsblk`, and `df`.

After confirming the data disk used ext4 without partitions, I expanded the filesystem using `resize2fs`.

I verified the new capacity, confirmed the original data checksum was unchanged, and cleaned up the temporary AWS infrastructure.

---

## 20. Repository Structure

```text
04-ebs-resize-not-reflected/
├── README.md
└── ec2-trust-policy.json
```

### File Descriptions

| File | Purpose |
|---|---|
| `README.md` | Complete EBS resize troubleshooting documentation |
| `ec2-trust-policy.json` | EC2 IAM trust policy for Systems Manager |

Do not commit AWS credentials, private keys, or sensitive information.

---

## 21. Lessons Learned

This lab demonstrates that expanding an Amazon EBS volume does not automatically increase the usable capacity of an existing Linux filesystem.

The EBS volume can report its new capacity while the filesystem continues operating at the original size.

The most important troubleshooting step was comparing the AWS volume size, Linux block device size, and mounted filesystem capacity.

We also learned why it is important to identify the correct filesystem type and partition layout before running resize commands.

Using `resize2fs`, we expanded the existing ext4 filesystem without recreating it or deleting the application data.

The pre-resize snapshot provided an additional recovery option.

**Key Lesson:** A successful EBS volume modification is only one part of a complete Linux storage expansion. Always verify that the operating system, partition (if applicable), and filesystem can use the new capacity.

---

## 22. Skills Demonstrated

- Amazon EBS provisioning
- AWS Elastic Volumes
- EC2 storage management
- AWS CLI v2
- EBS volume modification
- EBS snapshot creation
- Linux block device troubleshooting
- NVMe device identification
- ext4 filesystem management
- `lsblk`
- `df`
- `blkid`
- `blockdev`
- `tune2fs`
- `resize2fs`
- Partition resizing fundamentals
- XFS resizing fundamentals
- Online filesystem expansion
- Data integrity verification
- Root cause analysis
- Safe storage remediation
- AWS resource cleanup

## Final Outcome

| Item | Expected Result |
|---|---|
| Incident | INC-EBS-004 |
| Problem | EBS resize not reflected in Linux |
| Initial EBS Capacity | 5 GiB |
| Expanded EBS Capacity | 10 GiB |
| Initial Filesystem | ~5 GiB |
| Filesystem During Fault | ~5 GiB |
| Final Filesystem | ~10 GiB |
| Filesystem | ext4 |
| Mount Point | `/data` |
| Root Cause | ext4 filesystem not expanded |
| Remediation | `resize2fs` |
| Recovery Snapshot | Created |
| Original Data | Preserved |
| Cleanup | Temporary resources removed |
| Expected Incident Status | Resolved |

**Key Takeaway:** Amazon EBS volume resizing and Linux filesystem resizing are separate operations. Successful cloud storage troubleshooting requires validating and extending the complete storage stack while preserving data integrity.
