# Lab 05 — EBS Snapshot Restore Failure

## Overview

This lab demonstrates how to troubleshoot an Amazon EBS snapshot restoration where the restored volume is successfully created and attached to an EC2 instance, but the recovered application files are not accessible through their expected Linux directory.

We create an Amazon Linux 2023 EC2 instance with a 5 GiB `gp3` EBS data volume. We format the original empty volume with ext4, mount it at `/data`, and create sample application files.

We then generate SHA-256 checksums, create an EBS snapshot, detach the original volume, and restore a new EBS volume from the snapshot.

To simulate a production recovery failure, we attach the restored volume to EC2 without mounting its filesystem.

The application cannot find its original files even though AWS shows that the restored volume is attached.

We investigate the snapshot state, volume attachment, Linux NVMe device, filesystem metadata, and mount status.

After identifying the root cause, we mount the restored ext4 filesystem without formatting it, verify the original data using checksums, and test read-write operations.

Finally, we safely clean up all temporary AWS resources.

**Troubleshooting workflow:**

Provision → Create Application Data → Snapshot → Restore → Fault Injection → Investigation → Root Cause Analysis → Remediation → Verification → Cleanup

---

## 1. Lab Information

| Item | Configuration |
|---|---|
| Incident ID | INC-EBS-005 |
| Difficulty | Intermediate |
| AWS Region | `us-east-1` |
| AWS Services | EC2, EBS, EBS Snapshots, VPC, IAM, Systems Manager |
| Operating System | Amazon Linux 2023 |
| Instance Type | `t3.micro` |
| Source EBS Volume | 5 GiB gp3 |
| Restored EBS Volume | 5 GiB gp3 |
| Filesystem | ext4 |
| Mount Point | `/data` |
| AWS Attachment Name | `/dev/sdf` |
| Linux Device | NVMe, such as `/dev/nvme1n1` |
| Fault Injection | Attach restored EBS without mounting |
| Primary Symptom | Recovered files inaccessible |
| Root Cause | Restored filesystem not mounted |
| Remediation | Mount existing ext4 filesystem |
| Data Integrity Verification | SHA-256 checksums |
| Expected Result | Original data recovered |

---

## 2. Production Incident Scenario

### Incident Description

A company runs an application on an Amazon EC2 instance.

The application stores its configuration and business records on a dedicated EBS data volume mounted at `/data`.

The engineering team maintains EBS snapshots for backup and disaster recovery.

During a recovery operation, an engineer creates a new EBS volume from a completed snapshot.

AWS reports that the restored volume is available and that the attachment to EC2 succeeds.

However, the application still cannot access its expected data files.

For example:

```bash
cat /data/application/config.txt
```

Returns:

```text
cat: /data/application/config.txt: No such file or directory
```

The engineer initially suspects that the snapshot restoration failed.

After investigation, the engineer discovers that the restored volume contains the original ext4 filesystem, but the filesystem has not been mounted at `/data`.

### Incident Symptoms

- Source snapshot is completed.
- New EBS volume is created from the snapshot.
- Restored volume state is `in-use`.
- EBS attachment is successful.
- Linux detects the restored NVMe block device.
- Existing ext4 filesystem is present.
- `/data` is not mounted.
- Application files cannot be accessed through `/data`.
- No evidence of missing snapshot data is established.

### Business Impact

- Application recovery is delayed.
- Important configuration files appear missing.
- Recovery time objectives may be missed.
- Applications cannot access their expected storage.
- Engineers may incorrectly attempt to reformat a restored filesystem.
- Incorrect recovery procedures increase the risk of data loss.

### Objective

Identify why application data remains inaccessible after snapshot restoration, mount the correct restored filesystem without overwriting existing data, and verify successful recovery.

---

## 3. AWS Architecture

### Original Architecture

```text
                  Amazon EC2
               Amazon Linux 2023
                       |
                       v
                Source EBS Volume
                   gp3 — 5 GiB
                       |
                       v
                 ext4 Filesystem
                       |
                       v
                  Mounted /data
                       |
                       v
                Application Files
                       |
                       v
                SHA-256 Checksums
```

### Snapshot Backup Architecture

```text
               Source EBS Volume
                       |
                       v
               Filesystem Unmounted
                       |
                       v
               Create EBS Snapshot
                       |
                       v
                 Snapshot Completed
                       |
                       v
               Recovery Point Stored
```

### Broken Restore Architecture

```text
                 EBS Snapshot
                       |
                       v
             New EBS Volume Created
                       |
                       v
              Volume Attached to EC2
                       |
                       v
             Linux Detects ext4 Device
                       |
                       X
               Filesystem Not Mounted
                       |
                       v
              /data Has No Restored Data
                       |
                       v
                Application Failure
```

