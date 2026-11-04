# Lab 02 — Linux Volume Not Mounted

## Overview

This lab demonstrates how to troubleshoot an Amazon EBS volume that is successfully attached to an EC2 instance but whose filesystem is not mounted in Linux.

We deploy an Amazon Linux 2023 EC2 instance and attach a separate 5 GiB `gp3` EBS volume.

After identifying the correct Linux block device, we create an ext4 filesystem, mount it at `/data`, and write a sample application file.

We then intentionally unmount `/data` to simulate a production storage failure.

The EBS volume remains attached and its data remains intact, but the application can no longer access its expected files.

We investigate the issue using Linux tools, restore the filesystem without formatting it again, and configure a persistent mount using the filesystem UUID in `/etc/fstab`.

Finally, we verify recovery and safely remove the temporary AWS infrastructure.

**Troubleshooting workflow:**

Provision → Healthy Baseline → Fault Injection → Investigation → Root Cause Analysis → Remediation → Verification → Cleanup

---

## 1. Lab Information

| Item | Configuration |
|---|---|
| Incident ID | INC-EBS-002 |
| Difficulty | Beginner |
| AWS Region | `us-east-1` |
| AWS Services | EC2, EBS, VPC, IAM, Systems Manager |
| Operating System | Amazon Linux 2023 |
| Instance Type | `t3.micro` |
| EBS Volume | 5 GiB `gp3` |
| AWS Attachment | `/dev/sdf` |
| Linux Device | NVMe device, such as `/dev/nvme1n1` |
| Filesystem | ext4 |
| Mount Directory | `/data` |
| Fault Injection | Manually unmount `/data` |
| Primary Symptom | Application file cannot be found |
| Root Cause | EBS filesystem not mounted |
| Remediation | Remount existing filesystem |
| Permanent Fix | UUID-based `/etc/fstab` entry |
| Verification | Original file and checksum preserved |

---

## 2. Production Incident Scenario

### Incident Description

A company hosts an application on an EC2 instance.

The application stores important data on a separate Amazon EBS volume mounted at `/data`.

During a system maintenance operation, the filesystem is unmounted.

The EC2 instance remains running, and AWS reports that the EBS volume is still attached with the state `in-use`.

However, the application can no longer find its data files.

The cloud engineer must investigate why the attached EBS volume is unavailable inside Linux.

### Incident Symptoms

- EC2 instance is running.
- EBS volume state is `in-use`.
- EBS attachment state is `attached`.
- Linux detects the block device.
- The ext4 filesystem still exists.
- `/data` is not mounted.
- Previously saved application files are unavailable through `/data`.
- No persistent mount entry was initially configured.

### Business Impact

- Application cannot access required files.
- File-based applications may fail.
- Data processing operations may stop.
- Application logs may be written to an unexpected filesystem.
- Customers may experience application errors.
- Engineers may incorrectly assume the EBS volume is missing or damaged.

### Objective

Identify the missing filesystem mount, restore access to existing application data, and configure persistent mounting without destroying or recreating the filesystem.

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
                 Linux Block Device
                     /dev/nvme1n1
                          |
                          v
                   ext4 Filesystem
                          |
                          v
                   Mounted at /data
                          |
                          v
                 /data/test-file.txt
                          |
                          v
                    File Accessible
```

### Broken Architecture

```text
                    Amazon EC2
                          |
                          v
                   EBS Attached
                   State: in-use
                          |
                          v
                 Linux Device Exists
                          |
                          v
                  ext4 Filesystem
                          |
                          X
                  Filesystem Unmounted
                          |
                          v
                 /data Not Mounted
                          |
                          v
               Application File Missing
```

### Recovered Architecture

```text
EBS Volume
     |
     v
Linux NVMe Device
     |
     v
Existing ext4 Filesystem
     |
     v
Mount Using Filesystem UUID
     |
     v
/data
     |
     v
Original Test File Accessible
     |
     v
