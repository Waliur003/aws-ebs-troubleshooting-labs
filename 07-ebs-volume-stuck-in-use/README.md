# Lab 07 — EBS Volume Stuck in Use

## Overview

This lab demonstrates how to troubleshoot an Amazon EBS volume that cannot be safely detached because its Linux filesystem is being used by an active process.

We deploy an Amazon Linux 2023 EC2 instance and attach a dedicated 5 GiB `gp3` EBS data volume.

After identifying the correct Linux NVMe block device, we create an ext4 filesystem, mount it at `/data`, and store an important test file.

We then intentionally keep a shell process inside `/data` and attempt to unmount the filesystem.

Linux returns:

```text
umount: /data: target is busy.
```

We investigate the error using `fuser`, `findmnt`, `lsblk`, `ps`, and AWS CLI.

After identifying the blocking process, we release the filesystem, flush pending writes, unmount it safely, and detach the EBS volume without using force.

Finally, we reattach the volume, verify the original application data, and clean up the temporary AWS infrastructure.

**Troubleshooting workflow:**

Provision → Healthy Baseline → Fault Injection → Investigation → Root Cause Analysis → Remediation → Verification → Cleanup

---

## 1. Lab Information

| Item | Configuration |
|---|---|
| Incident ID | INC-EBS-007 |
| Difficulty | Intermediate |
| AWS Region | `us-east-1` |
| AWS Services | EC2, EBS, VPC, IAM, Systems Manager |
| Operating System | Amazon Linux 2023 |
| Instance Type | `t3.micro` |
| EBS Volume | 5 GiB `gp3` |
| AWS Device Name | `/dev/sdf` |
| Linux Block Device | Verified NVMe device, such as `/dev/nvme1n1` |
| Filesystem | ext4 |
| Mount Point | `/data` |
| Fault Injection | Keep shell working directory inside `/data` |
| Expected Error | `target is busy` |
| Investigation | `fuser`, `ps`, `findmnt`, `lsblk`, AWS CLI |
| Root Cause | Active process holding filesystem reference |
| Remediation | Release process, unmount, detach |
| Data Verification | SHA-256 checksum |
| Expected Final EBS State | `available` after detachment |

---

## 2. Production Incident Scenario

### Incident Description

A company uses an EC2 instance to run an application that stores important data on a separate EBS volume mounted at `/data`.

The infrastructure team needs to detach the volume for maintenance or migration.

Before detaching it through AWS, the engineer attempts to unmount the filesystem.

However, Linux returns:

```text
umount: /data: target is busy.
```

The EBS volume still shows `in-use` in AWS.

The engineer needs to identify which process is using the filesystem, safely release the process, and perform a normal EBS detachment.

### Incident Symptoms

- EC2 instance is running.
- EBS volume state is `in-use`.
- EBS attachment state is `attached`.
- Linux recognizes the EBS block device.
- The ext4 filesystem is mounted at `/data`.
- `umount /data` fails.
- `fuser` identifies an active process using `/data`.
- Original application files remain accessible.

### Potential Business Impact

- Planned maintenance is delayed.
- Storage migration cannot proceed safely.
- Backup or recovery operations may be interrupted.
- Incorrect forced-detachment procedures can risk data corruption.
- Application I/O may be interrupted.
- Operational recovery time may increase.

### Objective

Identify the active Linux process preventing unmounting, safely release the filesystem, detach the EBS volume, and verify that the original data remains intact.

### Important Technical Distinction

An EBS volume showing `in-use` is normally attached to an EC2 instance.

It is not necessarily experiencing a failure.

This lab intentionally reproduces a Linux filesystem-busy condition, not an actual AWS control-plane detachment failure.

We will also discuss how to investigate an EBS attachment that genuinely becomes stuck in `detaching` or `busy`.

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
                 Mounted at /data
                        |
                        v
                 Application Files
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
                  ext4 Mounted
                    at /data
                        |
                        v
                 Active Shell Process
                  Working in /data
                        |
                        v
                  sudo umount /data
                        |
                        X
                    target is busy
                        |
                        v
               Safe Detachment Blocked
