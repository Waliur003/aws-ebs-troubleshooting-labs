# Lab 10 — EBS Multi-Attach Misconfiguration

## Overview

This lab demonstrates how to troubleshoot an Amazon EBS volume attachment failure caused by attempting to attach a standard gp3 volume to two EC2 instances at the same time.

We deploy two Amazon Linux 2023 EC2 instances in the same Availability Zone and create a dedicated 5 GiB gp3 EBS volume.

We attach the volume to EC2-A, create an ext4 filesystem, mount it at `/data`, and store sample application files.

Next, we intentionally attempt to attach the same gp3 volume to EC2-B while it is still attached to EC2-A.

AWS rejects the operation because gp3 volumes do not support EBS Multi-Attach.

We investigate the EBS volume type, `MultiAttachEnabled` setting, Availability Zone placement, and existing volume attachment using AWS CLI.

After identifying the root cause, we safely unmount and detach the volume from EC2-A, attach it to EC2-B, and verify that the original files remain intact.

We also compare standard EBS, io2 Multi-Attach, and Amazon EFS to understand appropriate shared-storage architectures.

Finally, we clean up all temporary AWS resources.

**Troubleshooting workflow:**

Provision → Create Data → Fault Injection → Investigation → Root Cause Analysis → Safe Detachment → Reattachment → Verification → Cleanup

---

## 1. Lab Information

| Item | Configuration |
|---|---|
| Incident ID | INC-EBS-010 |
| Difficulty | Intermediate–Advanced |
| AWS Region | `us-east-1` |
| AWS Services | EC2, EBS, VPC, IAM, Systems Manager |
| Operating System | Amazon Linux 2023 |
| EC2 Instances | Two `t3.micro` instances |
| EBS Volume | 5 GiB gp3 |
| Availability Zone | Same AZ for both instances |
| EBS Multi-Attach | Disabled and unsupported on gp3 |
| AWS Device Name | `/dev/sdf` |
| Linux Device | Verified NVMe device |
| Filesystem | ext4 |
| Mount Point | `/data` |
| Fault Injection | Attach gp3 to EC2-B while attached to EC2-A |
| Expected Error | `VolumeInUse` |
| Root Cause | Unsupported simultaneous attachment |
| Remediation | Safely move the volume to EC2-B |
| Data Integrity | SHA-256 checksum |
| Expected Result | Volume attached to EC2-B with original data intact |

---

## 2. Production Incident Scenario

### Incident Description

A company runs an application on two Amazon EC2 instances.

EC2-A stores important application files on an EBS gp3 volume.

A cloud engineer wants EC2-B to access the same files and assumes that an EBS volume can be attached to multiple EC2 instances simultaneously.

The engineer attempts to attach the volume to EC2-B.

AWS returns an error similar to:

```text
An error occurred (VolumeInUse) when calling
the AttachVolume operation:

vol-xxxxxxxx is already attached to an instance.
```

The engineer needs to determine whether the issue is caused by the EBS volume type, Multi-Attach configuration, instance placement, or existing attachment.

### Incident Symptoms

- EC2-A is running.
- EC2-B is running.
- Both instances are in the same Availability Zone.
- EBS volume type is gp3.
- EBS volume state is `in-use`.
- `MultiAttachEnabled` is `false`.
- EBS is attached to EC2-A.
- Attachment attempt to EC2-B fails.
- Original application data remains accessible on EC2-A.

### Business Impact

- Storage migration may be delayed.
- Application failover may fail.
- Shared-storage designs may not work as expected.
- Application recovery may take longer.
- Incorrect detachment procedures may risk filesystem corruption.
- Choosing the wrong storage architecture may increase costs.

### Objective

Identify why the second EBS attachment fails, safely transfer the gp3 volume between EC2 instances, verify data integrity, and understand supported alternatives for concurrent shared storage.

---

## 3. AWS Architecture

### Initial Architecture

```text
                  AWS VPC
                      |
              Public Subnet
                      |
             -----------------
             |               |
             v               v
           EC2-A           EC2-B
             |
             v
          EBS gp3
           5 GiB
             |
             v
       ext4 Filesystem
             |
             v
           /data
             |
             v
       Application Files
```

### Broken Architecture

```text
                 EBS gp3 Volume
                       |
                       v
                Attached EC2-A
                       |
                       v
              Second Attach Request
                     EC2-B
                       |
                       X
                   VolumeInUse
                       |
                       v
             EC2-B Cannot Attach
                       |
                       v
             Multi-Attach Unsupported
```