Persistent Mount via /etc/fstab
```

---

## 4. Understanding EBS Attachment and Linux Mounting

Amazon EBS provides persistent block storage for EC2 instances.

Attaching an EBS volume makes a block device available to Linux.

However, attachment does not automatically mean that the filesystem is mounted.

### Attachment vs. Mounting

| Operation | Description |
|---|---|
| Attach | Connects an EBS volume to EC2 |
| Format | Creates a filesystem on a block device |
| Mount | Makes a filesystem accessible through a directory |
| Unmount | Removes the filesystem from the directory tree |
| Detach | Disconnects the EBS volume from EC2 |

### Example

An EBS volume may show:

```text
AWS Volume State: in-use
AWS Attachment State: attached
```

But Linux may show:

```text
/dev/nvme1n1  5G  ext4
```

with no mount point.

This means the volume is attached and contains a filesystem, but its files are not accessible through the intended directory.

### Understanding `/etc/fstab`

`/etc/fstab` is a Linux configuration file that defines filesystems and their mount settings.

A typical persistent entry is:

```text
UUID=<filesystem-uuid> /data ext4 defaults,nofail 0 2
```

- `UUID` identifies the filesystem.
- `/data` is the mount directory.
- `ext4` is the filesystem type.
- `defaults` enables standard mount options.
- `nofail` allows boot to continue if this optional volume is unavailable.
- `0` disables the legacy dump backup flag.
- `2` configures filesystem check ordering.

Using a UUID is safer than relying on an NVMe device name that might change.

---

## 5. Prerequisites and Safety

### Requirements

- AWS account
- AWS CLI v2
- Ubuntu WSL2
- Bash
- Amazon EC2 permissions
- Amazon EBS permissions
- IAM permissions
- AWS Systems Manager
- Git and GitHub
- Basic Linux knowledge

### Safety Requirements

**Important: Storage operations can permanently destroy data.**

- Use a new disposable EBS volume.
- Verify the EBS volume ID before formatting.
- Never format the EC2 root filesystem.
- Never run `mkfs` on an existing data volume during recovery.
- Unmount filesystems safely.
- Preserve the filesystem UUID when restoring access.
- Back up `/etc/fstab` before editing.
- Test `/etc/fstab` before rebooting.
- Verify the AWS account and resource IDs before cleanup.

### Cost Considerations

EC2 instances, EBS volumes, root storage, public IPv4 addresses, and applicable network traffic may incur AWS charges.

---

## 6. Prepare the Environment

### Step 1 — Set Environment Variables

Run in Ubuntu WSL2:

```bash
export AWS_REGION="us-east-1"
export PROJECT_TAG="EBS-Troubleshooting-Lab02"

export INSTANCE_NAME="cloud-ebs-lab02-ec2"
export VOLUME_NAME="cloud-ebs-lab02-data"

export ROLE_NAME="cloud-ebs-lab02-ssm-role"
export PROFILE_NAME="cloud-ebs-lab02-ssm-profile"

export SG_NAME="cloud-ebs-lab02-sg"