```

### Recovered Architecture

```text
                Identify Busy Process
                         |
                         v
                    sudo fuser
                         |
                         v
                 Release /data Reference
                         |
                         v
                         sync
                         |
                         v
                   sudo umount
                         |
                         v
                 AWS DetachVolume
                         |
                         v
                EBS State: available
                         |
                         v
                 Reattach and Verify
                         |
                         v
                 Original Data Intact
```

### Infrastructure Configuration

| Resource | Configuration |
|---|---|
| VPC | `10.178.0.0/16` |
| Public Subnet | `10.178.1.0/24` |
| Internet Gateway | Attached |
| Route Table | Internet access through IGW |
| EC2 Security Group | No inbound rules |
| EC2 Management | AWS Systems Manager |
| Data Volume | 5 GiB gp3 |
| Filesystem | ext4 |
| Mount Point | `/data` |

---

## 4. Understanding EBS Volume States

Amazon EBS volumes can move through different states during creation, attachment, detachment, and deletion.

### Common EBS Volume States

| State | Meaning |
|---|---|
| `creating` | Volume is being provisioned |
| `available` | Volume exists and is not attached |
| `in-use` | Volume is attached |
| `deleting` | Volume deletion is in progress |
| `deleted` | Volume has been deleted |
| `error` | Volume has encountered a serious problem |

### Common Attachment States

| State | Meaning |
|---|---|
| `attaching` | Attachment request is in progress |
| `attached` | Volume is attached |
| `detaching` | Detachment is in progress |
| `detached` | Volume is detached |
| `busy` | AWS reports an attachment-related busy condition |

### Attachment vs. Mounting

| Operation | Description |
|---|---|
| Attach EBS | Connect block storage to EC2 |
| Mount filesystem | Make the filesystem available through a directory |
| Unmount filesystem | Release the filesystem from the Linux directory tree |
| Detach EBS | Disconnect the volume from the EC2 instance |

An EBS volume can remain attached even after the Linux filesystem is unmounted.

Likewise, an EBS volume can be attached while its filesystem is busy.

### Why Linux Prevents Unmounting

Linux normally prevents unmounting a filesystem when active references still exist.

Common causes include:

- A process has its working directory inside the mount.
- An application has an open file.
- A service is writing logs.
- A backup process is reading data.
- A database is using the filesystem.
- Another filesystem is mounted beneath the target directory.

In this exercise, we intentionally use a shell process whose current working directory is `/data`.

---

## 5. Prerequisites and Safety

### Requirements

- AWS account
- AWS CLI v2
- Ubuntu WSL2
- Bash
- EC2 permissions
- EBS permissions
- IAM permissions
- VPC permissions
- AWS Systems Manager
- Git and GitHub

### Safety Rules

**Never force-detach a production EBS volume without understanding the consequences.**

- Use a dedicated disposable EBS volume.
- Verify the correct volume ID.
- Verify the Linux NVMe device.
- Never format the EC2 root disk.
- Identify active processes before attempting unmounting.
- Stop applications gracefully when needed.
- Do not terminate unrelated processes.
- Avoid lazy unmounting as a routine workaround.
- Avoid forced unmounting.
- Do not use AWS `detach-volume --force` in this lab.
- Verify data integrity after reattachment.
- Remove only dedicated lab resources.

### Cost Considerations

EC2 instances, EBS volumes, root EBS storage, public IPv4 addresses, and associated traffic may incur AWS charges.

---

## 6. Prepare the Environment

### Step 1 — Set Environment Variables

Run in Ubuntu WSL2:

```bash
export AWS_REGION="us-east-1"
export PROJECT_TAG="EBS-Troubleshooting-Lab07"

export INSTANCE_NAME="cloud-ebs-lab07-ec2"
export VOLUME_NAME="cloud-ebs-lab07-data"

export SG_NAME="cloud-ebs-lab07-sg"
export ROLE_NAME="cloud-ebs-lab07-ssm-role"
export PROFILE_NAME="cloud-ebs-lab07-ssm-profile"

export AZ_A="us-east-1a"
export VOLUME_SIZE="5"
```

### Step 2 — Verify AWS Identity

```bash
aws sts get-caller-identity
aws --version
```

### Step 3 — Verify Instance Type Availability

```bash
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters "Name=instance-type,Values=t3.micro" \
  --query 'InstanceTypeOfferings[].Location' \
  --output table \
  --region "$AWS_REGION"
