# Lab 01 — EBS Volume Not Attached

## Overview

This lab demonstrates how to troubleshoot an Amazon Elastic Block Store (EBS) volume that cannot be attached to an EC2 instance because the volume and instance are located in different Availability Zones.

We create an Amazon Linux 2023 EC2 instance in one Availability Zone and a separate 5 GiB `gp3` EBS volume in another Availability Zone.

When we attempt to attach the volume, AWS rejects the request because ordinary EBS volumes must be attached to compatible EC2 instances in the same Availability Zone.

We investigate the EC2 instance state, EBS volume state, attachment information, and Availability Zone configuration using AWS CLI.

After identifying the mismatch, we create a replacement EBS volume in the correct Availability Zone, attach it successfully, and verify that the new NVMe block device appears inside Linux.

Finally, we safely detach and delete the disposable test volumes and clean up the temporary infrastructure.

**Troubleshooting workflow:**

Provision → Fault Injection → Investigation → Root Cause Analysis → Remediation → Verification → Cleanup

---

## 1. Lab Information

| Item | Details |
|---|---|
| Incident ID | INC-EBS-001 |
| Difficulty | Beginner |
| AWS Region | `us-east-1` |
| AWS Services | EC2, EBS, VPC, IAM, Systems Manager |
| Operating System | Amazon Linux 2023 |
| Instance Type | `t3.micro` |
| EC2 Availability Zone | `us-east-1a` |
| Incorrect Volume AZ | `us-east-1b` |
| Correct Volume AZ | `us-east-1a` |
| Volume Type | `gp3` |
| Volume Size | 5 GiB |
| Requested Attachment Device | `/dev/sdf` |
| Linux Device | NVMe device such as `/dev/nvme1n1` |
| Fault Injection | Cross-AZ attachment attempt |
| Expected Error | Availability Zone mismatch |
| Root Cause | EC2 and EBS exist in different AZs |
| Remediation | Create compatible volume in EC2 AZ |
| Final Verification | Volume attached and visible in Linux |

The Availability Zones shown are example selections. Actual zones depend on instance type availability and the AWS account.

---

## 2. Production Incident Scenario

### Incident Description

A company launches an EC2 instance that requires additional EBS storage for application data.

The engineer creates a new EBS volume and attempts to attach it to the running instance.

However, the AWS API rejects the attachment request.

The EC2 instance is running, the EBS volume exists, and the volume state is `available`.

After investigation, the engineer discovers that the EC2 instance and EBS volume are located in different Availability Zones.

The instance is in `us-east-1a`, while the volume is in `us-east-1b`.

Although both resources belong to the same AWS Region, an ordinary EBS volume cannot be attached directly across Availability Zones.

### Incident Symptoms

- EC2 instance is running.
- EBS volume is available.
- Attachment request fails.
- Volume remains unattached.
- No new data disk appears in Linux.
- EC2 and EBS Availability Zones do not match.

### Possible Business Impact

- Application deployment delays.
- Unavailable application storage.
- Failed storage provisioning.
- Delayed application startup.
- Increased troubleshooting time.
- Potential downtime for applications depending on the missing storage.

### Objective

Determine why the EBS volume cannot be attached, correct the Availability Zone mismatch, and verify successful attachment without affecting the EC2 root volume.

---

## 3. AWS Architecture

### Broken Architecture

```text
AWS Region: us-east-1

+-----------------------------------+
| Availability Zone: us-east-1a     |
|                                   |
|   +---------------------------+   |
|   | EC2 Instance              |   |
|   | Amazon Linux 2023         |   |
|   | State: Running            |   |
|   +---------------------------+   |
|                                   |
+-----------------------------------+

                  X
          AttachVolume Fails
          Different AZs
                  X

+-----------------------------------+
| Availability Zone: us-east-1b     |
|                                   |
|   +---------------------------+   |
|   | EBS Volume                |   |
|   | gp3, 5 GiB                |   |
|   | State: Available          |   |
|   +---------------------------+   |
|                                   |
+-----------------------------------+
```