### Recovered Architecture

```text
                 Restored EBS Volume
                         |
                         v
                Identify NVMe Device
                         |
                         v
                  Existing ext4
                         |
                         v
              Mount Read-only for Check
                         |
                         v
                Verify SHA-256 Data
                         |
                         v
              Mount for Normal Access
                         |
                         v
                 /data/application
                         |
                         v
                  Recovery Complete
```

### Infrastructure Configuration

| Resource | Configuration |
|---|---|
| VPC | `10.176.0.0/16` |
| Public Subnet | `10.176.1.0/24` |
| Internet Gateway | Attached |
| Route Table | Public internet route |
| EC2 Security Group | No inbound access |
| EC2 Management | Systems Manager |
| Source Volume | 5 GiB gp3 |
| Snapshot | Point-in-time source backup |
| Restored Volume | Created from snapshot |
| Filesystem | ext4 |
| Expected Mount | `/data` |

---

## 4. Understanding Amazon EBS Snapshots

An Amazon EBS snapshot is a point-in-time backup of an EBS volume.

Snapshots can be used to create new EBS volumes, including volumes in different Availability Zones within the supported scope.

An EBS snapshot preserves block-level data, which includes filesystem structures and files stored on the source volume.

### EBS Snapshot Workflow

```text
Source EBS
    |
    v
Create Snapshot
    |
    v
Wait for Completion
    |
    v
Create New Volume from Snapshot
    |
    v
Attach to EC2
    |
    v
Identify Existing Filesystem
    |
    v
Mount and Verify Data
```

### Important AWS Resource States

| Resource | State | Meaning |
|---|---|---|
| Snapshot | `pending` | Snapshot creation in progress |
| Snapshot | `completed` | Snapshot ready for normal restore operations |
| EBS Volume | `creating` | Volume provisioning in progress |
| EBS Volume | `available` | Volume ready and unattached |
| EBS Volume | `in-use` | Volume attached or otherwise in use |

### Snapshot Restoration vs. Formatting

Snapshot restoration and filesystem creation are different operations.

A volume restored from a snapshot of an ext4 filesystem already contains ext4 metadata.

**Never run `mkfs.ext4` on the restored volume as a routine recovery action.**

Formatting may overwrite the filesystem you are trying to recover.

### Attachment vs. Mounting

| Operation | Purpose |
|---|---|
| Restore snapshot | Create a new EBS volume from backup |
| Attach EBS | Connect restored block storage to EC2 |
| Mount filesystem | Make recovered files accessible through a directory |
| Verify checksums | Confirm restored files match the original recorded content |
| Test application | Confirm the recovered data can be used |

---

## 5. Prerequisites and Safety

### Requirements

- AWS account
- AWS CLI v2
- Ubuntu WSL2
- Bash
- Amazon EC2 permissions
- EBS volume and snapshot permissions
- IAM permissions
- Systems Manager Session Manager access
- Git and GitHub
- Basic Linux filesystem knowledge

### Important Safety Rules

- Use dedicated disposable EBS volumes.
- Verify the volume ID before any destructive operation.
- Never format the restored EBS volume.
- Do not accidentally select the EC2 root disk.
- Unmount the source filesystem cleanly before snapshot creation.
- Wait for snapshot completion before proceeding.
- Match EBS and EC2 Availability Zones for attachment.
- Keep the source volume detached while inspecting the restored clone.
- Use read-only mounting for initial recovery inspection.
- Preserve the original source volume until the restored data is verified.
- Delete only dedicated lab resources during cleanup.

### Cost Considerations

EC2 instances, EBS volumes, snapshots, root EBS storage, and public IPv4 addresses may generate AWS charges.

---

## 6. Prepare the Environment

### Step 1 — Set Environment Variables

Run in Ubuntu WSL2:

```bash
export AWS_REGION="us-east-1"
export PROJECT_TAG="EBS-Troubleshooting-Lab05"

export INSTANCE_NAME="cloud-ebs-lab05-ec2"
export SOURCE_NAME="cloud-ebs-lab05-source"
export RESTORED_NAME="cloud-ebs-lab05-restored"

export ROLE_NAME="cloud-ebs-lab05-ssm-role"
export PROFILE_NAME="cloud-ebs-lab05-ssm-profile"

export SG_NAME="cloud-ebs-lab05-sg"

export AZ_A="us-east-1a"
export VOLUME_SIZE="5"
```

### Step 2 — Verify AWS Access

```bash
aws sts get-caller-identity
aws --version
```

### Step 3 — Check Instance Availability

```bash
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters "Name=instance-type,Values=t3.micro" \
  --query 'InstanceTypeOfferings[].Location' \
  --output table \
  --region "$AWS_REGION"
```

Select a supported AZ if necessary.

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