export VOLUME_SIZE="5"
export AZ_A="us-east-1a"
```

### Step 2 — Verify AWS Credentials

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

Select a supported Availability Zone.

### Step 4 — Check Existing Lab Resources

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

Do not create duplicate billable resources.

---

## 7. Create EC2 Infrastructure

### Step 1 — Create VPC

```bash
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block "10.173.0.0/16" \
  --tag-specifications \
  "ResourceType=vpc,Tags=[{Key=Name,Value=cloud-ebs-lab02-vpc},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --cidr-block "10.173.1.0/24" \
  --availability-zone "$AZ_A" \
  --tag-specifications \
  "ResourceType=subnet,Tags=[{Key=Name,Value=cloud-ebs-lab02-subnet},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 3 — Create Internet Gateway

```bash
export IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications \
  "ResourceType=internet-gateway,Tags=[{Key=Name,Value=cloud-ebs-lab02-igw},{Key=Project,Value=$PROJECT_TAG}]" \
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
  "ResourceType=route-table,Tags=[{Key=Name,Value=cloud-ebs-lab02-rt},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --description "Lab02 EBS Linux mounting" \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
  "ResourceType=security-group,Tags=[{Key=Name,Value=$SG_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query GroupId \
  --output text \
  --region "$AWS_REGION")
```

No inbound rules are needed because the instance will be accessed using Systems Manager.

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

Check that the names are unused.

```bash
aws iam get-role --role-name "$ROLE_NAME"

aws iam get-instance-profile \
  --instance-profile-name "$PROFILE_NAME"
```

Create the role and profile:

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

### Step 10 — Wait for EC2 Readiness

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

### Step 1 — Create a New 5 GiB gp3 Volume

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

### Step 2 — Wait for Volume Availability

```bash
aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 3 — Verify EBS Volume

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[].{Volume:VolumeId,State:State,AZ:AvailabilityZone,Size:Size}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
State: available
Size: 5 GiB
AZ: Same as EC2
```

### Step 4 — Attach Volume

```bash
aws ec2 attach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdf \
  --region "$AWS_REGION"
```

### Step 5 — Wait for Attachment

```bash
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

Expected:

```text
Volume State: in-use
Attachment State: attached
AWS Device: /dev/sdf
```

---

## 9. Create the Linux Filesystem

### Step 1 — Connect to EC2

Check SSM:

```bash
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
  --query 'InstanceInformationList[].{ID:InstanceId,Status:PingStatus}' \
  --output table \
  --region "$AWS_REGION"
```

Connect once online:

```bash
aws ssm start-session \
  --target "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

The next Linux commands run inside EC2.

### Step 2 — Inspect Block Devices

```bash
lsblk

lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,SERIAL
```

Possible output:

```text
NAME          SIZE  TYPE MOUNTPOINTS
nvme0n1         8G  disk
└─nvme0n1p1     8G  part /
nvme1n1         5G  disk
```

The exact names may differ.

### Step 3 — Identify the Correct EBS Volume

Compare the disk serial number with the EBS volume ID.

For supported Amazon Linux instances:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

Replace the example device with the identified lab volume.

After verification:

```bash
export DATA_DEVICE="/dev/nvme1n1"
```

Never proceed with formatting until you are certain this is the disposable 5 GiB EBS volume.

### Step 4 — Inspect Existing Filesystem

```bash
lsblk -f

sudo blkid "$DATA_DEVICE" || true

sudo file -s "$DATA_DEVICE"
```

Expected on a new empty volume:

```text
No filesystem
No existing mount
```

### Step 5 — Create ext4 Filesystem

**Destructive command: Only for the verified, empty Lab 02 volume.**

```bash
sudo mkfs.ext4 "$DATA_DEVICE"
```

Do not run this on an existing filesystem or the EC2 root disk.

### Step 6 — Verify Filesystem

```bash
sudo blkid "$DATA_DEVICE"

lsblk -f
```

Expected: ext4 filesystem with a UUID.

### Step 7 — Create Mount Directory

```bash
sudo mkdir -p /data
```

### Step 8 — Mount Volume

```bash
sudo mount "$DATA_DEVICE" /data
```

Verify:

```bash
findmnt /data

df -hT /data
```

Expected: `/data` is backed by the ext4 filesystem on the lab EBS volume.

### Step 9 — Create Application Test Data

```bash
echo "EBS Lab 02 - Application Data" | sudo tee /data/test-file.txt
```

Verify:

```bash
cat /data/test-file.txt

sha256sum /data/test-file.txt
```

Record the checksum for later comparison.

---

## 10. Healthy Baseline

| Component | Expected Result |
|---|---|
| EC2 | Running |
| EBS Volume | In-use |
| EBS Attachment | Attached |
| Linux Device | Visible |
| Filesystem | ext4 |
| Mount Point | `/data` |
| Mount State | Mounted |
| Application File | Accessible |
| Test Data | Preserved |

Verify:

```bash
lsblk -f

findmnt /data

df -hT /data

cat /data/test-file.txt
```

All checks should succeed before injecting the failure.

---

## 11. Fault Injection — Unmount the Filesystem

### Step 1 — Unmount the Data Volume

Inside EC2:

```bash
cd /

sudo umount /data
```

This unmounts the filesystem but does not detach the EBS volume.

### Step 2 — Verify Mount Is Missing

```bash
findmnt /data
```

Expected: no mounted filesystem at `/data`.

Check:

```bash
lsblk -f
```

The ext4 block device should still be present.

### Step 3 — Reproduce the Application Failure

```bash
cat /data/test-file.txt
```

Expected:

```text
cat: /data/test-file.txt: No such file or directory
```

This does not mean the test file was deleted.

The filesystem containing the file is no longer mounted.

### Step 4 — Verify AWS Attachment

From a separate WSL2 terminal:

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
Attachment: attached
```

### Fault Summary

| Check | Finding |
|---|---|
| EC2 | Running |
| EBS | Attached |
| EBS State | In-use |
| Linux Device | Present |
| Filesystem | ext4 |
| `/data` Mount | Missing |
| Application File | Inaccessible |
| Root Cause | Filesystem unmounted |

---

## 12. Troubleshooting Investigation

### Check 1 — Inspect Linux Block Devices

Inside EC2:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,SERIAL
```

Finding: the EBS device exists.

### Check 2 — Inspect Filesystem Metadata

```bash
sudo blkid "$DATA_DEVICE"

sudo file -s "$DATA_DEVICE"
```

Finding: the existing ext4 filesystem is present.

### Check 3 — Verify Active Mounts

```bash
findmnt /data
```

Finding: `/data` is not mounted.

### Check 4 — Inspect Disk Usage

```bash
df -hT
```

Finding: the data volume is not listed as mounted at `/data`.

### Check 5 — Inspect Mount Directory

```bash
ls -la /data
```

Finding: Linux shows the underlying directory rather than the EBS filesystem's contents.

### Check 6 — Inspect `/etc/fstab`

```bash
cat /etc/fstab

grep -nE '[[:space:]]/data[[:space:]]' /etc/fstab || true
```

Finding: there is no configured persistent mount entry for `/data` in our initial setup.

### Check 7 — Verify AWS Attachment

From WSL2:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{State:State,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Finding: the EBS volume is still attached.

### Investigation Summary

| Investigation | Finding |
|---|---|
| EC2 Instance | Running |
| EBS Attachment | Correct |
| Linux Device | Present |
| ext4 Filesystem | Present |
| Active Mount | Missing |
| `/etc/fstab` Entry | Missing |
| Existing Data | Still on EBS |
| Root Cause | EBS filesystem unmounted |

---

## 13. Root Cause Analysis

### Root Cause

The EBS volume was correctly attached to the EC2 instance, but the Linux filesystem was unmounted from `/data`.

Therefore, the application could no longer access its stored files through the expected directory.

The original data remained on the EBS volume.

### Technical Explanation

A block device and filesystem can exist without being mounted.

When the filesystem is unmounted, Linux no longer exposes its files through the mount directory.

The AWS EBS attachment continues to exist independently.

### Root Cause Flow

```text
EBS Volume Attached
        |
        v
ext4 Filesystem Created
        |
        v
Mounted at /data
        |
        v
Application Data Written
        |
        v
Filesystem Unmounted
        |
        v
/data Mount Missing
        |
        v
Application File Unavailable
```

### Five Whys

1. **Why could the application not find its data?** `/data` was not mounted.
2. **Why was the mount missing?** The ext4 filesystem was manually unmounted.
3. **Why did AWS still show the volume as attached?** EBS attachment and Linux mounting are separate operations.
4. **Why was the filesystem not configured for automatic mounting?** No `/etc/fstab` entry had been created.
5. **How can recurrence be reduced?** Use a validated UUID-based persistent mount configuration and monitor required mount points.

### Important Safety Lesson

Do not run `mkfs.ext4` to fix a missing mount.

Running `mkfs` again can destroy existing filesystem metadata and data.

The correct action is to identify and mount the existing filesystem.

---

## 14. Remediation — Restore the Filesystem

### Step 1 — Mount the Existing Filesystem

Inside EC2:

```bash
sudo mount "$DATA_DEVICE" /data
```

### Step 2 — Verify Mount Recovery

```bash
findmnt /data

df -hT /data
```

Expected: `/data` is mounted successfully.

### Step 3 — Verify Original Data

```bash
cat /data/test-file.txt

sha256sum /data/test-file.txt
```

Expected:

```text
EBS Lab 02 - Application Data
```

The checksum should match the original value.

### Step 4 — Retrieve Filesystem UUID

```bash
export DATA_UUID=$(sudo blkid -s UUID -o value "$DATA_DEVICE")

echo "$DATA_UUID"
```

### Step 5 — Back Up `/etc/fstab`

```bash
sudo cp -a /etc/fstab /etc/fstab.lab02.backup
```

### Step 6 — Check for Existing Mount Configuration

```bash
grep -nE '[[:space:]]/data[[:space:]]' /etc/fstab || true
```

Review any existing entry before making changes. Avoid duplicate `/data` entries.

### Step 7 — Add Persistent Mount

For the dedicated lab filesystem:

```bash
echo "UUID=$DATA_UUID /data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
```

Verify:

```bash
tail -n 5 /etc/fstab
```

Expected format:

```text
UUID=<your-filesystem-uuid> /data ext4 defaults,nofail 0 2
```

### Step 8 — Test `/etc/fstab`

Unmount:

```bash
cd /

sudo umount /data
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Test the mount configuration:

```bash
sudo mount -a
```

Verify:

```bash
findmnt /data

cat /data/test-file.txt
```

Expected: mount succeeds and the original test file remains accessible.

If `mount -a` fails, investigate the UUID, filesystem type, and mount options before rebooting.

### Step 9 — Optional Reboot Test

After validating `/etc/fstab`:

```bash
sudo reboot
```

Reconnect through Systems Manager after EC2 becomes available.

Verify:

```bash
findmnt /data

df -hT /data

cat /data/test-file.txt
```

Expected: `/data` is automatically mounted and the file is accessible.

---

## 15. Final Verification

### Verify Linux Filesystem

Inside EC2:

```bash
lsblk -f

findmnt /data

df -hT /data

cat /data/test-file.txt

sha256sum /data/test-file.txt
```

### Verify AWS EBS State

From WSL2:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{Volume:VolumeId,State:State,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

### Verification Matrix

| Check | During Incident | After Remediation |
|---|---|---|
| EC2 | Running | Running |
| EBS | In-use | In-use |
| Block Device | Present | Present |
| Filesystem | ext4 | ext4 |
| Mount Point | Missing | `/data` |
| Test File | Inaccessible | Accessible |
| Data Integrity | Preserved | Verified |
| Persistent Mount | Missing | Configured |
| Incident | Active | Resolved |

### Success Criteria

- EC2 created successfully.
- EBS volume attached.
- Correct Linux block device identified.
- ext4 filesystem created on a verified empty disk.
- `/data` mounted.
- Test file created.
- Filesystem intentionally unmounted.
- Missing mount diagnosed.
- Existing filesystem remounted.
- Original data verified.
- Persistent mount configured.
- `/etc/fstab` tested.
- Temporary resources cleaned up.

---

## 16. Production Best Practices

### Linux EBS Mount Troubleshooting Checklist

1. Verify EC2 state.
2. Confirm EBS attachment.
3. Inspect Linux block devices.
4. Identify the correct EBS volume.
5. Check filesystem type.
6. Verify active mount points.
7. Inspect `/etc/fstab`.
8. Review filesystem UUIDs.
9. Mount the existing filesystem.
10. Verify application data.
11. Configure persistent mounting.
12. Test the configuration before rebooting.

### Common Mount Problems

| Problem | Symptom | Remediation |
|---|---|---|
| EBS not attached | Device missing | Investigate AWS attachment |
| Filesystem not mounted | Directory missing data | Mount filesystem |
| Wrong device selected | Incorrect or missing files | Identify correct EBS volume |
| Incorrect UUID | Boot-time mount failure | Correct `/etc/fstab` |
| Wrong filesystem type | Mount error | Verify using `blkid` |
| Mount directory missing | Mount failure | Create directory |
| Filesystem corruption | Mount errors | Investigate safely, preserve data |
| Missing persistent configuration | Mount absent after reboot | Configure `/etc/fstab` |

### Prevention

- Use filesystem UUIDs instead of changing NVMe device names.
- Validate `/etc/fstab` with `mount -a`.
- Use `nofail` for optional EBS data volumes where appropriate.
- Avoid manual unmounting during application activity.
- Monitor required application mount points.
- Create snapshots before risky storage operations.
- Use infrastructure automation for repeatable deployments.
- Verify data integrity after recovery.

### Security Considerations

- Use Systems Manager instead of public SSH where appropriate.
- Keep security groups restrictive.
- Apply least-privilege IAM permissions.
- Enable EBS encryption as required by the environment.
- Protect filesystem permissions.
- Avoid exposing sensitive application data.
- Maintain tested backup and recovery procedures.

---

## 17. Cleanup

**Warning:** Cleanup permanently deletes the disposable EBS data volume and terminates the lab EC2 instance. Verify ownership before proceeding.

### Step 1 — Unmount the Volume

Inside EC2:

```bash
cd /

findmnt /data

sudo fuser -vm /data
```

Ensure no active processes need the filesystem.

Unmount:

```bash
sudo umount /data
```

### Step 2 — Restore `/etc/fstab`

Restore the lab backup:

```bash
sudo cp -a /etc/fstab.lab02.backup /etc/fstab

sudo systemctl daemon-reload
```

Verify:

```bash
grep -nE '[[:space:]]/data[[:space:]]' /etc/fstab || true

findmnt /data
```

The Lab 02 mount entry should no longer exist, and `/data` should not be mounted.

Exit the EC2 session.

### Step 3 — Verify AWS Resource Ownership

From WSL2:

```bash
aws sts get-caller-identity

aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{Volume:VolumeId,State:State,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"

aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"
```

Confirm `Project=EBS-Troubleshooting-Lab02`.

### Step 4 — Detach the Data Volume

```bash
aws ec2 detach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

Wait:

```bash
aws ec2 wait volume-available \
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

Expected: `available` with no attachments.

### Step 5 — Delete Data Volume

```bash
aws ec2 delete-volume \
  --volume-id "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 6 — Verify Root EBS Deletion Setting

Inspect the EC2 root volume mapping:

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].{RootDevice:RootDeviceName,BlockDevices:BlockDeviceMappings}' \
  --output json \
  --region "$AWS_REGION"
```

Verify that the root volume has `DeleteOnTermination=true`.

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

Verify the IAM role and profile are dedicated to the lab.

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

### Step 9 — Check Network Interfaces

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'NetworkInterfaces[].{ID:NetworkInterfaceId,Status:Status}' \
  --output table \
  --region "$AWS_REGION"
```

Wait until the lab's EC2 network interfaces have been removed.

### Step 10 — Delete VPC Networking

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

Expected: no remaining tagged volumes or VPC resources.

Also verify that the EC2 instance, root volume, IAM role, and instance profile have been removed.

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
| VPC | Deleted |
| Default VPC | Unchanged |

---

## 18. Interview Questions and Answers

### Q1. What is the difference between attaching and mounting an EBS volume?

Attaching connects an EBS block device to EC2. Mounting makes the filesystem available through a Linux directory.

### Q2. Why might EBS show `in-use` while its data is unavailable?

The volume may be attached but not mounted, mounted at the wrong directory, or affected by filesystem problems.

### Q3. Which command lists Linux block devices?

`lsblk`.

### Q4. Which command checks whether a filesystem is mounted?

`findmnt`.

### Q5. What does `blkid` do?

It displays block device filesystem information, including UUIDs and filesystem types.

### Q6. What is `/etc/fstab`?

A Linux configuration file used to define filesystem mounting settings.

### Q7. Why use UUIDs in `/etc/fstab`?

Filesystem UUIDs are more stable than NVMe device names, which may change after reboot.

### Q8. What does `nofail` mean?

It allows startup to continue if an optional filesystem cannot be mounted.

### Q9. Does `umount` delete EBS data?

No. It removes the filesystem from the active directory tree without deleting its data.

### Q10. Does `umount` detach EBS from AWS?

No. Linux unmounting and AWS EBS detachment are separate operations.

### Q11. Why should you avoid `mkfs` during mount recovery?

It can overwrite existing filesystem structures and destroy data. Mount the existing filesystem instead.

### Q12. How would you explain this lab in an interview?

I deployed an Amazon EC2 instance with a separate EBS data volume.

I created an ext4 filesystem, mounted it at `/data`, and stored a test file.

Then I intentionally unmounted the filesystem to simulate an application storage failure.

The EBS volume remained attached in AWS, but the application file was unavailable.

I investigated the block device, filesystem UUID, mount state, and `/etc/fstab` configuration using Linux commands.

I restored the filesystem without formatting it again, verified that the original data was preserved, and configured a UUID-based persistent mount.

---

## 19. Repository Structure

```text
02-linux-volume-not-mounted/
├── README.md
└── ec2-trust-policy.json
```

### File Descriptions

| File | Purpose |
|---|---|
| `README.md` | Complete filesystem mounting troubleshooting documentation |
| `ec2-trust-policy.json` | IAM trust policy for Systems Manager access |

Do not commit AWS credentials, private keys, or sensitive information.

---

## 20. Lessons Learned

This lab demonstrates that an EBS volume can be attached correctly in AWS but still be unavailable to applications because its filesystem is not mounted.

The incident was not caused by an EC2 failure, EBS attachment error, or missing data.

The filesystem had simply been unmounted.

Using `lsblk`, `blkid`, `findmnt`, and `df`, we identified the existing filesystem and confirmed that its mount point was missing.

The filesystem was remounted without formatting or recreating it, and the original test file was recovered.

We also configured `/etc/fstab` using the filesystem UUID to support persistent mounting.

**Key Lesson:** An EBS volume showing `in-use` does not guarantee that its filesystem is mounted or accessible. Always verify both AWS attachment status and Linux mount state.

---

## 21. Skills Demonstrated

- Amazon EBS provisioning
- EBS attachment
- Amazon EC2
- Linux block device identification
- NVMe device mapping
- ext4 filesystem creation
- Linux filesystem mounting
- Linux filesystem unmounting
- `lsblk`
- `blkid`
- `findmnt`
- `df`
- `/etc/fstab`
- Filesystem UUID configuration
- Data integrity verification
- Systems Manager
- Root cause analysis
- Safe remediation
- AWS resource cleanup

## Final Outcome

| Item | Expected Result |
|---|---|
| Incident | INC-EBS-002 |
| Problem | Linux EBS filesystem not mounted |
| Root Cause | Missing active mount at `/data` |
| EBS State | In-use |
| EC2 State | Running |
| Filesystem | ext4 |
| Fault | Unmounted filesystem |
| Remediation | Mount existing ext4 filesystem |
| Persistent Fix | UUID-based `/etc/fstab` |
| Data Integrity | Original file preserved |
| Final Mount | `/data` |
| Cleanup | Temporary resources removed |
| Expected Incident Status | Resolved |

**Key Takeaway:** Amazon EBS attachment and Linux filesystem mounting are independent. Reliable storage recovery requires identifying the correct block device, preserving existing data, and restoring the intended mount configuration.