### Corrected Architecture

```text
AWS Region: us-east-1

+-------------------------------------------+
| Availability Zone: us-east-1a             |
|                                           |
|  +-------------------------------+        |
|  | EC2 Instance                  |        |
|  | Amazon Linux 2023             |        |
|  | State: Running                |        |
|  +---------------+---------------+        |
|                  |                        |
|                  | EBS Attachment         |
|                  | /dev/sdf               |
|                  v                        |
|  +-------------------------------+        |
|  | EBS Volume                    |        |
|  | gp3, 5 GiB                    |        |
|  | State: in-use                 |        |
|  | Linux: NVMe block device      |        |
|  +-------------------------------+        |
|                                           |
+-------------------------------------------+
```

### Resource Configuration

| Resource | Configuration |
|---|---|
| VPC | `10.172.0.0/16` |
| Public Subnet | `10.172.1.0/24` |
| Internet Gateway | Attached |
| EC2 | Amazon Linux 2023 |
| Security Group | No inbound access |
| Management | AWS Systems Manager |
| Incorrect EBS | 5 GiB gp3 in wrong AZ |
| Correct EBS | 5 GiB gp3 in instance AZ |

---

## 4. Understanding Amazon EBS Attachments

Amazon EBS provides persistent block storage for EC2 instances.

An EBS volume is created in a specific Availability Zone.

For standard attachments, the instance and volume must be in the same Availability Zone.

### Important EBS Volume States

| State | Meaning |
|---|---|
| `creating` | Volume is being created |
| `available` | Volume is ready and unattached |
| `in-use` | Volume is attached or otherwise in use |
| `deleting` | Volume deletion has started |
| `deleted` | Volume has been deleted |
| `error` | Volume encountered an error |

### Common Attachment Failure Causes

| Problem | Explanation |
|---|---|
| Availability Zone mismatch | Volume and instance are in different AZs |
| Volume already attached | Volume cannot be attached in an unsupported configuration |
| Invalid EC2 instance | Instance ID or state is unsuitable |
| Invalid volume state | Volume is not ready for the requested operation |
| Device name conflict | Requested attachment name is already in use |
| Attachment limit reached | Instance supports no additional attachments |
| Permission failure | IAM or applicable KMS permissions are insufficient |

### EBS Attachment vs. Filesystem Mounting

These are separate operations.

**Attachment** makes the EBS block device available to the operating system.

**Mounting** makes a filesystem on that device accessible through a directory.

An attached EBS volume does not automatically become a usable filesystem.

This lab focuses only on attachment.

Filesystem formatting, mounting, and persistent `/etc/fstab` configuration are covered in Lab 02.

### Important NVMe Concept

On Nitro-based EC2 instances, EBS volumes appear as NVMe block devices.

For example:

```text
AWS attachment name: /dev/sdf
Linux device name:   /dev/nvme1n1
```

The Linux device name can differ between instances and attachment operations.

Always identify the correct volume using its EBS volume ID or NVMe serial information.

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

### Safety Requirements

- Use dedicated disposable EBS volumes.
- Never format the EC2 root volume.
- Verify volume IDs before attaching or deleting.
- Do not delete EBS volumes containing important data.
- Do not use forced detachment as a normal troubleshooting method.
- Use project tags.
- Verify the AWS account before provisioning and cleanup.

### Cost Considerations

EC2 instances, public IPv4 addresses, EBS volumes, and related resources may incur charges.

Delete temporary infrastructure after completing the lab.

---

## 6. Prepare the Environment

### Step 1 — Configure Variables

Open Ubuntu WSL2.

```bash
export AWS_REGION="us-east-1"
export PROJECT_TAG="EBS-Troubleshooting-Lab01"

export INSTANCE_NAME="cloud-ebs-lab01-ec2"
export SG_NAME="cloud-ebs-lab01-sg"

export ROLE_NAME="cloud-ebs-lab01-ssm-role"
export PROFILE_NAME="cloud-ebs-lab01-ssm-profile"

export VOLUME_SIZE="5"

export AZ_A="us-east-1a"
export AZ_B="us-east-1b"
```