aws ec2 describe-snapshots \
  --owner-ids self \
  --filters "Name=tag:Project,Values=$PROJECT_TAG" \
  --query 'Snapshots[].{ID:SnapshotId,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

Avoid provisioning duplicate resources.

---

## 7. Create EC2 Infrastructure

### Step 1 — Create a VPC

```bash
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block "10.176.0.0/16" \
  --tag-specifications \
    "ResourceType=vpc,Tags=[{Key=Name,Value=cloud-ebs-lab05-vpc},{Key=Project,Value=$PROJECT_TAG}]" \
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

### Step 2 — Create a Public Subnet

```bash
export SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block "10.176.1.0/24" \
  --availability-zone "$AZ_A" \
  --tag-specifications \
    "ResourceType=subnet,Tags=[{Key=Name,Value=cloud-ebs-lab05-subnet},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 3 — Create Internet Gateway

```bash
export IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications \
    "ResourceType=internet-gateway,Tags=[{Key=Name,Value=cloud-ebs-lab05-igw},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'InternetGateway.InternetGatewayId' \
  --output text \
  --region "$AWS_REGION")

aws ec2 attach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID" \
  --region "$AWS_REGION"
```

### Step 4 — Configure Route Table

```bash
export RT_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
    "ResourceType=route-table,Tags=[{Key=Name,Value=cloud-ebs-lab05-rt},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --description "Lab05 EBS snapshot restoration" \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
    "ResourceType=security-group,Tags=[{Key=Name,Value=$SG_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query GroupId \
  --output text \
  --region "$AWS_REGION")
```

No inbound access is required for this lab.

### Step 6 — Create EC2 Trust Policy

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

### Step 7 — Create IAM Role and Instance Profile

Check for existing resources with these names before creating them.

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

Allow IAM propagation.

### Step 8 — Get Amazon Linux 2023 AMI

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

### Step 11 — Retrieve Availability Zone

```bash
export INSTANCE_AZ=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' \
  --output text \
  --region "$AWS_REGION")

echo "$INSTANCE_AZ"
```

---

## 8. Create the Original EBS Data Volume

### Step 1 — Create the Source Volume

```bash
export SOURCE_VOLUME_ID=$(aws ec2 create-volume \
  --availability-zone "$INSTANCE_AZ" \
  --size "$VOLUME_SIZE" \
  --volume-type gp3 \
  --tag-specifications \
    "ResourceType=volume,Tags=[{Key=Name,Value=$SOURCE_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query VolumeId \
  --output text \
  --region "$AWS_REGION")
```

### Step 2 — Wait Until Available

```bash
aws ec2 wait volume-available \
  --volume-ids "$SOURCE_VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 3 — Verify the Source Volume

```bash
aws ec2 describe-volumes \
  --volume-ids "$SOURCE_VOLUME_ID" \
  --query 'Volumes[].{ID:VolumeId,State:State,Size:Size,AZ:AvailabilityZone}' \
  --output table \
  --region "$AWS_REGION"
```

Expected: 5 GiB, `available`, same AZ as EC2.

### Step 4 — Attach the Source Volume

```bash
aws ec2 attach-volume \
  --volume-id "$SOURCE_VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdf \
  --region "$AWS_REGION"

aws ec2 wait volume-in-use \
  --volume-ids "$SOURCE_VOLUME_ID" \
  --region "$AWS_REGION"
```

---

## 9. Prepare the Original Filesystem

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

### Step 2 — Identify the Source NVMe Device

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,SERIAL
```

Identify the new 5 GiB disk and compare its serial with the source EBS volume ID.

For example:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

After confirming the correct device:

```bash
export DATA_DEVICE="/dev/nvme1n1"
```

Replace the example path with the actual device.

### Step 3 — Inspect Filesystem Metadata

```bash
lsblk -f

sudo blkid "$DATA_DEVICE" || true

sudo file -s "$DATA_DEVICE"
```

Expected: no existing filesystem on the new empty data volume.

### Step 4 — Create ext4

**Destructive operation — only run on the confirmed empty source volume.**

```bash
sudo mkfs.ext4 "$DATA_DEVICE"
```

### Step 5 — Mount at `/data`

```bash
sudo mkdir -p /data

sudo mount "$DATA_DEVICE" /data
```

Verify:

```bash
findmnt /data

df -hT /data
```

### Step 6 — Create Sample Application Files

```bash
sudo mkdir -p /data/application
```

Create three fictional test files:

```bash
echo "DATABASE_HOST=production-db" \
  | sudo tee /data/application/config.txt

echo "Customer ID: 1001 | Status: Active" \
  | sudo tee /data/application/records.txt

echo "EBS Lab 05 - Snapshot Recovery Test" \
  | sudo tee /data/application/recovery-marker.txt
```