```

Select a supported AZ if necessary.

### Step 4 — Check for Existing Resources

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

Avoid creating duplicate billable resources.

---

## 7. Create EC2 Infrastructure

### Step 1 — Create VPC

```bash
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block "10.178.0.0/16" \
  --tag-specifications \
    "ResourceType=vpc,Tags=[{Key=Name,Value=cloud-ebs-lab07-vpc},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --cidr-block "10.178.1.0/24" \
  --availability-zone "$AZ_A" \
  --tag-specifications \
    "ResourceType=subnet,Tags=[{Key=Name,Value=cloud-ebs-lab07-subnet},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 3 — Create Internet Gateway

```bash
export IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications \
    "ResourceType=internet-gateway,Tags=[{Key=Name,Value=cloud-ebs-lab07-igw},{Key=Project,Value=$PROJECT_TAG}]" \
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
    "ResourceType=route-table,Tags=[{Key=Name,Value=cloud-ebs-lab07-rt},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --description "Lab07 EBS volume busy troubleshooting" \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
    "ResourceType=security-group,Tags=[{Key=Name,Value=$SG_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query GroupId \
  --output text \
  --region "$AWS_REGION")
```

No inbound rules are required for Systems Manager.

### Step 6 — Create EC2 Trust Policy

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

### Step 7 — Create IAM Role and Instance Profile

Verify that these names are not already used by unrelated resources.

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

## 8. Create and Attach EBS

### Step 1 — Create 5 GiB EBS Volume

```bash
export VOLUME_ID=$(aws ec2 create-volume \
  --availability-zone "$INSTANCE_AZ" \
  --size "$VOLUME_SIZE" \
  --volume-type gp3 \
  --tag-specifications \
    "ResourceType=volume,Tags=[{Key=Name,Value=$VOLUME_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query VolumeId \
  --output text \
  --region "$AWS_REGION")
```

### Step 2 — Wait for Availability

```bash
aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 3 — Verify Volume

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,State:State,Size:Size,AZ:AvailabilityZone}' \
  --output table \
  --region "$AWS_REGION"
```

Expected: 5 GiB, `available`, same AZ as EC2.

### Step 4 — Attach Volume

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
  --query 'Volumes[0].{State:State,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Expected:

```text
Volume State: in-use
Attachment State: attached
```

---

## 9. Prepare Linux Filesystem

### Step 1 — Connect Using Systems Manager

From WSL2:

```bash
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
  --query 'InstanceInformationList[].{ID:InstanceId,Status:PingStatus}' \
  --output table \
  --region "$AWS_REGION"
```

When the instance is online:

```bash
aws ssm start-session \
  --target "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

The following Linux commands run inside EC2.

### Step 2 — Identify EBS NVMe Device

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,SERIAL
```

The new data device may appear as:

```text
nvme1n1  5G  disk
```

Verify it using its NVMe serial or:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

Match the ID to the EBS volume created in AWS.

After verification:

```bash
export DATA_DEVICE="/dev/nvme1n1"
```

Replace the example path with the correct verified device.

### Step 3 — Inspect Existing Filesystem

```bash
lsblk -f

sudo blkid "$DATA_DEVICE" || true

sudo file -s "$DATA_DEVICE"
```

Expected: new blank EBS volume.

### Step 4 — Create ext4

**Destructive command: Run only on the verified blank Lab 07 data volume.**

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

lsblk -f
```

### Step 6 — Create Important Data

```bash
echo "EBS Lab 07 - Important Application Data" \
  | sudo tee /data/important-data.txt
```

Record its checksum:

```bash
sha256sum /data/important-data.txt
```

### Step 7 — Verify Healthy Baseline

```bash
cat /data/important-data.txt

findmnt /data

df -hT /data
```

Expected: the file is readable and the filesystem is mounted.

---

## 10. Fault Injection — Filesystem Busy

### Step 1 — Enter the Mount Directory

Inside EC2:

```bash
cd /data
```

Verify:

```bash
pwd
```

Expected:

```text
/data
```

The shell is now using the mounted filesystem as its current working directory.

### Step 2 — Attempt to Unmount

```bash
sudo umount /data
```

Expected:

```text
umount: /data: target is busy.
```

The unmount fails because the current shell holds an active filesystem reference.

### Step 3 — Verify Mount Remains Active

```bash
findmnt /data

df -hT /data
```

Expected: `/data` remains mounted.

### Step 4 — Verify AWS EBS State

From another WSL2 terminal:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{State:State,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Expected:

```text
State: in-use
Attachment State: attached
```

### Fault Summary

| Check | Expected Finding |
|---|---|
| EC2 | Running |
| EBS State | In-use |
| Attachment | Attached |
| Linux Filesystem | Mounted |
| `/data` | In use |
| Unmount Attempt | Failed |
| Linux Error | `target is busy` |
| Important Data | Still accessible |
| Root Cause | Active shell using mount |

---

## 11. Troubleshooting Investigation

### Check 1 — Identify Active Filesystem Users

Inside EC2, while the shell remains in `/data`:

```bash
sudo fuser -vm /data
```

Example:

```text
                     USER        PID ACCESS COMMAND
/data:               ssm-user   1234 ..c.. bash
```

The actual username, PID, and command may differ.

The `c` access indicator means the process has its current working directory inside the filesystem.

### Check 2 — Investigate the Blocking Process

Use the PID reported by `fuser`:

```bash
ps -fp 1234
```

Replace `1234` with the actual PID.

Expected: a shell process using `/data`.

### Check 3 — Inspect Mount Details

```bash
findmnt -o SOURCE,TARGET,FSTYPE,OPTIONS /data
```

Expected: ext4 mounted at `/data`.

### Check 4 — Inspect Open Files

If `lsof` is available:

```bash
sudo lsof +D /data
```

This can identify processes holding files or directories open.

`lsof +D` recursively searches directories and may be expensive on large production filesystems.

### Check 5 — Inspect Linux Block Device

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
```

Finding: the EBS block device is still connected to the EC2 instance.

### Check 6 — Inspect AWS Attachment State

From WSL2:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,State:State,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Finding: AWS correctly reports the volume as attached.

### Check 7 — Verify Original Data

Inside EC2:

```bash
cat /data/important-data.txt

sha256sum /data/important-data.txt
```

Expected: original file remains unchanged.

### Investigation Summary

| Investigation | Finding |
|---|---|
| EC2 | Running |
| EBS Volume | In-use |
| Attachment | Attached |
| Linux Filesystem | Mounted |
| Unmount Operation | Failed |
| Error | Target is busy |
| Blocking Process | Shell working in `/data` |
| Data | Preserved |
| Root Cause | Active filesystem reference |

---

## 12. Root Cause Analysis

### Root Cause

The Linux filesystem mounted at `/data` was actively used by a running shell process.

Because the shell's working directory was located inside the filesystem, Linux prevented normal unmounting.

The EBS volume itself was healthy and remained attached.

### Technical Explanation

Linux keeps track of filesystems that are actively referenced by processes.

A process may prevent an unmount when it has:

- Its current directory inside the filesystem.
- An open file on the filesystem.
- An active directory or resource reference.
- Ongoing application I/O.

In our lab, the active shell's current directory was `/data`.

### Root Cause Flow

```text
EBS Attached
      |
      v
ext4 Mounted at /data
      |
      v
Shell Changes Directory to /data
      |
      v
Active Filesystem Reference
      |
      v
sudo umount /data
      |
      X
target is busy
      |
      v
Safe Detachment Deferred
```

### Five Whys

1. **Why did unmounting fail?** The filesystem was busy.
2. **Why was it busy?** A shell process held an active directory reference.
3. **Why did Linux block unmounting?** Normal unmounting requires active filesystem references to be released.
4. **Why shouldn't EBS be detached immediately?** Doing so while the filesystem is mounted can risk data loss or corruption.
5. **How can the issue be prevented?** Identify active users, release the filesystem, flush writes, unmount, and detach normally.

### Prevention

- Stop applications gracefully before maintenance.
- Check active processes with `fuser`.
- Inspect open files when necessary.
- Unmount filesystems before EBS detachment.
- Avoid force-detach except as a last resort.
- Monitor EBS attachment transitions.
- Document storage maintenance procedures.

---

## 13. Remediation — Release and Unmount

### Step 1 — Leave the Mounted Directory

Inside EC2:

```bash
cd /
```

Verify:

```bash
pwd
```

Expected:

```text
/
```

### Step 2 — Recheck Active Processes

```bash
sudo fuser -vm /data
```

Confirm no unexpected processes remain using the filesystem.

Do not terminate unrelated processes.

### Step 3 — Flush Pending Writes

```bash
sync
```

### Step 4 — Unmount Safely

```bash
sudo umount /data
```

Expected: successful unmount without an error.

### Step 5 — Verify Mount Removal

```bash
findmnt /data

lsblk -f
```

Expected: `/data` is no longer mounted, but the EBS block device remains visible.

### Step 6 — Exit EC2

```bash
exit
```

### Step 7 — Detach EBS Normally

From WSL2:

```bash
aws ec2 detach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

**Do not use `--force`.**

### Step 8 — Wait for Detachment

```bash
aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 9 — Verify AWS State

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{State:State,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Expected:

```text
State: available
Attachments: []
```

### Remediation Summary

| Check | During Incident | After Remediation |
|---|---|---|
| Active Process | Using `/data` | Released |
| `umount` | Failed | Successful |
| Linux Filesystem | Mounted | Unmounted |
| EBS Attachment | Attached | Detached |
| EBS State | In-use | Available |
| Force Detach | Not used | Not needed |
| Original Data | Preserved | Preserved |

---

## 14. Verify Data Integrity After Reattachment

### Step 1 — Reattach EBS

From WSL2:

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

### Step 2 — Reconnect to EC2

```bash
aws ssm start-session \
  --target "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

### Step 3 — Identify the Reattached Device

Inside EC2:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,SERIAL
```

Identify the correct device using its EBS volume serial.

For example:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

After verifying:

```bash
export DATA_DEVICE="/dev/nvme1n1"
```

Replace the example path if necessary.

### Step 4 — Mount Existing Filesystem

```bash
sudo mount "$DATA_DEVICE" /data
```

Do not format the volume again.

### Step 5 — Verify Original Data

```bash
findmnt /data

cat /data/important-data.txt

sha256sum /data/important-data.txt
```

Expected:

```text
EBS Lab 07 - Important Application Data
```

Compare the checksum against the healthy baseline.

### Step 6 — Test New Writes

```bash
echo "EBS detach and reattach successful" \
  | sudo tee /data/recovery-test.txt
```

Verify:

```bash
cat /data/recovery-test.txt
```

Expected: successful read and write.

### Final Verification Matrix

| Check | Fault | After Recovery |
|---|---|---|
| EBS State | In-use | In-use after reattachment |
| Filesystem Busy | Yes | No |
| Unmount Operation | Failed | Succeeded |
| Normal Detachment | Deferred | Completed |
| Reattachment | Not yet tested | Successful |
| ext4 Filesystem | Preserved | Preserved |
| Original File | Present | Present |
| Checksum | Original | Matches baseline |
| New Writes | Not tested during detachment | Successful |

---

## 15. Additional Investigation — AWS EBS Actually Stuck Detaching

Our exercise reproduces a Linux busy-filesystem error.

A real AWS attachment issue may instead involve an attachment stuck in `detaching` or `busy`.

### Check Attachment State

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{VolumeState:State,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

### Interpret AWS States

| Observation | Meaning | Response |
|---|---|---|
| `in-use` / `attached` | Normal attached volume | Inspect Linux usage if detachment is needed |
| `detaching` | Detachment in progress | Monitor status and application activity |
| Attachment `busy` | Detachment cannot complete normally | Investigate filesystem and active I/O |
| `available` | Volume detached | Safe to attach elsewhere if compatible |
| `error` | Storage issue requiring investigation | Preserve evidence and investigate recovery |

### Check Linux Mount Status

```bash
findmnt /data

lsblk -f

sudo fuser -vm /data
```

### Safe Recovery Approach

1. Identify the correct EBS volume and instance.
2. Check AWS attachment state.
3. Check whether the volume is still mounted.
4. Identify processes using the filesystem.
5. Stop affected applications gracefully.
6. Flush pending writes.
7. Unmount normally.
8. Monitor EBS detachment.
9. Escalate a persistent control-plane issue when normal recovery cannot complete.

### Why Avoid Force Detachment?

Force detachment can disconnect a volume while applications or the filesystem still expect it to be available.

Potential consequences include:

- Incomplete writes.
- Lost or inconsistent data.
- Filesystem damage.
- Required filesystem recovery.
- Application downtime.

Force detachment is a last-resort measure and is not performed in this lab.

---

## 16. Production Best Practices

### EBS Detachment Checklist

1. Confirm the correct AWS account and Region.
2. Identify the volume ID and attached instance.
3. Check the volume and attachment states.
4. Identify the Linux block device.
5. Verify the filesystem mount.
6. Determine which applications use the storage.
7. Stop or quiesce application I/O.
8. Check open file references.
9. Flush filesystem writes.
10. Unmount safely.
11. Perform normal AWS detachment.
12. Wait for the volume to become available.
13. Verify original data after reattachment.

### Prevention

- Document EBS mount points.
- Automate safe application shutdown where appropriate.
- Avoid detaching actively mounted data volumes.
- Monitor filesystem and application health.
- Keep backups and EBS snapshots according to recovery requirements.
- Verify storage state after infrastructure maintenance.
- Use appropriate change management for storage operations.
- Avoid unnecessary forced operations.

### Security Considerations

- Use least-privilege IAM policies.
- Keep EC2 security groups restrictive.
- Use Systems Manager instead of open SSH when appropriate.
- Restrict access to application files.
- Verify resource ownership before deleting storage.
- Avoid exposing sensitive data in logs or repository documentation.

---

## 17. Cleanup

**Warning:** Cleanup permanently deletes the dedicated Lab 07 EBS volume and terminates the test EC2 instance.

### Step 1 — Unmount the Data Volume

Inside EC2:

```bash
cd /

findmnt /data

sudo fuser -vm /data
```

Confirm no important processes are using the filesystem.

```bash
sync

sudo umount /data
```

Verify:

```bash
findmnt /data
```

Exit the SSM session:

```bash
exit
```

### Step 2 — Verify Resource Ownership

From Ubuntu WSL2:

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

Confirm the resources carry the expected project tag.

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

### Step 4 — Delete EBS Volume

Only after confirming the data is no longer needed:

```bash
aws ec2 delete-volume \
  --volume-id "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 5 — Verify Root Volume Termination Setting

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].{RootDevice:RootDeviceName,BlockDevices:BlockDeviceMappings}' \
  --output json \
  --region "$AWS_REGION"
```

Confirm `DeleteOnTermination=true` for the root volume.

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

Verify that the IAM role and instance profile are dedicated to Lab 07.

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

### Step 8 — Verify Network Interface Cleanup

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'NetworkInterfaces[].{ID:NetworkInterfaceId,Status:Status}' \
  --output table \
  --region "$AWS_REGION"
```

Wait until the instance network interface has been removed.

### Step 9 — Delete VPC Networking

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

Expected: empty arrays after cleanup completes.

Also confirm that the EC2 instance, root EBS volume, IAM role, and instance profile are removed.

### Cleanup Summary

| Resource | Expected Result |
|---|---|
| Data EBS Volume | Deleted |
| EC2 Instance | Terminated |
| Root EBS Volume | Deleted |
| IAM Role | Deleted |
| Instance Profile | Deleted |
| Security Group | Deleted |
| Route Table | Deleted |
| Subnet | Deleted |
| Internet Gateway | Deleted |
| Lab VPC | Deleted |
| Default VPC | Unchanged |

---

## 18. Interview Questions and Answers

### Q1. What does EBS `in-use` mean?

It means the EBS volume is attached to an EC2 instance. It is not necessarily an error.

### Q2. What causes `umount: target is busy`?

A process has an active reference to a file, directory, or another resource on the mounted filesystem.

### Q3. How do you identify processes using a mounted filesystem?

Use `fuser -vm /data` or `lsof` where appropriate.

### Q4. Why should you unmount before detaching EBS?

Unmounting allows the operating system to release the filesystem safely and reduces the risk of data corruption.

### Q5. What does `sync` do?

It requests that pending filesystem writes be flushed to storage.

### Q6. What is the difference between EBS attachment and mounting?

EBS attachment connects the block device to EC2. Mounting exposes its filesystem through a Linux directory.

### Q7. What is the normal state of an attached EBS volume?

`in-use`.

### Q8. What is the expected state after successful detachment?

`available`, with no active attachments.

### Q9. What does an AWS attachment state of `busy` mean?

It indicates that AWS has encountered a condition preventing normal detachment or completion of the attachment transition.

### Q10. Should force detachment be used routinely?

No. It is a last-resort operation because it can cause data inconsistency or filesystem corruption.

### Q11. How do you verify data after reattachment?

Mount the existing filesystem, read the original files, compare checksums, and test normal writes.

### Q12. How would you explain this lab in an interview?

I created an Amazon Linux EC2 instance with a separate 5 GiB EBS data volume.

I formatted the new disk using ext4, mounted it at `/data`, and created an application test file.

I intentionally kept a shell process inside `/data` and attempted to unmount the filesystem.

Linux returned `target is busy`.

I investigated the issue using `fuser`, `ps`, `findmnt`, and AWS CLI.

After identifying the active shell reference, I changed to a directory outside the mount, flushed pending writes, and successfully unmounted the filesystem.

I then detached the EBS volume normally without forcing the operation.

Finally, I reattached the volume, mounted the existing filesystem, and verified that the original application data remained unchanged.

---

## 19. Repository Structure

```text
07-ebs-volume-stuck-in-use/
├── README.md
└── ec2-trust-policy.json
```

### File Descriptions

| File | Purpose |
|---|---|
| `README.md` | Complete EBS busy-filesystem troubleshooting documentation |
| `ec2-trust-policy.json` | EC2 trust policy for Systems Manager access |

No AWS credentials, private keys, or sensitive application data should be committed.

---

## 20. Lessons Learned

This lab demonstrates the importance of understanding the difference between Linux filesystem state and AWS EBS attachment state.

An EBS volume showing `in-use` is normally attached and may be functioning correctly.

The real problem in this incident was an active shell process holding a reference to the mounted filesystem.

Linux prevented unmounting because the filesystem was still being used.

Using `fuser`, `ps`, and `findmnt`, we identified the process and safely released the mount.

We then performed a normal EBS detachment without using force.

Finally, we reattached the volume and verified that the original data remained intact.

**Key Lesson:** Before detaching an EBS data volume, identify and stop processes using its filesystem, unmount the filesystem safely, and verify the AWS attachment transition. Avoid forced operations unless a properly assessed incident requires them.

---

## 21. Skills Demonstrated

- Amazon EBS provisioning
- EC2 storage administration
- EBS attachment and detachment
- AWS CLI v2
- Linux ext4 filesystem management
- Filesystem busy troubleshooting
- `fuser`
- `ps`
- `findmnt`
- `lsblk`
- `sync`
- `umount`
- Linux process investigation
- NVMe device identification
- SHA-256 data verification
- Systems Manager
- Root cause analysis
- Safe storage remediation
- AWS resource cleanup

## Final Outcome

| Item | Expected Result |
|---|---|
| Incident | INC-EBS-007 |
| Problem | Filesystem busy during EBS detachment preparation |
| EBS Volume | 5 GiB gp3 |
| Initial EBS State | In-use |
| Filesystem | ext4 |
| Mount | `/data` |
| Fault | Shell process using filesystem |
| Error | `target is busy` |
| Root Cause | Active filesystem reference |
| Remediation | Release process, unmount, detach |
| Forced Detachment | Not used |
| Detached Volume State | Available |
| Reattachment | Successful |
| Original Data | Preserved |
| Checksum Verification | Matches baseline |
| Cleanup | Temporary resources removed |
| Expected Incident Status | Resolved |

**Key Takeaway:** Safe EBS detachment requires understanding both AWS attachment states and Linux filesystem usage. The correct recovery process is to identify active filesystem users, release them safely, unmount the filesystem, and detach the volume normally.