### Step 2 — Verify AWS Credentials

```bash
aws sts get-caller-identity
aws --version
```

### Step 3 — Verify Availability Zones

```bash
aws ec2 describe-availability-zones \
  --region "$AWS_REGION" \
  --query 'AvailabilityZones[].{Zone:ZoneName,State:State}' \
  --output table
```

Verify the instance type is supported:

```bash
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters "Name=instance-type,Values=t3.micro" \
  --region "$AWS_REGION" \
  --query 'InstanceTypeOfferings[].Location' \
  --output table
```

Choose two supported and different Availability Zones.

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
  --query 'Volumes[].{ID:VolumeId,State:State,AZ:AvailabilityZone}' \
  --output table \
  --region "$AWS_REGION"
```

Avoid creating duplicate resources.

---

## 7. Create EC2 Infrastructure

### Step 1 — Create VPC

```bash
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block "10.172.0.0/16" \
  --tag-specifications \
  "ResourceType=vpc,Tags=[{Key=Name,Value=cloud-ebs-lab01-vpc},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Vpc.VpcId' \
  --output text \
  --region "$AWS_REGION")
```

Enable VPC DNS:

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
  --cidr-block "10.172.1.0/24" \
  --availability-zone "$AZ_A" \
  --tag-specifications \
  "ResourceType=subnet,Tags=[{Key=Name,Value=cloud-ebs-lab01-subnet},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 3 — Create Internet Gateway

```bash
export IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications \
  "ResourceType=internet-gateway,Tags=[{Key=Name,Value=cloud-ebs-lab01-igw},{Key=Project,Value=$PROJECT_TAG}]" \
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
  "ResourceType=route-table,Tags=[{Key=Name,Value=cloud-ebs-lab01-rt},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --description "EBS Lab01 instance no inbound access" \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
  "ResourceType=security-group,Tags=[{Key=Name,Value=$SG_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query GroupId \
  --output text \
  --region "$AWS_REGION")
```

No inbound permissions are needed because Systems Manager will provide access.

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

### Step 7 — Create IAM Role

Check whether the role and instance profile already exist.

For new resources:

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

### Step 8 — Retrieve Amazon Linux 2023 AMI

```bash
export AMI_ID=$(aws ssm get-parameter \
  --name "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64" \
  --query Parameter.Value \
  --output text \
  --region "$AWS_REGION")
```

### Step 9 — Launch EC2

Allow IAM propagation before launching the instance.

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

### Step 11 — Verify EC2 Availability Zone

```bash
export INSTANCE_AZ=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' \
  --output text \
  --region "$AWS_REGION")

echo "EC2 Instance: $INSTANCE_ID"
echo "EC2 AZ: $INSTANCE_AZ"
```

Expected: `us-east-1a`.

---

## 8. Fault Injection — Create EBS in the Wrong AZ

### Step 1 — Create Incorrect Volume

```bash
export WRONG_VOLUME_ID=$(aws ec2 create-volume \
  --availability-zone "$AZ_B" \
  --size "$VOLUME_SIZE" \
  --volume-type gp3 \
  --tag-specifications \
    "ResourceType=volume,Tags=[{Key=Name,Value=cloud-ebs-lab01-wrong-az},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'VolumeId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 2 — Wait for Availability

```bash
aws ec2 wait volume-available \
  --volume-ids "$WRONG_VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 3 — Inspect Volume

```bash
aws ec2 describe-volumes \
  --volume-ids "$WRONG_VOLUME_ID" \
  --query 'Volumes[].{ID:VolumeId,State:State,AZ:AvailabilityZone,Size:Size,Type:VolumeType}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
State: available
AZ: us-east-1b
Size: 5
Type: gp3
```

### Step 4 — Attempt Attachment

```bash
aws ec2 attach-volume \
  --volume-id "$WRONG_VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdf \
  --region "$AWS_REGION"
```

Expected: AWS rejects the request because the instance and volume are in different Availability Zones.

An error may resemble:

```text
InvalidVolume.ZoneMismatch

The volume and instance are in different availability zones.
```

The exact error code or text may vary.

### Step 5 — Verify Unattached State

```bash
aws ec2 describe-volumes \
  --volume-ids "$WRONG_VOLUME_ID" \
  --query 'Volumes[0].{State:State,AZ:AvailabilityZone,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Expected:

```text
State: available
Attachments: []
```

---

## 9. Troubleshooting Investigation

### Check 1 — Verify EC2 State

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,AZ:Placement.AvailabilityZone}' \
  --output table \
  --region "$AWS_REGION"