### Recovered Architecture

```text
                 EBS gp3 Volume
                       |
                       v
                Mounted on EC2-A
                       |
                       v
                Stop Application I/O
                       |
                       v
                Unmount Filesystem
                       |
                       v
                Detach From EC2-A
                       |
                       v
                 State: available
                       |
                       v
                 Attach to EC2-B
                       |
                       v
                Mount Existing ext4
                       |
                       v
                 Verify Original Data
                       |
                       v
                 Recovery Complete
```

### Infrastructure Configuration

| Resource | Configuration |
|---|---|
| VPC | `10.181.0.0/16` |
| Public Subnet | `10.181.1.0/24` |
| Internet Gateway | Attached |
| Route Table | Internet route |
| Security Group | No inbound rules |
| EC2 Management | Systems Manager |
| EC2-A | Amazon Linux 2023 |
| EC2-B | Amazon Linux 2023 |
| Data Volume | 5 GiB gp3 |
| Filesystem | ext4 |
| Mount Directory | `/data` |

---

## 4. Understanding EBS Multi-Attach

Amazon EBS Multi-Attach allows eligible Provisioned IOPS SSD volumes to be attached to multiple supported EC2 instances within the same Availability Zone.

Multi-Attach is supported on eligible io1 and io2 volumes, subject to AWS compatibility and configuration restrictions.

Standard gp3 volumes do not support Multi-Attach.

### EBS Volume Type Comparison

| EBS Type | Multi-Attach Support |
|---|---|
| gp2 | No |
| gp3 | No |
| io1 | Yes, with restrictions |
| io2 | Yes, with restrictions |
| st1 | No |
| sc1 | No |

### Important Multi-Attach Requirements

- The EBS volume must use a supported type.
- The EC2 instances must be compatible.
- All participating instances must be in the same Availability Zone.
- Multi-Attach must be configured on the volume.
- A suitable clustered application or filesystem is required for concurrent write access.
- Multi-Attach alone does not coordinate filesystem writes.

### Why ext4 Cannot Be Shared for Concurrent Writes

Traditional ext4 filesystems are not designed to coordinate independent read-write operations from multiple Linux hosts over a shared block device.

If two instances independently modify filesystem metadata, the result may be corruption or data loss.

For concurrent block-storage access, use an appropriate cluster-aware filesystem and application coordination.

Alternatively, use managed shared file storage such as Amazon EFS when its semantics fit the workload.

### EBS vs. EFS

| Feature | EBS gp3 | EBS io2 Multi-Attach | Amazon EFS |
|---|---|---|---|
| Storage Type | Block | Block | Network file |
| Simultaneous EC2 Access | No | Supported with restrictions | Yes |
| Filesystem Management | Linux | Cluster-aware design required | Managed shared filesystem |
| Same AZ Requirement | For attachment | Yes | Supports regional access patterns |
| Typical Use | Application data disk | Clustered block-storage workloads | Shared files and directories |

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
- AWS Systems Manager
- Git and GitHub

### Safety Rules

- Use only dedicated lab instances and EBS volumes.
- Verify EBS volume IDs before attachment or detachment.
- Verify the correct Linux NVMe device.
- Never format the EC2 root volume.
- Do not force-attach an unsupported volume.
- Do not simultaneously mount ordinary ext4 read-write across multiple hosts.
- Stop application writes before detachment.
- Unmount the filesystem before detaching.
- Do not format the transferred EBS volume.
- Verify SHA-256 checksums after recovery.
- Delete only dedicated lab resources.

### Cost Considerations

Two EC2 instances, their root EBS volumes, the additional gp3 data volume, and public IPv4 addresses may incur AWS charges.

The optional io2 Multi-Attach architecture is explained but not provisioned in this lab to avoid unnecessary expenses.

---

## 6. Prepare the AWS Environment

### Step 1 — Configure Variables

Run in Ubuntu WSL2:

```bash
export AWS_REGION="us-east-1"
export PROJECT_TAG="EBS-Troubleshooting-Lab10"

export AZ_A="us-east-1a"
export INSTANCE_TYPE="t3.micro"

export ROLE_NAME="cloud-ebs-lab10-ssm-role"
export PROFILE_NAME="cloud-ebs-lab10-ssm-profile"

export SG_NAME="cloud-ebs-lab10-sg"
export VOLUME_NAME="cloud-ebs-lab10-data"
```

### Step 2 — Verify AWS Identity