These are fictional examples. No real credentials or customer data should be stored.

### Step 7 — Create SHA-256 Checksums

```bash
cd /data/application

sha256sum config.txt records.txt recovery-marker.txt \
  | sudo tee /data/original-checksums.sha256
```

Check the manifest:

```bash
cat /data/original-checksums.sha256
```

### Step 8 — Verify the Healthy Baseline

```bash
findmnt /data

df -hT /data

ls -lah /data/application

cat /data/application/config.txt

cat /data/application/records.txt
```

Expected: all files are accessible.

---

## 10. Create the EBS Snapshot

### Step 1 — Flush Writes and Unmount

Inside EC2:

```bash
cd /

sync

sudo umount /data
```

Verify:

```bash
findmnt /data
```

Expected: `/data` is not mounted.

Exit the SSM session:

```bash
exit
```

### Step 2 — Create Snapshot

From Ubuntu WSL2:

```bash
export SNAPSHOT_ID=$(aws ec2 create-snapshot \
  --volume-id "$SOURCE_VOLUME_ID" \
  --description "Lab05 application data recovery snapshot" \
  --tag-specifications \
    "ResourceType=snapshot,Tags=[{Key=Name,Value=cloud-ebs-lab05-snapshot},{Key=Project,Value=$PROJECT_TAG}]" \
  --query SnapshotId \
  --output text \
  --region "$AWS_REGION")

echo "$SNAPSHOT_ID"
```

### Step 3 — Wait for Completion

```bash
aws ec2 wait snapshot-completed \
  --snapshot-ids "$SNAPSHOT_ID" \
  --region "$AWS_REGION"
```

### Step 4 — Verify Snapshot Metadata

```bash
aws ec2 describe-snapshots \
  --snapshot-ids "$SNAPSHOT_ID" \
  --query 'Snapshots[].{ID:SnapshotId,State:State,Size:VolumeSize,Source:VolumeId}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
State: completed
Volume Size: 5 GiB
Source: Original volume ID
```

---

## 11. Restore the EBS Snapshot

### Step 1 — Detach the Source Volume

The filesystem has already been unmounted.

```bash
aws ec2 detach-volume \
  --volume-id "$SOURCE_VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait volume-available \
  --volume-ids "$SOURCE_VOLUME_ID" \
  --region "$AWS_REGION"
```

Keep the original volume detached for the remainder of the recovery test.

### Step 2 — Create the Restored Volume

```bash
export RESTORED_VOLUME_ID=$(aws ec2 create-volume \
  --snapshot-id "$SNAPSHOT_ID" \
  --availability-zone "$INSTANCE_AZ" \
  --volume-type gp3 \
  --tag-specifications \
    "ResourceType=volume,Tags=[{Key=Name,Value=$RESTORED_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query VolumeId \
  --output text \
  --region "$AWS_REGION")

echo "$RESTORED_VOLUME_ID"
```

### Step 3 — Wait Until Available

```bash
aws ec2 wait volume-available \
  --volume-ids "$RESTORED_VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 4 — Verify the Restored Volume

```bash
aws ec2 describe-volumes \
  --volume-ids "$RESTORED_VOLUME_ID" \
  --query 'Volumes[].{ID:VolumeId,State:State,Size:Size,Snapshot:SnapshotId,AZ:AvailabilityZone}' \
  --output table \
  --region "$AWS_REGION"
```

Expected: 5 GiB volume created from the correct snapshot.

### Step 5 — Attach the Restored Volume

```bash
aws ec2 attach-volume \
  --volume-id "$RESTORED_VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdf \
  --region "$AWS_REGION"

aws ec2 wait volume-in-use \
  --volume-ids "$RESTORED_VOLUME_ID" \
  --region "$AWS_REGION"
```

Verify:

```bash
aws ec2 describe-volumes \
  --volume-ids "$RESTORED_VOLUME_ID" \
  --query 'Volumes[0].{State:State,Snapshot:SnapshotId,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Expected: `in-use` and attached.

---

## 12. Fault Injection — Restored Volume Not Mounted

### Step 1 — Connect to EC2

```bash
aws ssm start-session \
  --target "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

### Step 2 — Attempt to Access Restored Application Data

Inside EC2:

```bash
cat /data/application/config.txt
```

Expected:

```text
cat: /data/application/config.txt: No such file or directory
```

Check another file:

```bash
cat /data/application/records.txt
```

Expected: file not found.

**This is the intentionally reproduced recovery failure.**

### Step 3 — Check the Active Mount

```bash
findmnt /data
```

Expected: no active mount.

### Step 4 — Check Linux Block Devices

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,SERIAL
```

Expected: a 5 GiB block device already containing ext4.