```

Expected: instance running in `AZ_A`.

### Check 2 — Verify Volume State

```bash
aws ec2 describe-volumes \
  --volume-ids "$WRONG_VOLUME_ID" \
  --query 'Volumes[].{ID:VolumeId,State:State,AZ:AvailabilityZone}' \
  --output table \
  --region "$AWS_REGION"
```

Expected: volume available in `AZ_B`.

### Check 3 — Inspect Attachments

```bash
aws ec2 describe-volumes \
  --volume-ids "$WRONG_VOLUME_ID" \
  --query 'Volumes[0].Attachments' \
  --output json \
  --region "$AWS_REGION"
```

Expected:

```json
[]
```

### Check 4 — Inspect EC2 Block Device Mappings

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings' \
  --output json \
  --region "$AWS_REGION"
```

Verify that `/dev/sdf` is not already assigned.

### Check 5 — Compare Availability Zones

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[].Instances[].{Instance:InstanceId,AZ:Placement.AvailabilityZone}' \
  --output table \
  --region "$AWS_REGION"

aws ec2 describe-volumes \
  --volume-ids "$WRONG_VOLUME_ID" \
  --query 'Volumes[].{Volume:VolumeId,AZ:AvailabilityZone}' \
  --output table \
  --region "$AWS_REGION"
```

### Investigation Summary

| Check | Finding |
|---|---|
| Region | Same |
| EC2 State | Running |
| Volume State | Available |
| Volume Type | gp3 |
| Volume Size | 5 GiB |
| Existing Attachment | None |
| EC2 AZ | `us-east-1a` |
| EBS AZ | `us-east-1b` |
| Root Cause | Availability Zone mismatch |

---

## 10. Root Cause Analysis

### Root Cause

The EBS volume was created in a different Availability Zone from the EC2 instance.

The EC2 instance existed in `us-east-1a`, while the EBS volume existed in `us-east-1b`.

AWS rejected the attachment because standard EBS volume attachments require both resources to be in the same Availability Zone.

### Root Cause Flow

```text
EC2 Instance in AZ A
         |
         v
EBS Volume in AZ B
         |
         v
AttachVolume Request
         |
         X
Availability Zone Mismatch
         |
         v