```bash
aws sts get-caller-identity

aws --version
```

### Step 3 — Verify AZ Availability

```bash
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters "Name=instance-type,Values=$INSTANCE_TYPE" \
  --query 'InstanceTypeOfferings[].Location' \
  --output table \
  --region "$AWS_REGION"
```

Choose a supported AZ if necessary.

### Step 4 — Check Existing Resources

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=$PROJECT_TAG" \
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

## 7. Create AWS Infrastructure

### Step 1 — Create VPC

```bash
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.181.0.0/16 \
  --tag-specifications \
    "ResourceType=vpc,Tags=[{Key=Name,Value=cloud-ebs-lab10-vpc},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --cidr-block 10.181.1.0/24 \
  --availability-zone "$AZ_A" \
  --tag-specifications \
    "ResourceType=subnet,Tags=[{Key=Name,Value=cloud-ebs-lab10-subnet},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 3 — Create Internet Gateway

```bash
export IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications \
    "ResourceType=internet-gateway,Tags=[{Key=Name,Value=cloud-ebs-lab10-igw},{Key=Project,Value=$PROJECT_TAG}]" \
  --query InternetGateway.InternetGatewayId \
  --output text \
  --region "$AWS_REGION")

aws ec2 attach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID" \
  --region "$AWS_REGION"
```

### Step 4 — Configure Routing

```bash
export RT_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
    "ResourceType=route-table,Tags=[{Key=Name,Value=cloud-ebs-lab10-rt},{Key=Project,Value=$PROJECT_TAG}]" \
  --query RouteTable.RouteTableId \
  --output text \
  --region "$AWS_REGION")

aws ec2 create-route \
  --route-table-id "$RT_ID" \
  --destination-cidr-block 0.0.0.0/0 \
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
  --description "Lab10 EBS Multi Attach Troubleshooting" \
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

### Step 9 — Launch Two EC2 Instances

```bash
read -r INSTANCE_A INSTANCE_B < <(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type "$INSTANCE_TYPE" \
  --iam-instance-profile "Name=$PROFILE_NAME" \
  --network-interfaces \
    "DeviceIndex=0,SubnetId=$SUBNET_ID,Groups=$SG_ID,AssociatePublicIpAddress=true" \
  --metadata-options "HttpTokens=required,HttpEndpoint=enabled" \
  --count 2 \
  --tag-specifications \
    "ResourceType=instance,Tags=[{Key=Name,Value=cloud-ebs-lab10-ec2},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Instances[].InstanceId' \
  --output text \
  --region "$AWS_REGION")

echo "EC2-A: $INSTANCE_A"
echo "EC2-B: $INSTANCE_B"
```

### Step 10 — Assign Instance Names

```bash
aws ec2 create-tags \
  --resources "$INSTANCE_A" \
  --tags Key=Name,Value=cloud-ebs-lab10-ec2-a \
  --region "$AWS_REGION"

aws ec2 create-tags \
  --resources "$INSTANCE_B" \
  --tags Key=Name,Value=cloud-ebs-lab10-ec2-b \
  --region "$AWS_REGION"
```

### Step 11 — Wait for Both Instances

```bash
aws ec2 wait instance-running \
  --instance-ids "$INSTANCE_A" "$INSTANCE_B" \
  --region "$AWS_REGION"

aws ec2 wait instance-status-ok \
  --instance-ids "$INSTANCE_A" "$INSTANCE_B" \
  --region "$AWS_REGION"
```

### Step 12 — Verify Availability Zones

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_A" "$INSTANCE_B" \
  --query 'Reservations[].Instances[].{ID:InstanceId,AZ:Placement.AvailabilityZone,State:State.Name}' \
  --output table \
  --region "$AWS_REGION"