### Step 5 — Identify the Restored Volume

Use the NVMe serial and volume identification information:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

Replace the device path as required.

Confirm it matches the restored EBS volume ID.

Set the verified device:

```bash
export RESTORE_DEVICE="/dev/nvme1n1"
```

### Step 6 — Check Existing Filesystem Metadata

```bash
sudo blkid "$RESTORE_DEVICE"

lsblk -f

sudo file -s "$RESTORE_DEVICE"
```

Expected: an existing ext4 filesystem.

**Do not run `mkfs.ext4`.** The filesystem already exists and contains the backed-up data.

---

## 13. Troubleshooting Investigation

### Check 1 — Verify Snapshot State

From WSL2:

```bash
aws ec2 describe-snapshots \
  --snapshot-ids "$SNAPSHOT_ID" \
  --query 'Snapshots[].{ID:SnapshotId,State:State,Size:VolumeSize}' \
  --output table \
  --region "$AWS_REGION"
```

Finding: snapshot completed.

### Check 2 — Verify Restored Volume State

```bash
aws ec2 describe-volumes \
  --volume-ids "$RESTORED_VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,State:State,AZ:AvailabilityZone,Snapshot:SnapshotId,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Finding: restored volume attached.

### Check 3 — Inspect Linux Storage

Inside EC2:

```bash
lsblk -f
```

Finding: ext4 filesystem exists.

### Check 4 — Check Mount Status

```bash
findmnt /data
```

Finding: no active mount.

### Check 5 — Inspect Filesystem Type

```bash
sudo blkid "$RESTORE_DEVICE"
```

Finding: restored device has ext4 filesystem metadata.

### Check 6 — Check Visible Filesystems

```bash
df -hT
```

Finding: the restored ext4 volume is absent from mounted filesystem output.

### Investigation Summary

| Investigation | Finding |
|---|---|
| Snapshot | Completed |
| Snapshot Source | Correct original volume |
| Restored Volume | Created successfully |
| Availability Zone | Compatible |
| AWS Attachment | Successful |
| Linux Block Device | Visible |
| Existing Filesystem | ext4 |
| `/data` Mount | Missing |
| Application Files | Inaccessible at expected path |
| Root Cause | Restored filesystem not mounted |

---

## 14. Root Cause Analysis

### Root Cause

The EBS snapshot was restored successfully, but its existing ext4 filesystem had not been mounted at `/data`.

The EBS attachment made the block device available inside Linux, but Linux did not automatically expose the restored files at the expected directory.

### Technical Explanation

EBS snapshot restoration creates a new volume containing the source volume's block data.

That includes filesystem structures and stored files.

After attaching the new volume, the operating system must recognize the correct block device and mount its existing filesystem.

Without a mount, the application cannot access recovered files through the expected mount point.

### Root Cause Flow

```text
Source EBS Volume
        |
        v
Completed Snapshot
        |
        v
Restored EBS Volume
        |
        v
Successful EC2 Attachment
        |
        v
Linux Detects ext4
        |
        X
No Mount at /data
        |
        v
Application Files Unavailable
```

### Five Whys

1. **Why are restored files unavailable?** The expected application directory is not backed by the restored filesystem.
2. **Why is the filesystem inaccessible?** The restored volume has not been mounted.
3. **Why does AWS report a successful attachment?** EBS attachment and Linux filesystem mounting are separate operations.
4. **Why is ext4 already present?** The snapshot preserved the source filesystem metadata.
5. **How can the problem be prevented?** Use a documented recovery procedure that verifies snapshot state, attachment, device identity, mounting, and file integrity.

### Prevention

Implement a recovery checklist covering:

```text
Verify Snapshot
      |
      v
Create Volume
      |
      v
Verify Availability Zone
      |
      v
Attach Volume
      |
      v
Identify Block Device
      |
      v
Inspect Filesystem
      |
      v
Mount Existing Filesystem
      |
      v
Verify Data Integrity
      |
      v
Restore Application
```

---

## 15. Remediation — Recover the Filesystem

### Step 1 — Verify the Correct Device

Inside EC2:

```bash
echo "$RESTORE_DEVICE"

lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,SERIAL

sudo blkid "$RESTORE_DEVICE"
```

Confirm the EBS volume ID and filesystem type before proceeding.

### Step 2 — Mount the Restored Filesystem Read-Only

```bash
sudo mkdir -p /data

sudo mount -t ext4 -o ro,noload \
  "$RESTORE_DEVICE" /data
```

`ro,noload` is used for the initial read-only inspection without ext4 journal replay.

The source filesystem was cleanly unmounted before snapshot creation, making this an appropriate initial inspection method for the lab.

### Step 3 — Verify the Mount

```bash
findmnt /data