Attachment Rejected
```

### Five Whys

1. **Why did the attachment fail?** AWS rejected the `AttachVolume` request.
2. **Why was it rejected?** The volume and instance were in different Availability Zones.
3. **Why did the mismatch occur?** The volume was created using a different AZ from the instance.
4. **Why was the mismatch not prevented?** The volume provisioning process did not verify the instance's Availability Zone.
5. **How can this be prevented?** Automatically derive the volume's AZ from the target instance during provisioning.

### Production Remediation Strategy

For an empty, disposable volume, create a replacement volume in the correct Availability Zone.

For an existing volume containing important data, create a snapshot, wait for completion, then create a new volume from that snapshot in the destination Availability Zone.

Do not delete the original data volume before validating recovery.

---

## 11. Remediation — Create a Correct-AZ Volume

### Step 1 — Create Replacement Volume

```bash
export CORRECT_VOLUME_ID=$(aws ec2 create-volume \
  --availability-zone "$INSTANCE_AZ" \
  --size "$VOLUME_SIZE" \
  --volume-type gp3 \
  --tag-specifications \
    "ResourceType=volume,Tags=[{Key=Name,Value=cloud-ebs-lab01-correct-az},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'VolumeId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 2 — Wait for Availability

```bash
aws ec2 wait volume-available \
  --volume-ids "$CORRECT_VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 3 — Verify AZ

```bash
export CORRECT_VOLUME_AZ=$(aws ec2 describe-volumes \
  --volume-ids "$CORRECT_VOLUME_ID" \
  --query 'Volumes[0].AvailabilityZone' \
  --output text \
  --region "$AWS_REGION")

echo "Instance AZ: $INSTANCE_AZ"
echo "Volume AZ: $CORRECT_VOLUME_AZ"
```

Confirm they match.

### Step 4 — Attach Volume

```bash
aws ec2 attach-volume \
  --volume-id "$CORRECT_VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdf \
  --region "$AWS_REGION"
```

### Step 5 — Wait for Attachment

```bash
aws ec2 wait volume-in-use \
  --volume-ids "$CORRECT_VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 6 — Inspect Attachment

```bash
aws ec2 describe-volumes \
  --volume-ids "$CORRECT_VOLUME_ID" \
  --query 'Volumes[0].{State:State,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Expected:

```text
Volume State: in-use
Attachment State: attached
Device: /dev/sdf
```

---

## 12. Verify the Volume Inside Linux

### Step 1 — Check SSM Availability

```bash
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
  --query 'InstanceInformationList[].{ID:InstanceId,Status:PingStatus}' \
  --output table \
  --region "$AWS_REGION"
```

### Step 2 — Connect to EC2

```bash
aws ssm start-session \
  --target "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

### Step 3 — List Block Devices

Inside EC2:

```bash
lsblk
```

Possible output:

```text
NAME          SIZE TYPE MOUNTPOINTS
nvme0n1         8G disk
├─nvme0n1p1     8G part /
└─nvme0n1p128   1M part
nvme1n1         5G disk
```

The device names and root partition layout may differ.

### Step 4 — Identify the EBS Volume

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS,SERIAL
```

Compare the disk serial with the EBS volume ID.

On supported Amazon Linux images, you can also use:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

Replace `/dev/nvme1n1` with the actual device.

The goal is to confirm the correct EBS volume rather than relying only on the device name.

### Step 5 — Inspect the Device Safely

```bash
lsblk -f
```

After identifying the correct new disk:

```bash
sudo file -s /dev/nvme1n1
```

A newly created empty volume may appear as:

```text
/dev/nvme1n1: data
```

Do not format or mount the volume in this lab.

### Step 6 — Exit EC2

```bash
exit
```

---

## 13. Final Verification

### Verify Correct Attachment

```bash
aws ec2 describe-volumes \
  --volume-ids "$CORRECT_VOLUME_ID" \
  --query 'Volumes[0].{Volume:VolumeId,AZ:AvailabilityZone,State:State,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

### Compare Both Volumes

```bash
aws ec2 describe-volumes \
  --volume-ids "$WRONG_VOLUME_ID" "$CORRECT_VOLUME_ID" \
  --query 'Volumes[].{Volume:VolumeId,AZ:AvailabilityZone,State:State,Instance:Attachments[0].InstanceId}' \
  --output table \
  --region "$AWS_REGION"
```

### Verification Matrix

| Check | Fault | After Remediation |
|---|---|---|
| EC2 State | Running | Running |
| Volume AZ | Wrong | Correct |
| Volume State | Available | In-use |
| Attachment | Rejected | Successful |
| Instance Association | None | Correct Instance |
| Linux Device | Not Present | Present |
| Root Volume | Unchanged | Unchanged |

### Success Criteria

- EC2 launched in AZ A.
- EBS volume created in AZ B.
- Cross-AZ attachment failed.
- Mismatch identified using AWS CLI.
- Replacement volume created in AZ A.
- Correct volume attached successfully.
- Linux detected the new block device.
- Root filesystem remained unchanged.
- Temporary resources cleaned up.

---

## 14. Production Best Practices

### EBS Attachment Troubleshooting Checklist

1. Verify AWS account and Region.
2. Identify the intended EC2 instance.
3. Confirm instance state.
4. Identify the EBS volume.
5. Check volume state.
6. Compare Availability Zones.
7. Inspect current attachments.
8. Verify the device name is not already assigned.
9. Check EC2 attachment limits.
10. Review IAM and KMS permissions if applicable.
11. Attach the compatible volume.
12. Verify EBS state and Linux device recognition.

### Common Attachment Errors

| Problem | Possible Symptom | Remediation |
|---|---|---|
| Different Availability Zones | Attachment rejected | Create volume in correct AZ |
| Volume already in use | Attachment fails | Review existing attachment |
| Invalid device name | Attachment error | Use available device name |
| Volume still creating | Not ready | Wait for `available` |
| Instance attachment limit | Attachment fails | Review instance limitations |
| IAM permission denied | AccessDenied | Correct IAM permissions |
| KMS access issue | Encrypted volume failure | Investigate KMS access |

### Prevention

- Use Terraform or CloudFormation for provisioning.
- Retrieve the EC2 instance's AZ dynamically.
- Avoid hardcoding inconsistent Availability Zones.
- Validate resource IDs and AZs before attachment.
- Tag storage resources.
- Create snapshots before risky storage changes.
- Verify the correct Linux device using its volume ID.
- Maintain backup and recovery procedures.
- Avoid formatting unfamiliar block devices.

---

## 15. Cleanup

**Warning:** These commands permanently delete the disposable EBS volumes and terminate the lab EC2 instance. Verify resource ownership before continuing.

### Step 1 — Verify Ownership

```bash
aws sts get-caller-identity

aws ec2 describe-volumes \
  --volume-ids "$WRONG_VOLUME_ID" "$CORRECT_VOLUME_ID" \
  --query 'Volumes[].{ID:VolumeId,State:State,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"

aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"
```

Confirm that all resources belong to `EBS-Troubleshooting-Lab01`.

### Step 2 — Detach Correct Volume

```bash
aws ec2 detach-volume \
  --volume-id "$CORRECT_VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait volume-available \
  --volume-ids "$CORRECT_VOLUME_ID" \
  --region "$AWS_REGION"
```

If the volume contains a mounted filesystem, safely unmount it before detaching.

### Step 3 — Delete Disposable Volumes

```bash
aws ec2 delete-volume \
  --volume-id "$WRONG_VOLUME_ID" \
  --region "$AWS_REGION"

aws ec2 delete-volume \
  --volume-id "$CORRECT_VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 4 — Verify Root Volume Settings

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings' \
  --output json \
  --region "$AWS_REGION"
```

Confirm that the root EBS volume is configured with `DeleteOnTermination=true`.

### Step 5 — Terminate EC2

```bash
aws ec2 terminate-instances \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait instance-terminated \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

### Step 6 — Delete IAM Resources

Verify the role and instance profile are dedicated to this lab and not shared.

```bash
aws iam list-role-tags \
  --role-name "$ROLE_NAME"

aws iam get-instance-profile \
  --instance-profile-name "$PROFILE_NAME"
```

Then:

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

### Step 7 — Verify ENI Cleanup

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'NetworkInterfaces[].{ID:NetworkInterfaceId,Status:Status}' \
  --output table \
  --region "$AWS_REGION"
```

Wait for the terminated instance's network interfaces to be removed.

### Step 8 — Delete Networking

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

### Step 9 — Verify Cleanup

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

Expected: both return empty arrays after cleanup finishes.

### Cleanup Summary

| Resource | Expected Result |
|---|---|
| Wrong-AZ EBS Volume | Deleted |
| Correct-AZ EBS Volume | Deleted |
| EC2 Instance | Terminated |
| EC2 Root EBS Volume | Deleted |
| Security Group | Deleted |
| IAM Role | Deleted |
| Instance Profile | Deleted |
| Route Table | Deleted |
| Public Subnet | Deleted |
| Internet Gateway | Deleted |
| Lab VPC | Deleted |
| Default VPC | Unchanged |

---

## 16. Interview Questions and Answers

### Q1. What is Amazon EBS?

Amazon EBS provides persistent block storage for Amazon EC2 instances.

### Q2. Why did the EBS attachment fail in this lab?

The EC2 instance and EBS volume were created in different Availability Zones.

### Q3. Can an EBS volume be attached across Availability Zones?

No. Standard EBS attachments require the instance and volume to be in the same Availability Zone.

### Q4. What does `available` mean for an EBS volume?

The volume exists and is not currently attached to an instance.

### Q5. What does `in-use` mean?

The EBS volume is attached or otherwise in use.

### Q6. Which CLI command attaches EBS?

`aws ec2 attach-volume`.

### Q7. Why might `/dev/sdf` appear as `/dev/nvme1n1`?

Nitro-based EC2 instances expose EBS volumes as NVMe block devices.

### Q8. How do you identify the correct EBS volume in Linux?

Use `lsblk`, NVMe serial information, and `ebsnvme-id` to match the device with its EBS volume ID.

### Q9. How do you move EBS data to another Availability Zone?

Create a snapshot and restore it as a new volume in the destination Availability Zone.

### Q10. What is the difference between attaching and mounting a volume?

Attaching exposes a block device to the instance. Mounting makes a filesystem accessible through a Linux directory.

### Q11. Should you format an EBS volume immediately after attachment?

Only after verifying the correct device and confirming it contains no data that must be preserved.

### Q12. How would you describe this lab in an interview?

I created an EC2 instance and an EBS volume in different Availability Zones to simulate a storage attachment failure.

AWS rejected the attachment request.

I investigated the instance state, volume state, and Availability Zone configuration using AWS CLI.

After identifying the AZ mismatch, I created a replacement EBS volume in the correct zone, attached it successfully, and verified that Linux detected the new NVMe block device.

Finally, I safely detached and deleted the temporary test volumes.

---

## 17. Repository Structure

```text
01-ebs-volume-not-attached/
├── README.md
└── ec2-trust-policy.json
```

### File Descriptions

| File | Purpose |
|---|---|
| `README.md` | Complete EBS attachment troubleshooting documentation |
| `ec2-trust-policy.json` | EC2 trust policy for Systems Manager access |

---

## 18. Lessons Learned

This lab demonstrates that an EBS volume can be in the `available` state but still fail to attach to an EC2 instance.

The problem was not caused by the EC2 instance being stopped, the EBS volume being unavailable, or a Linux filesystem issue.

Instead, the volume and instance were created in different Availability Zones.

The most important troubleshooting step was comparing the Availability Zones before attempting attachment.

Creating a compatible volume in the instance's Availability Zone resolved the issue.

We also learned that Nitro-based EC2 instances can expose EBS attachments using NVMe device names that differ from the AWS attachment name.

**Key Lesson:** Always verify Availability Zone compatibility, volume state, and existing attachments before attaching EBS storage to EC2.

---

## 19. Skills Demonstrated

- Amazon EBS provisioning
- EC2 volume attachment
- Availability Zone troubleshooting
- AWS CLI v2
- EBS volume state investigation
- EC2 block device mapping
- Linux NVMe device identification
- Systems Manager
- VPC networking
- IAM instance profiles
- Root cause analysis
- Safe storage remediation
- EBS detachment
- AWS resource cleanup

## Final Outcome

| Item | Expected Result |
|---|---|
| Incident | INC-EBS-001 |
| Problem | EBS volume not attached |
| Fault | Cross-AZ attachment attempt |
| Root Cause | Availability Zone mismatch |
| EC2 State | Running |
| Wrong Volume | Available, unattached |
| Correct Volume | In-use, attached |
| Linux Verification | New NVMe block device visible |
| Remediation | Create EBS in matching AZ |
| Cleanup | Temporary resources removed |
| Expected Incident Status | Resolved |

**Key Takeaway:** An EBS volume and its EC2 instance must be in compatible Availability Zones. Correct attachment begins with validating the instance location, volume state, and device mapping.