```

Both instances must be in the same AZ.

---

## 8. Create the EBS gp3 Volume

### Step 1 — Create 5 GiB gp3

```bash
export VOLUME_ID=$(aws ec2 create-volume \
  --availability-zone "$AZ_A" \
  --volume-type gp3 \
  --size 5 \
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

### Step 3 — Verify Multi-Attach Configuration

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,Type:VolumeType,Size:Size,MultiAttach:MultiAttachEnabled,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
VolumeType: gp3
MultiAttach: False
State: available
```

### Step 4 — Attach to EC2-A

```bash
aws ec2 attach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_A" \
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

Expected: one attachment to EC2-A.

---

## 9. Create Application Data on EC2-A

### Step 1 — Connect Through Systems Manager

```bash
aws ssm start-session \
  --target "$INSTANCE_A" \
  --region "$AWS_REGION"
```

The following commands run inside EC2-A.

### Step 2 — Identify the EBS Data Device

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,SERIAL
```

Identify the new 5 GiB disk.

Verify the EBS volume identity:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

Use the actual device path if different.

After confirming the serial matches the Lab 10 EBS volume ID:

```bash
export DATA_DEVICE="/dev/nvme1n1"
```

### Step 3 — Inspect the Blank Disk

```bash
lsblk -f

sudo blkid "$DATA_DEVICE" || true

sudo file -s "$DATA_DEVICE"
```

Expected: no existing filesystem.

### Step 4 — Create ext4

**Destructive operation. Only run on the verified empty Lab 10 EBS data volume.**

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

### Step 6 — Create Test Files

```bash
echo "EBS Lab 10 - Application Data From EC2-A" \
  | sudo tee /data/important-data.txt

echo "Original owner: EC2-A" \
  | sudo tee /data/source-instance.txt
```

### Step 7 — Record Original Checksum

```bash
sha256sum /data/important-data.txt
```

Save the checksum for comparison after moving the volume to EC2-B.

### Step 8 — Verify Original Data

```bash
cat /data/important-data.txt

cat /data/source-instance.txt

findmnt /data
```

Expected:

```text
EBS Lab 10 - Application Data From EC2-A
Original owner: EC2-A
```

Exit SSM:

```bash
exit
```

Keep the volume mounted on EC2-A for the intentional attachment-failure test. Do not modify its files while testing the rejected second attachment.

---

## 10. Fault Injection — Simultaneous Attachment Failure

### Step 1 — Confirm Existing Attachment

From WSL2:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{Type:VolumeType,MultiAttach:MultiAttachEnabled,State:State,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Expected:

```text
Volume Type: gp3
MultiAttachEnabled: false
Volume State: in-use
Attached Instance: EC2-A
```

### Step 2 — Attempt Attachment to EC2-B

**Intentional fault injection:**

```bash
aws ec2 attach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_B" \
  --device /dev/sdf \
  --region "$AWS_REGION"
```

### Expected Error

```text
An error occurred (VolumeInUse) when calling
the AttachVolume operation:

vol-xxxxxxxx is already attached to an instance.
```

The exact message may vary.

### Step 3 — Verify Attachment List

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].Attachments[].{Instance:InstanceId,State:State,Device:Device}' \
  --output table \
  --region "$AWS_REGION"
```

Expected: the volume remains attached only to EC2-A.

### Step 4 — Verify Multi-Attach State

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,Type:VolumeType,MultiAttach:MultiAttachEnabled}' \
  --output table \
  --region "$AWS_REGION"
```

Expected: gp3 with Multi-Attach disabled.

### Fault Summary

| Check | Expected Finding |
|---|---|
| EC2-A | Running |
| EC2-B | Running |
| Same Availability Zone | Yes |
| EBS Type | gp3 |
| Multi-Attach | False |
| EC2-A Attachment | Successful |
| EC2-B Attachment | Rejected |
| Error | VolumeInUse |
| Original Data | Preserved |
| Root Cause | Unsupported simultaneous attachment |

---

## 11. Troubleshooting Investigation

### Check 1 — Inspect Volume Type

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].VolumeType' \
  --output text \
  --region "$AWS_REGION"
```

Expected:

```text
gp3
```

### Check 2 — Inspect Multi-Attach Configuration

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].MultiAttachEnabled' \
  --output text \
  --region "$AWS_REGION"
```

Expected:

```text
False
```

### Check 3 — Verify Availability Zones

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_A" "$INSTANCE_B" \
  --query 'Reservations[].Instances[].{Instance:InstanceId,AZ:Placement.AvailabilityZone}' \
  --output table \
  --region "$AWS_REGION"
```

Finding: both instances are in the same Availability Zone.

### Check 4 — Identify the Existing Attachment

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].Attachments' \
  --output json \
  --region "$AWS_REGION"