df -hT /data
```

Expected: `/data` backed by the restored ext4 filesystem.

### Step 4 — Recover Application Files

```bash
ls -lah /data/application

cat /data/application/config.txt

cat /data/application/records.txt

cat /data/application/recovery-marker.txt
```

Expected:

```text
DATABASE_HOST=production-db
Customer ID: 1001 | Status: Active
EBS Lab 05 - Snapshot Recovery Test
```

### Step 5 — Verify SHA-256 Checksums

```bash
cd /data/application

sha256sum -c /data/original-checksums.sha256
```

Expected:

```text
config.txt: OK
records.txt: OK
recovery-marker.txt: OK
```

All three checksums should pass.

### Step 6 — Remount for Read-Write Access

```bash
cd /

sudo umount /data

sudo mount -t ext4 "$RESTORE_DEVICE" /data
```

Verify:

```bash
findmnt /data

findmnt -no OPTIONS /data
```

Expected: normal read-write mount.

### Step 7 — Test Recovery Writes

```bash
echo "Snapshot recovery successful" \
  | sudo tee /data/application/recovery-success.txt
```

Verify:

```bash
cat /data/application/recovery-success.txt
```

Expected:

```text
Snapshot recovery successful
```

### Step 8 — Verify the Filesystem UUID

```bash
sudo blkid "$RESTORE_DEVICE"
```

A snapshot-restored filesystem normally retains the source filesystem's UUID.

Avoid ambiguous UUID-based mount configurations when the original and restored filesystems are both available to the same operating system.

In this lab, only the restored volume is attached during recovery testing.

---

## 16. Final Verification

### AWS Verification

From WSL2:

```bash
aws ec2 describe-snapshots \
  --snapshot-ids "$SNAPSHOT_ID" \
  --query 'Snapshots[].{ID:SnapshotId,State:State}' \
  --output table \
  --region "$AWS_REGION"

aws ec2 describe-volumes \
  --volume-ids "$RESTORED_VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,State:State,Snapshot:SnapshotId,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

### Linux Verification

Inside EC2:

```bash
lsblk -f

findmnt /data

df -hT /data

ls -lah /data/application

cd /data/application

sha256sum -c /data/original-checksums.sha256

cat /data/application/recovery-success.txt
```

### Verification Matrix

| Check | During Incident | After Remediation |
|---|---|---|
| Snapshot | Completed | Completed |
| Restored EBS | In-use | In-use |
| Linux Device | Visible | Visible |
| Filesystem | ext4 | ext4 |
| `/data` Mounted | No | Yes |
| Application Files | Inaccessible | Accessible |
| SHA-256 Checksums | Not yet verified | All expected to pass |
| New Writes | Unavailable at data mount | Successful |
| Original Source | Detached | Remains preserved until cleanup |
| Recovery Status | Incomplete | Complete |

### Success Criteria

- EC2 instance provisioned.
- Original EBS volume attached.
- ext4 filesystem created.
- Test application files written.
- SHA-256 checksums recorded.
- Source filesystem safely unmounted.
- Snapshot completed.
- Original volume detached.
- Restored volume created from snapshot.
- Restored volume attached successfully.
- Missing mount problem reproduced.
- Correct restored NVMe device identified.
- Existing ext4 filesystem mounted without formatting.
- All original checksums verified.
- Restored volume tested for read-write access.
- Temporary AWS resources cleaned up.

---

## 17. Additional Snapshot Recovery Troubleshooting

### Problem 1 — Incorrect Availability Zone

A new volume must be created in a compatible Availability Zone for attachment.

Check:

```bash
aws ec2 describe-volumes \
  --volume-ids "$RESTORED_VOLUME_ID" \
  --query 'Volumes[0].AvailabilityZone' \
  --output text \
  --region "$AWS_REGION"
```

Remediation: Create a volume from the snapshot in the instance's Availability Zone.

### Problem 2 — Wrong Snapshot Selected

Check the snapshot ID, creation time, and source volume:

```bash
aws ec2 describe-snapshots \
  --snapshot-ids "$SNAPSHOT_ID" \
  --query 'Snapshots[].{ID:SnapshotId,Start:StartTime,Source:VolumeId,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

Confirm the snapshot represents the required recovery point.

### Problem 3 — Incorrect Linux Device

Device names such as `/dev/nvme1n1` can change between attachments.

Use:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,SERIAL
```

Match the disk serial against the intended EBS volume ID.

### Problem 4 — Duplicate Filesystem UUID

Snapshot restoration preserves filesystem metadata, often including UUIDs.

If the original and restored volumes are attached simultaneously, mount identification can become ambiguous.

Identify devices using their actual EBS volume IDs and apply a deliberate UUID management strategy when necessary.

### Problem 5 — Slow Initial Reads