```

Finding: gp3 remains attached to EC2-A.

### Check 5 — Compare Supported EBS Types

Multi-Attach supports eligible io1 and io2 Provisioned IOPS SSD configurations, not gp3.

Therefore, the attempted architecture is incompatible with the selected volume type.

### Investigation Summary

| Investigation | Finding |
|---|---|
| EBS State | In-use |
| EBS Type | gp3 |
| MultiAttachEnabled | False |
| Existing Attachment | EC2-A |
| Second Attachment | Rejected |
| AZ Placement | Correct |
| Application Files | Unchanged |
| Root Cause | Unsupported EBS Multi-Attach configuration |

---

## 12. Root Cause Analysis

### Root Cause

The cloud engineer attempted to attach a standard gp3 EBS volume to two EC2 instances simultaneously.

The gp3 volume was already attached to EC2-A.

Because gp3 does not support Multi-Attach, AWS rejected the additional attachment request to EC2-B.

### Technical Explanation

Standard gp3 EBS volumes operate as single-attached block storage.

The volume can be detached and attached to another eligible EC2 instance, but it cannot remain attached to multiple instances simultaneously.

Supported io1 and io2 Multi-Attach configurations allow multiple attachments under specific conditions.

However, even supported Multi-Attach does not make ordinary ext4 safe for concurrent independent writers.

### Root Cause Flow

```text
gp3 EBS Volume
       |
       v
Attached to EC2-A
       |
       v
Attach Request to EC2-B
       |
       v
Multi-Attach Unsupported
       |
       X
VolumeInUse
       |
       v
Second Attachment Fails
```

### Five Whys

1. **Why did attachment to EC2-B fail?** The EBS volume was already attached to EC2-A.
2. **Why couldn't both instances attach simultaneously?** The volume did not support Multi-Attach.
3. **Why was Multi-Attach unsupported?** The volume type was gp3.
4. **Why not simply use io2 with ext4 on both servers?** Ordinary ext4 cannot safely coordinate concurrent filesystem writes across independent hosts.
5. **How should the issue be prevented?** Choose the correct storage architecture and understand AWS volume attachment restrictions before deployment.

### Prevention

- Validate EBS volume type before designing shared storage.
- Check Multi-Attach compatibility.
- Verify Availability Zone requirements.
- Use standard EBS for single-owner block storage.
- Use coordinated clustered storage when concurrent block access is required.
- Consider Amazon EFS for shared file access.
- Implement safe detach and reattach procedures.

---

## 13. Remediation — Safely Move EBS to EC2-B

### Step 1 — Reconnect to EC2-A

```bash
aws ssm start-session \
  --target "$INSTANCE_A" \
  --region "$AWS_REGION"
```

### Step 2 — Verify Source Data

Inside EC2-A:

```bash
findmnt /data

cat /data/important-data.txt

sha256sum /data/important-data.txt
```

Compare the checksum with the original recorded value.

### Step 3 — Check Active Filesystem Users

```bash
sudo fuser -vm /data
```

Confirm no application is actively using the filesystem.

Stop application I/O gracefully if necessary.

### Step 4 — Unmount Safely

```bash
cd /

sync

sudo umount /data
```

Verify:

```bash
findmnt /data
```

Expected: no active mount.

Exit:

```bash
exit
```

### Step 5 — Detach EBS From EC2-A

From WSL2:

```bash
aws ec2 detach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_A" \
  --region "$AWS_REGION"
```

Wait:

```bash
aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 6 — Verify Detached State

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

### Step 7 — Attach EBS to EC2-B

```bash
aws ec2 attach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_B" \
  --device /dev/sdf \
  --region "$AWS_REGION"

aws ec2 wait volume-in-use \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 8 — Verify New Attachment

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].Attachments[].{Instance:InstanceId,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

Expected: EC2-B is the only attached instance.

### Step 9 — Connect to EC2-B

```bash
aws ssm start-session \
  --target "$INSTANCE_B" \
  --region "$AWS_REGION"
```

### Step 10 — Identify Transferred NVMe Disk

Inside EC2-B:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,SERIAL
```

Verify the EBS volume ID:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

After identifying the actual data disk:

```bash
export DATA_DEVICE="/dev/nvme1n1"
```

Replace the example path if required.

### Step 11 — Verify Existing Filesystem

```bash
sudo blkid "$DATA_DEVICE"

lsblk -f
```

Expected: ext4 filesystem already exists.

**Do not run `mkfs.ext4` on the transferred volume.**

### Step 12 — Mount the Volume on EC2-B

```bash
sudo mkdir -p /data

sudo mount "$DATA_DEVICE" /data
```

Verify:

```bash
findmnt /data

df -hT /data
```

### Step 13 — Verify Original Application Files

```bash
cat /data/important-data.txt

cat /data/source-instance.txt

sha256sum /data/important-data.txt
```

Expected:

```text
EBS Lab 10 - Application Data From EC2-A
Original owner: EC2-A
```

Compare the checksum with the original EC2-A result.

### Step 14 — Test Read-Write Access

```bash
echo "EBS successfully moved to EC2-B" \
  | sudo tee /data/recovery-test.txt
```

Verify:

```bash
cat /data/recovery-test.txt
```

Expected: successful read and write.

### Remediation Summary

| Check | During Incident | After Remediation |
|---|---|---|
| EBS Type | gp3 | gp3 |
| Multi-Attach | Unsupported | Not required |
| EC2-A Attachment | Attached | Detached |
| EC2-B Attachment | Failed | Attached |
| Original Files | Accessible on A | Accessible on B |
| ext4 Filesystem | Preserved | Preserved |
| SHA-256 | Original | Matches |
| Data Corruption | Not observed | Not observed |
| Recovery | Blocked | Successful |

---

## 14. Advanced Storage Architecture Alternatives

### Option 1 — Standard EBS Single-Instance Attachment

Choose gp3 EBS when an application needs block storage attached to one EC2 instance at a time.

If the data must move to another instance:

1. Stop application writes.
2. Unmount the filesystem.
3. Detach the EBS volume.
4. Wait until the volume is available.
5. Attach it to the new EC2 instance in the same AZ.
6. Mount the existing filesystem.
7. Verify application data.

This is the remediation used in our lab.

### Option 2 — io2 Multi-Attach

For specialized workloads requiring concurrent access to block storage, AWS offers compatible io2 Multi-Attach configurations.

Important requirements include:

- Supported EBS volume type.
- Compatible Nitro-based instances.
- Same Availability Zone.
- Suitable application coordination.
- Appropriate clustered filesystem for concurrent filesystem writers.

Multi-Attach is not a replacement for distributed locking or filesystem coordination.

### Option 3 — Amazon EFS

Amazon EFS provides network file storage that can be accessed concurrently by multiple EC2 instances.

Example:

```text
              Application EC2-A
                       |
                       |
                    Amazon EFS
                       |
                       |
              Application EC2-B
```

EFS is often suitable when the actual requirement is shared files and directories rather than shared block devices.

### Choosing the Correct Service

| Requirement | Recommended Approach |
|---|---|
| Application needs a single block disk | gp3 EBS |
| Move a disk between instances | Safe EBS detach and reattach |
| Multiple servers need shared POSIX files | Amazon EFS |
| Specialized clustered block storage | io2 Multi-Attach with appropriate coordination |
| Shared object storage | Amazon S3 |

### Important Design Lesson

Do not choose EBS Multi-Attach simply because multiple servers need access to the same files.

First determine whether the application needs block storage, file storage, or object storage.

---

## 15. Final Verification

### AWS Verification