Restored EBS volumes may experience increased first-read latency while data is fetched from the snapshot storage layer.

This does not necessarily indicate a failed restore.

### Problem 6 — KMS Access Failure

Encrypted EBS snapshots require appropriate permissions to the relevant KMS key.

Investigate IAM policies, KMS key policies, grants, and snapshot encryption information.

This subject is covered in Lab 06.

### Problem 7 — Filesystem Metadata or Data Problems

If the correct restored volume cannot be mounted or the checksums fail, preserve the volume and investigate before writing or performing repairs.

Avoid running destructive filesystem repair or formatting operations without a backup and a recovery plan.

---

## 18. Production Best Practices

### Snapshot Recovery Checklist

1. Identify the affected application and required data.
2. Determine the correct recovery point.
3. Verify snapshot completion.
4. Confirm the snapshot belongs to the expected source volume.
5. Create a restored volume in the target AZ.
6. Verify restored volume state.
7. Attach it to the correct EC2 instance.
8. Identify its Linux block device.
9. Inspect existing filesystem metadata.
10. Mount read-only for initial recovery inspection.
11. Verify expected files and checksums.
12. Mount for normal operation when appropriate.
13. Verify application access.
14. Preserve original recovery evidence until validation is complete.
15. Document the incident and recovery.

### Prevention

- Automate EBS snapshot creation.
- Use AWS Backup or a suitable snapshot lifecycle solution.
- Define recovery point and recovery time objectives.
- Practice snapshot restoration regularly.
- Verify snapshot and restored-volume identifiers.
- Maintain accurate mount documentation.
- Store critical data integrity information securely.
- Avoid unnecessary formatting.
- Test recovered applications, not just AWS resource states.
- Monitor storage and backup health.

### Security Considerations

- Use least-privilege IAM policies.
- Use EBS encryption according to security requirements.
- Limit snapshot sharing permissions.
- Review KMS access for encrypted backups.
- Protect restored application data.
- Avoid storing real secrets in test files.
- Use Systems Manager for secure administration.
- Delete temporary recovery resources after verification.

---

## 19. Cleanup

**Warning:** These steps permanently delete the Lab 05 source volume, restored volume, snapshot, EC2 instance, and supporting infrastructure.

Verify that all resources belong to `EBS-Troubleshooting-Lab05` and are no longer needed.

### Step 1 — Unmount the Restored Filesystem

Inside EC2:

```bash
cd /

findmnt /data

sudo fuser -vm /data
```

After confirming no required processes are using it:

```bash
sudo umount /data
```

Verify:

```bash
findmnt /data
```

Exit SSM:

```bash
exit
```

### Step 2 — Verify AWS Ownership

From WSL2:

```bash
aws sts get-caller-identity

aws ec2 describe-volumes \
  --volume-ids "$SOURCE_VOLUME_ID" "$RESTORED_VOLUME_ID" \
  --query 'Volumes[].{ID:VolumeId,State:State,Tags:Tags}' \
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

Confirm that these are disposable lab resources.

### Step 3 — Detach Restored Volume

```bash
aws ec2 detach-volume \
  --volume-id "$RESTORED_VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait volume-available \
  --volume-ids "$RESTORED_VOLUME_ID" \
  --region "$AWS_REGION"
```

The original source volume should already be detached.

### Step 4 — Delete Both Data Volumes

```bash
aws ec2 delete-volume \
  --volume-id "$SOURCE_VOLUME_ID" \
  --region "$AWS_REGION"

aws ec2 delete-volume \
  --volume-id "$RESTORED_VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 5 — Delete Snapshot

Only after confirming the recovery point is no longer needed:

```bash
aws ec2 delete-snapshot \
  --snapshot-id "$SNAPSHOT_ID" \
  --region "$AWS_REGION"
```

### Step 6 — Inspect Root Volume Settings

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].{RootDevice:RootDeviceName,BlockDevices:BlockDeviceMappings}' \
  --output json \
  --region "$AWS_REGION"
```

Confirm the root EBS volume has `DeleteOnTermination=true`.

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

Verify the role and profile are not shared with other infrastructure.

```bash
aws iam list-role-tags \
  --role-name "$ROLE_NAME"

aws iam get-instance-profile \
  --instance-profile-name "$PROFILE_NAME"