From WSL2:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{State:State,Type:VolumeType,MultiAttach:MultiAttachEnabled,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Expected:

- Volume state: `in-use`
- Volume type: `gp3`
- Multi-Attach: `false`
- Exactly one attachment
- Attached instance: EC2-B

### Linux Verification

Inside EC2-B:

```bash
lsblk -f

findmnt /data

df -hT /data

cat /data/important-data.txt

cat /data/source-instance.txt

sha256sum /data/important-data.txt

cat /data/recovery-test.txt
```

### Verification Matrix

| Check | Expected Result |
|---|---|
| EC2-A Running | Yes |
| EC2-B Running | Yes |
| Same AZ | Yes |
| EBS Volume Type | gp3 |
| Multi-Attach | False |
| Second Attachment Fault | Reproduced |
| Original EBS Detached | Yes |
| EBS Attached to EC2-B | Yes |
| Filesystem Mounted | Yes |
| Original Files | Accessible |
| SHA-256 Checksum | Matches |
| New File Writes | Successful |

### Success Criteria

- Two EC2 instances created in the same AZ.
- Dedicated gp3 EBS volume created.
- Multi-Attach configuration verified.
- EBS successfully attached to EC2-A.
- ext4 filesystem created.
- Application test files written.
- Original checksum recorded.
- Simultaneous attachment attempted.
- VolumeInUse error reproduced.
- EBS volume type investigated.
- Existing attachment verified.
- Unsupported Multi-Attach identified as the root cause.
- Filesystem safely unmounted on EC2-A.
- EBS detached without forcing.
- EBS attached to EC2-B.
- Original ext4 filesystem mounted.
- Application data recovered.
- SHA-256 checksum verified.
- New file writes successful.
- Temporary AWS resources cleaned up.

---

## 16. Production Best Practices

### EBS Attachment Troubleshooting Checklist

1. Identify the affected EBS volume.
2. Verify the AWS account and Region.
3. Check EBS volume state.
4. Inspect existing attachments.
5. Verify the EBS volume type.
6. Inspect `MultiAttachEnabled`.
7. Check EC2 instance compatibility.
8. Verify Availability Zone placement.
9. Determine whether simultaneous access is actually required.
10. Choose sequential attachment or a supported shared-storage design.
11. Stop application I/O before detachment.
12. Unmount filesystems safely.
13. Avoid force detachment.
14. Mount only the verified correct block device.
15. Verify application data after recovery.

### Prevention

- Document supported EBS volume configurations.
- Validate shared-storage requirements before deployment.
- Use gp3 for appropriate single-owner block storage.
- Use io2 Multi-Attach only for suitable clustered workloads.
- Avoid concurrent ext4 or XFS read-write mounts across hosts.
- Use EFS when shared file storage is appropriate.
- Maintain backups before risky storage changes.
- Monitor EBS attachment failures.
- Verify data integrity after storage migration.

### Security Considerations

- Use least-privilege IAM permissions.
- Restrict EBS attachment and detachment operations.
- Use Systems Manager where appropriate.
- Keep security groups restrictive.
- Verify volume ownership before deletion.
- Avoid storing production credentials in test files.
- Preserve application data during migrations.

---

## 17. Cleanup

**Warning:** These operations permanently delete the Lab 10 gp3 data volume, both EC2 instances, and supporting infrastructure.

### Step 1 — Unmount on EC2-B

Inside EC2-B:

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

Exit SSM:

```bash
exit
```

### Step 2 — Verify Resource Ownership

From WSL2:

```bash
aws sts get-caller-identity

aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,State:State,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"

aws ec2 describe-instances \
  --instance-ids "$INSTANCE_A" "$INSTANCE_B" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"
```

Confirm all resources belong to Lab 10.

### Step 3 — Detach From EC2-B

```bash
aws ec2 detach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_B" \
  --region "$AWS_REGION"

aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 4 — Delete the EBS Data Volume

Only after confirming the test data is no longer needed:

```bash
aws ec2 delete-volume \
  --volume-id "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 5 — Verify Root Volume Settings

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_A" "$INSTANCE_B" \
  --query 'Reservations[].Instances[].{ID:InstanceId,BlockDevices:BlockDeviceMappings}' \
  --output json \
  --region "$AWS_REGION"
```

Confirm both root EBS volumes have `DeleteOnTermination=true`.

### Step 6 — Terminate Both Instances

```bash
aws ec2 terminate-instances \
  --instance-ids "$INSTANCE_A" "$INSTANCE_B" \
  --region "$AWS_REGION"

aws ec2 wait instance-terminated \
  --instance-ids "$INSTANCE_A" "$INSTANCE_B" \
  --region "$AWS_REGION"
```

### Step 7 — Delete IAM Resources

Verify the IAM role and instance profile are dedicated to Lab 10:

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

### Step 8 — Verify Network Interface Cleanup

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'NetworkInterfaces[].{ID:NetworkInterfaceId,Status:Status}' \
  --output table \
  --region "$AWS_REGION"
```

Wait until no lab network interfaces remain.

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

Expected: empty arrays after cleanup completes.

Also verify that both EC2 instances, their root volumes, and dedicated IAM resources have been removed.

### Cleanup Summary

| Resource | Expected Result |
|---|---|
| EBS gp3 Data Volume | Deleted |
| EC2-A | Terminated |
| EC2-B | Terminated |
| EC2 Root Volumes | Deleted |
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

### Q1. What is EBS Multi-Attach?

EBS Multi-Attach allows eligible Provisioned IOPS SSD volumes to be attached to multiple compatible EC2 instances in the same Availability Zone.

### Q2. Does gp3 support Multi-Attach?

No. gp3 volumes support normal single-instance attachment, not Multi-Attach.

### Q3. Which EBS volume types support Multi-Attach?

Eligible io1 and io2 Provisioned IOPS SSD configurations.

### Q4. Why did the second attachment fail?

The gp3 volume was already attached to EC2-A and did not support simultaneous attachment to EC2-B.

### Q5. How do you check the Multi-Attach setting?

Use `aws ec2 describe-volumes` and inspect `MultiAttachEnabled`.

### Q6. Can a Multi-Attach volume span Availability Zones?

No. Participating instances must be in the same Availability Zone.

### Q7. Can ext4 safely support concurrent writes from multiple EC2 instances?

No. Standard ext4 does not coordinate filesystem writes across independent hosts.

### Q8. What is a cluster-aware filesystem?

A filesystem designed to coordinate access to shared block storage across multiple hosts.

### Q9. When is Amazon EFS a better choice?

When multiple EC2 instances need shared access to files and directories without managing clustered block-storage access.

### Q10. How do you safely transfer a gp3 volume between instances?

Stop application writes, unmount, detach from the original instance, attach to the target instance, mount the existing filesystem, and verify data integrity.

### Q11. Does Multi-Attach automatically provide high availability?

No. Application clustering, coordination, failover, and recovery still need to be designed.

### Q12. How would you explain this lab in an interview?

I deployed two Amazon Linux EC2 instances in the same Availability Zone.

I created a dedicated gp3 EBS volume and attached it to EC2-A.

After creating an ext4 filesystem and application test files, I attempted to attach the same volume to EC2-B.

AWS rejected the second attachment because gp3 does not support Multi-Attach.

I investigated the EBS volume type, `MultiAttachEnabled` configuration, instance placement, and existing attachment information using AWS CLI.

After identifying the root cause, I safely unmounted the filesystem on EC2-A and detached the volume without forcing the operation.

I then attached the volume to EC2-B, mounted the existing filesystem, and verified that the original files and SHA-256 checksum remained intact.

I also studied when io2 Multi-Attach or Amazon EFS should be used for shared-storage architectures.

---

## 19. Repository Structure

```text
10-ebs-multi-attach-misconfiguration/
├── README.md
└── ec2-trust-policy.json
```

### File Descriptions

| File | Purpose |
|---|---|
| `README.md` | Complete EBS Multi-Attach troubleshooting documentation |
| `ec2-trust-policy.json` | EC2 trust policy used for Systems Manager |

Do not commit AWS credentials, private keys, or sensitive application data.

---

## 20. Lessons Learned

This lab demonstrates why understanding EBS volume capabilities is important before designing shared storage.

The attempted second attachment failed because gp3 does not support EBS Multi-Attach.

The issue was not caused by the EBS filesystem, Availability Zone placement, or EC2 operating system.

We learned how to identify existing EBS attachments, inspect Multi-Attach configuration, and understand supported volume types.

We also practiced safely moving an EBS volume between two EC2 instances without formatting it or damaging existing files.

Finally, we compared standard EBS, io2 Multi-Attach, and Amazon EFS to understand which storage solution fits different application requirements.

**Key Lesson:** EBS gp3 volumes cannot be attached to multiple EC2 instances simultaneously. Use safe sequential attachment for single-owner storage, or choose an appropriate shared-storage architecture when concurrent access is required.

---

## 21. Skills Demonstrated

- Amazon EBS provisioning
- EBS gp3 configuration
- EBS attachment troubleshooting
- Multi-Attach compatibility investigation
- AWS CLI v2
- Amazon EC2 administration
- Availability Zone verification
- Linux NVMe device identification
- ext4 filesystem management
- Safe EBS detachment
- EBS volume migration
- SHA-256 data integrity verification
- AWS Systems Manager
- Block storage vs. file storage comparison
- io2 Multi-Attach fundamentals
- Amazon EFS architecture fundamentals
- Root cause analysis
- Safe AWS resource cleanup

## Final Outcome

| Item | Expected Result |
|---|---|
| Incident | INC-EBS-010 |
| Problem | Second EBS attachment rejected |
| EC2 Instances | Two |
| EBS Volume Type | gp3 |
| EBS Volume Size | 5 GiB |
| Multi-Attach | Unsupported |
| Fault Injection | Attach EBS to EC2-B while attached to EC2-A |
| Expected Error | VolumeInUse |
| Root Cause | Unsupported simultaneous attachment |
| Remediation | Safe EBS volume migration |
| Original Instance | EC2-A |
| Destination Instance | EC2-B |
| Final Attachment | EC2-B only |
| Original Files | Preserved |
| Checksum Verification | Matches baseline |
| Cleanup | Temporary resources removed |
| Expected Incident Status | Resolved |

**Key Takeaway:** Correct cloud storage architecture starts with understanding application access requirements. Standard EBS gp3 is suitable for single-owner block storage, while concurrent access requires a deliberately designed shared-storage solution.