```

Delete dedicated resources:

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

Wait until the instance ENI is removed.

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

### Step 11 — Final Cleanup Verification

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

Expected: empty arrays when cleanup finishes.

Also verify that the lab instance, root volume, IAM role, and instance profile have been removed.

### Cleanup Summary

| Resource | Expected Result |
|---|---|
| Source EBS Volume | Deleted |
| Restored EBS Volume | Deleted |
| EBS Snapshot | Deleted |
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

### Q1. What is an EBS snapshot?

An EBS snapshot is a point-in-time backup of an EBS volume that can be used to create another EBS volume.

### Q2. Does an EBS snapshot contain application data?

Yes. A snapshot preserves the source volume's stored blocks, including filesystem structures and data.

### Q3. How do you restore an EBS snapshot?

Create a new EBS volume from the snapshot, attach it to EC2, identify the filesystem, and mount it.

### Q4. Why did the snapshot restore appear to fail in this lab?

The restored volume was attached, but its filesystem was not mounted at the expected directory.

### Q5. Should you format a restored EBS volume?

Not when restoring an existing filesystem. Formatting could overwrite the data being recovered.

### Q6. Can an EBS snapshot be used to create a volume in another Availability Zone?

Yes. The volume can be created in a supported AZ and attached to a compatible EC2 instance in that AZ.

### Q7. What is the difference between EBS attachment and mounting?

Attachment connects block storage to EC2. Mounting exposes the filesystem through a Linux directory.

### Q8. How can you identify the correct restored EBS device in Linux?

Use `lsblk`, NVMe serial information, and `ebsnvme-id` to match the device to its EBS volume ID.

### Q9. How can you verify restored data integrity?

Use SHA-256 checksums and verify that important files match their recorded original content.

### Q10. What is EBS volume initialization?

It is the process of preparing blocks backed by an EBS snapshot for access on the restored volume. Initial reads may have increased latency.

### Q11. What security issue can prevent encrypted snapshot restoration?

Insufficient IAM or AWS KMS permissions can prevent some encrypted snapshot and volume operations.

### Q12. How would you explain this lab in an interview?

I created an EBS data volume attached to an Amazon Linux EC2 instance.

I formatted the empty volume with ext4, mounted it at `/data`, and wrote application test files.

After recording SHA-256 checksums, I created an EBS snapshot, detached the source volume, and restored a new EBS volume from the snapshot.

I intentionally attached the restored volume without mounting it, reproducing a situation where the application could not find its original data.

I investigated the snapshot state, volume attachment, NVMe device, filesystem metadata, and Linux mount state using AWS CLI and Linux commands.

After identifying the missing mount, I mounted the existing filesystem without formatting, verified all original checksums, and confirmed that new files could be written successfully.

---

## 21. Repository Structure

```text
05-snapshot-restore-failure/
├── README.md
└── ec2-trust-policy.json
```

### File Descriptions

| File | Purpose |
|---|---|
| `README.md` | Complete EBS snapshot restoration troubleshooting documentation |
| `ec2-trust-policy.json` | EC2 trust policy used for Systems Manager access |

No credentials, private keys, or sensitive data should be committed.

---

## 22. Lessons Learned

This lab demonstrates that creating a new EBS volume from a snapshot is only part of the recovery process.

The restored volume can be healthy and correctly attached while the application remains unable to access its data.

The key investigation was comparing AWS snapshot and attachment states with the Linux device, filesystem, and mount state.

We learned that an EBS snapshot preserves filesystem data, so a restored volume generally does not need to be formatted again.

We also practiced identifying the correct NVMe device, mounting the filesystem read-only for recovery inspection, and validating recovered files using SHA-256 checksums.

**Key Lesson:** A successful AWS snapshot restore does not automatically mean that an application has recovered. Always verify the filesystem mount, original files, data integrity, and application access.

---

## 23. Skills Demonstrated

- Amazon EBS provisioning
- EBS snapshot creation
- Snapshot restoration
- EC2 volume attachments
- Availability Zone compatibility
- AWS CLI v2
- Linux block device troubleshooting
- NVMe identification
- ext4 filesystem management
- Linux mounting and unmounting
- Read-only recovery inspection
- Filesystem UUID awareness
- SHA-256 integrity verification
- AWS Systems Manager
- Root cause analysis
- Disaster recovery fundamentals
- Safe AWS resource cleanup

## Final Outcome

| Item | Expected Result |
|---|---|
| Incident | INC-EBS-005 |
| Problem | Restored snapshot data inaccessible |
| Source EBS | 5 GiB gp3 |
| Snapshot | Completed |
| Restored EBS | 5 GiB gp3 |
| Attachment | Successful |
| Filesystem | ext4 |
| Fault | Restored filesystem not mounted |
| Root Cause | Missing `/data` mount |
| Remediation | Mount existing filesystem |
| Checksum Verification | All expected to pass |
| Read-Write Verification | Successful |
| Original Data | Preserved |
| Cleanup | Temporary resources removed |
| Expected Incident Status | Resolved |

**Key Takeaway:** EBS snapshot recovery must be validated at both the AWS infrastructure layer and the Linux filesystem layer. Restored data should be identified, mounted, and verified without unnecessary destructive operations.
