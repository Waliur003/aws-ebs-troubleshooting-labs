# Lab 06 — EBS Encryption / KMS Access Error

## Overview

This lab demonstrates how to troubleshoot an encrypted Amazon Elastic Block Store (EBS) volume that cannot be reattached to an EC2 instance because its AWS Key Management Service (KMS) key has been disabled.

We deploy an Amazon Linux 2023 EC2 instance and create a dedicated customer-managed KMS key.

Using this key, we provision a 5 GiB encrypted gp3 EBS volume, attach it to EC2, create an ext4 filesystem, and mount it at `/data`.

After storing test application data and recording a SHA-256 checksum, we safely unmount and detach the volume.

We then intentionally disable the customer-managed KMS key and attempt to reattach the encrypted EBS volume.

The attachment cannot successfully complete because the required encryption key is unavailable.

We investigate the KMS key state, EBS encryption metadata, volume attachment status, IAM permissions, and key policy.

After identifying the root cause, we re-enable the KMS key, reattach the volume, mount the existing filesystem, and verify that the original data remains intact.

Finally, we remove the temporary AWS infrastructure and schedule deletion of the dedicated lab KMS key.

**Troubleshooting workflow:**

Provision → Encrypt → Healthy Baseline → Fault Injection → Investigation → Root Cause Analysis → Remediation → Verification → Cleanup

---

## 1. Lab Information

| Item | Configuration |
|---|---|
| Incident ID | INC-EBS-006 |
| Difficulty | Advanced |
| AWS Region | `us-east-1` |
| AWS Services | EC2, EBS, KMS, IAM, VPC, Systems Manager, CloudTrail |
| Operating System | Amazon Linux 2023 |
| Instance Type | `t3.micro` |
| EBS Volume Type | `gp3` |
| EBS Volume Size | 5 GiB |
| Encryption | AWS KMS |
| KMS Key Type | Customer-managed symmetric key |
| KMS Alias | `alias/cloud-ebs-lab06-key` |
| AWS Attachment Name | `/dev/sdf` |
| Linux Block Device | NVMe, such as `/dev/nvme1n1` |
| Filesystem | ext4 |
| Mount Point | `/data` |
| Fault Injection | Disable KMS key and attempt reattachment |
| Primary Symptom | Encrypted EBS attachment fails |
| Root Cause | KMS key disabled |
| Remediation | Re-enable KMS key |
| Verification | Successful attachment and checksum verification |
| Key Cleanup | Schedule deletion with a waiting period |

---

## 2. Production Incident Scenario

### Incident Description

A company stores application data on an encrypted Amazon EBS volume.

The volume is encrypted using a customer-managed AWS KMS key.

The encrypted volume is attached to an EC2 instance, and the application operates normally.

During an administrative operation, the KMS key is disabled.

The existing EBS attachment may continue working because the instance already has access to the data key required for its current attachment.

Later, the engineer detaches the EBS volume and attempts to attach it again.

The attachment fails because AWS cannot use the disabled KMS key for the required cryptographic operations.

The engineer must investigate whether the failure is related to the EBS attachment, KMS key state, IAM permissions, or KMS key policy.

### Incident Symptoms

- EC2 instance remains running.
- EBS volume is encrypted.
- EBS volume uses a customer-managed KMS key.
- KMS key state is `Disabled`.
- A new attachment request does not successfully complete.
- The encrypted data volume cannot be accessed through the attempted new attachment.
- The original filesystem and application data have not been intentionally modified.
- Re-enabling the key allows the required EBS cryptographic operations to resume, assuming authorization remains correct.

### Potential Business Impact

- Application storage becomes unavailable after a detach or instance replacement.
- Disaster recovery operations may fail.
- Encrypted snapshots or volumes may be unusable.
- Recovery time objectives may be missed.
- Operational teams may experience prolonged application downtime.
- Incorrect KMS policy changes may introduce security risks.

### Objective

Identify the KMS-related cause of an encrypted EBS attachment failure, restore access without weakening encryption security, and verify the original application data.

---

## 3. AWS Architecture

### Healthy Architecture

```text
                   Amazon EC2
                Amazon Linux 2023
                        |
                        v
                Encrypted EBS Volume
                    gp3 — 5 GiB
                        |
                        v
                AWS KMS Integration
                        |
                        v
                Customer-Managed Key
                   State: Enabled
                        |
                        v
                Encrypted Data Key
                  Available to EBS
                        |
                        v
                Successful Attachment
                        |
                        v
                    ext4 /data
                        |
                        v
                 Application Data
```

### Broken Architecture

```text
               Customer-Managed KMS Key
                         |
                         v
                  Key Disabled
                         |
                         v
                 Encrypted EBS Volume
                         |
                         v
                New AttachVolume Request
                         |
                         X
                Required KMS Operation
                    Cannot Complete
                         |
                         v
                 Attachment Failure
                         |
                         v
               Application Data Unavailable
```

### Recovered Architecture

```text
                   KMS Key Disabled
                          |
                          v
                   Investigation
                describe-key / policy
                          |
                          v
                   Enable KMS Key
                          |
                          v
                   KMS Key Enabled
                          |
                          v
                 Reattach EBS Volume
                          |
                          v
                 Mount Existing ext4
                          |
                          v
                Verify Original Checksum
                          |
                          v
                  Application Recovered
```

### Infrastructure Configuration

| Resource | Configuration |
|---|---|
| VPC | `10.177.0.0/16` |
| Public Subnet | `10.177.1.0/24` |
| Internet Gateway | Attached |
| Public Route | `0.0.0.0/0` through IGW |
| EC2 Security Group | No inbound access |
| EC2 Management | Systems Manager |
| Data EBS Volume | Encrypted 5 GiB gp3 |
| KMS Key | Dedicated customer-managed key |
| KMS Alias | `alias/cloud-ebs-lab06-key` |
| Filesystem | ext4 |
| Mount Directory | `/data` |

---

## 4. Understanding EBS Encryption and KMS

Amazon EBS encryption protects block storage data at rest.

AWS KMS manages the cryptographic keys used by supported EBS encryption operations.

A customer-managed KMS key allows an AWS customer to control key authorization, enabled state, and lifecycle.

### Encryption Concepts

| Component | Purpose |
|---|---|
| EBS Encryption | Protects stored volume data |
| AWS KMS | Manages cryptographic keys |
| Customer-Managed Key | Key controlled by the customer |
| Data Key | Encrypts and decrypts volume data |
| Key Policy | Defines access to a particular KMS key |
| IAM Policy | Grants or restricts principal permissions |
| KMS Grant | Delegates certain key operations |
| CloudTrail | Records supported AWS API management activity |

### How EBS Encryption Works

At a high level:

```text
EC2 Requests Encrypted Storage
             |
             v
Amazon EBS Uses AWS KMS
             |
             v
KMS Provides Cryptographic
Support for the Data Key
             |
             v
EBS Encrypts and Decrypts
Data Transparently
```

The EC2 operating system can use a normal filesystem such as ext4.

The filesystem does not need to implement EBS encryption itself.

### Important KMS Key States

| State | Meaning |
|---|---|
| `Enabled` | Key is available for permitted cryptographic operations |
| `Disabled` | Cryptographic use is blocked |
| `PendingDeletion` | Key is scheduled for deletion and unavailable for normal cryptographic use |
| `PendingImport` | Key is awaiting imported key material |
| `Unavailable` | Key is temporarily unavailable in applicable configurations |

### Why a Disabled KMS Key Matters

Disabling a KMS key prevents new cryptographic operations that require the key.

An already attached encrypted EBS volume may continue operating, but later reattachment can fail.

The actual failure may be asynchronous: the `attach-volume` API can initially return a response before AWS reports that attachment did not complete.

### KMS Access Error Categories

| Problem | Typical Cause | Remediation |
|---|---|---|
| Disabled key | Key administration changed | Re-enable authorized key |
| IAM AccessDenied | Missing required IAM permissions | Correct identity policy |
| Key-policy denial | Principal not authorized by key policy | Correct key policy |
| Missing KMS grant | Integrated AWS service lacks necessary grant | Configure permitted grant creation |
| Key pending deletion | Key scheduled for removal | Cancel deletion and re-enable if authorized |
| Wrong KMS key | Volume references unexpected key | Investigate encryption metadata |
| Cross-account access | Missing cross-account authorization | Correct IAM and key policy permissions |

---

## 5. Prerequisites and Safety

### Requirements

- AWS account
- AWS CLI v2
- Ubuntu WSL2
- Bash
- EC2 permissions
- EBS permissions
- KMS key administration permissions
- IAM permissions
- VPC permissions
- Systems Manager
- Git and GitHub

### Critical Safety Rules

**Never disable a KMS key used by production resources.**

- Create a dedicated customer-managed KMS key.
- Tag the key clearly.
- Verify the key ID and ARN before disabling it.
- Never disable the default AWS-managed EBS key.
- Never use a KMS key shared with another application for fault injection.
- Do not disable a key while important production workloads depend on it.
- Do not reformat the encrypted volume during recovery.
- Keep the original data intact.
- Verify the filesystem checksum after remediation.
- Review dependencies before scheduling key deletion.
- Delete only lab resources.

### Cost Considerations

Customer-managed KMS keys, EBS volumes, EC2 instances, public IPv4 addresses, and other resources may incur charges.

KMS key deletion is delayed by a mandatory waiting period. Scheduling deletion does not mean the key immediately disappears.

---

## 6. Prepare the AWS Environment

### Step 1 — Configure Environment Variables

Run in Ubuntu WSL2:

```bash
export AWS_REGION="us-east-1"
export PROJECT_TAG="EBS-Troubleshooting-Lab06"

export INSTANCE_NAME="cloud-ebs-lab06-ec2"
export VOLUME_NAME="cloud-ebs-lab06-encrypted"

export SG_NAME="cloud-ebs-lab06-sg"
export ROLE_NAME="cloud-ebs-lab06-ssm-role"
export PROFILE_NAME="cloud-ebs-lab06-ssm-profile"

export KEY_ALIAS="alias/cloud-ebs-lab06-key"

export AZ_A="us-east-1a"
export VOLUME_SIZE="5"
```

### Step 2 — Verify AWS Identity

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

Choose a supported AZ.

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

Check whether the planned KMS alias exists:

```bash
aws kms describe-key \
  --key-id "$KEY_ALIAS" \
  --region "$AWS_REGION"
```

A `NotFoundException` is expected for a new lab.

Do not overwrite or reuse an unknown existing key.

---

## 7. Create the Customer-Managed KMS Key

### Step 1 — Create the Key

```bash
export KMS_KEY_ID=$(aws kms create-key \
  --description "Dedicated KMS key for EBS Lab06" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --tags TagKey=Project,TagValue="$PROJECT_TAG" \
  --query 'KeyMetadata.KeyId' \
  --output text \
  --region "$AWS_REGION")

echo "$KMS_KEY_ID"
```

When no explicit key policy is supplied, AWS KMS uses its default programmatic policy.

This allows account-level IAM delegation, but the intended principal must still have appropriate KMS permissions.

### Step 2 — Create Alias

```bash
aws kms create-alias \
  --alias-name "$KEY_ALIAS" \
  --target-key-id "$KMS_KEY_ID" \
  --region "$AWS_REGION"
```

### Step 3 — Retrieve Key ARN

```bash
export KMS_KEY_ARN=$(aws kms describe-key \
  --key-id "$KMS_KEY_ID" \
  --query 'KeyMetadata.Arn' \
  --output text \
  --region "$AWS_REGION")

echo "$KMS_KEY_ARN"
```

### Step 4 — Verify Key State

```bash
aws kms describe-key \
  --key-id "$KMS_KEY_ID" \
  --query 'KeyMetadata.{ID:KeyId,State:KeyState,Enabled:Enabled,Usage:KeyUsage,Arn:Arn}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
State: Enabled
Enabled: True
Usage: ENCRYPT_DECRYPT
```

### Step 5 — Inspect Key Policy

```bash
aws kms get-key-policy \
  --key-id "$KMS_KEY_ID" \
  --policy-name default \
  --query Policy \
  --output text \
  --region "$AWS_REGION"
```

Review the policy rather than making unnecessary broad changes.

---

## 8. Create EC2 Infrastructure

### Step 1 — Create VPC

```bash
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block "10.177.0.0/16" \
  --tag-specifications \
  "ResourceType=vpc,Tags=[{Key=Name,Value=cloud-ebs-lab06-vpc},{Key=Project,Value=$PROJECT_TAG}]" \
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

### Step 2 — Create Subnet

```bash
export SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block "10.177.1.0/24" \
  --availability-zone "$AZ_A" \
  --tag-specifications \
  "ResourceType=subnet,Tags=[{Key=Name,Value=cloud-ebs-lab06-subnet},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 3 — Create Internet Gateway

```bash
export IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications \
  "ResourceType=internet-gateway,Tags=[{Key=Name,Value=cloud-ebs-lab06-igw},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'InternetGateway.InternetGatewayId' \
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
  "ResourceType=route-table,Tags=[{Key=Name,Value=cloud-ebs-lab06-rt},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --description "Lab06 encrypted EBS troubleshooting" \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
  "ResourceType=security-group,Tags=[{Key=Name,Value=$SG_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query GroupId \
  --output text \
  --region "$AWS_REGION")
```

No inbound access is required for Systems Manager.

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

### Step 7 — Create IAM Role

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

Retrieve instance AZ:

```bash
export INSTANCE_AZ=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' \
  --output text \
  --region "$AWS_REGION")
```

---

## 9. Create the Encrypted EBS Volume

### Step 1 — Create Encrypted gp3 Volume

```bash
export VOLUME_ID=$(aws ec2 create-volume \
  --availability-zone "$INSTANCE_AZ" \
  --size "$VOLUME_SIZE" \
  --volume-type gp3 \
  --encrypted \
  --kms-key-id "$KMS_KEY_ARN" \
  --tag-specifications \
  "ResourceType=volume,Tags=[{Key=Name,Value=$VOLUME_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query VolumeId \
  --output text \
  --region "$AWS_REGION")
```

### Step 2 — Wait for Volume Availability

```bash
aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 3 — Verify Encryption Metadata

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,State:State,Encrypted:Encrypted,KmsKeyId:KmsKeyId,Size:Size}' \
  --output json \
  --region "$AWS_REGION"
```

Expected:

```text
State: available
Encrypted: true
KMS: Dedicated Lab06 key ARN
Size: 5 GiB
```

### Step 4 — Attach the Volume

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

Expected: attachment state `attached`.

If the initial attachment fails, resolve the KMS or EBS authorization problem before attempting the fault injection.

---

## 10. Prepare the Linux Filesystem

### Step 1 — Connect Using Systems Manager

```bash
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
  --query 'InstanceInformationList[].{ID:InstanceId,Status:PingStatus}' \
  --output table \
  --region "$AWS_REGION"
```

Once the instance reports `Online`:

```bash
aws ssm start-session \
  --target "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

The following Linux commands run inside EC2.

### Step 2 — Identify the EBS Device

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,SERIAL
```

Compare the 5 GiB disk's NVMe serial with the EBS volume ID.

For example:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

After confirming the correct volume:

```bash
export DATA_DEVICE="/dev/nvme1n1"
```

Replace the example path with the actual verified device.

### Step 3 — Inspect Existing Filesystem

```bash
lsblk -f

sudo blkid "$DATA_DEVICE" || true

sudo file -s "$DATA_DEVICE"
```

Expected: no existing filesystem on the new blank data volume.

### Step 4 — Create ext4 Filesystem

**Destructive operation: Only run on the verified empty Lab 06 volume.**

```bash
sudo mkfs.ext4 "$DATA_DEVICE"
```

### Step 5 — Mount the Volume

```bash
sudo mkdir -p /data

sudo mount "$DATA_DEVICE" /data
```

Verify:

```bash
findmnt /data

df -hT /data
```

### Step 6 — Create Important Test Data

```bash
echo "EBS Lab 06 - Encrypted Application Data" \
  | sudo tee /data/important-data.txt
```

Record the checksum:

```bash
sha256sum /data/important-data.txt
```

### Step 7 — Verify Healthy Baseline

```bash
cat /data/important-data.txt

findmnt /data

df -hT /data
```

Expected: the file is readable and `/data` is mounted.

### Healthy Baseline Summary

| Component | Expected |
|---|---|
| KMS Key | Enabled |
| EBS Encryption | Enabled |
| EBS State | In-use |
| EBS Attachment | Attached |
| Filesystem | ext4 |
| Mount | `/data` |
| Application File | Readable |
| Checksum | Recorded |

---

## 11. Fault Injection — Disable the KMS Key

### Step 1 — Flush and Unmount the Filesystem

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

Expected: no mount.

Exit SSM:

```bash
exit
```

### Step 2 — Detach the Encrypted EBS Volume

From WSL2:

```bash
aws ec2 detach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --region "$AWS_REGION"

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

Expected:

```text
State: available
Attachments: []
```

### Step 3 — Verify the Dedicated KMS Key

```bash
aws kms list-resource-tags \
  --key-id "$KMS_KEY_ID" \
  --region "$AWS_REGION"
```

Expected project tag:

```text
Project: EBS-Troubleshooting-Lab06
```

Confirm that the volume references the same KMS key:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{Encrypted:Encrypted,KMS:KmsKeyId}' \
  --output json \
  --region "$AWS_REGION"
```

### Step 4 — Disable the Key

**Intentional fault injection. Never perform this against a shared or production KMS key.**

```bash
aws kms disable-key \
  --key-id "$KMS_KEY_ID" \
  --region "$AWS_REGION"
```

### Step 5 — Verify Disabled State

```bash
aws kms describe-key \
  --key-id "$KMS_KEY_ID" \
  --query 'KeyMetadata.{ID:KeyId,State:KeyState,Enabled:Enabled}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
State: Disabled
Enabled: False
```

Allow the state change to become effective.

### Step 6 — Attempt EBS Reattachment

```bash
aws ec2 attach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdf \
  --region "$AWS_REGION"
```

Expected: the attachment cannot successfully complete because the KMS key is disabled.

AWS may return an immediate error, or the attachment request may initially be accepted and later fail asynchronously.

**An initially successful API response is not proof that the encrypted volume was attached successfully.**

### Step 7 — Check Attachment State

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{State:State,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Poll attachment status if necessary:

```bash
for i in {1..12}; do
  aws ec2 describe-volumes \
    --volume-ids "$VOLUME_ID" \
    --query 'Volumes[0].{State:State,Attachments:Attachments}' \
    --output json \
    --region "$AWS_REGION"

  sleep 10
done
```

Expected: the encrypted attachment does not reach or maintain the healthy attached state.

### Fault Summary

| Check | Expected Finding |
|---|---|
| EC2 | Running |
| Volume Encryption | Enabled |
| KMS Key State | Disabled |
| EBS Attachment | Fails to complete |
| Linux Data Volume | Unavailable through attempted attachment |
| Original Data | Preserved |
| Root Cause | Disabled KMS key |

---

## 12. Troubleshooting Investigation

### Check 1 — Verify EBS Encryption

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,State:State,Encrypted:Encrypted,KmsKeyId:KmsKeyId,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Finding: the data volume is encrypted using the lab KMS key.

### Check 2 — Inspect Key State

```bash
aws kms describe-key \
  --key-id "$KMS_KEY_ID" \
  --query 'KeyMetadata.{ID:KeyId,State:KeyState,Enabled:Enabled,Arn:Arn}' \
  --output json \
  --region "$AWS_REGION"
```

Finding: key state is `Disabled`.

### Check 3 — Inspect KMS Key Policy

```bash
aws kms get-key-policy \
  --key-id "$KMS_KEY_ID" \
  --policy-name default \
  --query Policy \
  --output text \
  --region "$AWS_REGION"
```

Finding: the key policy should be reviewed separately from the key's enabled state.

A correct policy cannot make a disabled key usable.

### Check 4 — Identify the Calling Principal

```bash
aws sts get-caller-identity
```

Finding: identify the actual AWS IAM identity performing the volume attachment.

### Check 5 — Inspect KMS Administrative Events

Where CloudTrail event history is available:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes \
    AttributeKey=EventName,AttributeValue=DisableKey \
  --max-results 20 \
  --region "$AWS_REGION" \
  --output json
```

Find the relevant event for the dedicated lab key.

Note that CloudTrail records should be filtered and interpreted carefully. Not every AWS service failure appears as a direct user-initiated KMS decryption event.

### Check 6 — Review Attachment Status

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].Attachments' \
  --output json \
  --region "$AWS_REGION"
```

Finding: the attachment does not successfully complete while the key remains disabled.

### Investigation Summary

| Investigation | Finding |
|---|---|
| EC2 | Running |
| EBS Volume | Encrypted |
| Encryption Key | Customer-managed KMS key |
| Key State | Disabled |
| Key Policy | Requires separate authorization review |
| Attachment | Unable to complete |
| Filesystem Data | Not intentionally changed |
| Root Cause | KMS key unavailable |

---

## 13. Root Cause Analysis

### Root Cause

The customer-managed KMS key used by the encrypted EBS volume was disabled.

The volume was detached before the key was disabled.

When EC2 attempted to attach the volume again, the required KMS operations could not successfully complete.

As a result, the encrypted volume could not become available through a healthy new attachment.

### Technical Explanation

Amazon EBS uses KMS integration to protect the cryptographic data keys associated with encrypted volumes.

When a volume is already attached, disabling its KMS key may not immediately stop active I/O because the volume already has the necessary data key for that attachment.

However, a new attachment requires the relevant KMS key to be usable.

A disabled key prevents required cryptographic operations.

### Root Cause Flow

```text
Encrypted EBS Data Volume
          |
          v
Customer-Managed KMS Key
          |
          v
Key Disabled
          |
          v
Volume Reattachment Attempt
          |
          v
Required KMS Operation Fails
          |
          v
EBS Attachment Cannot Complete
          |
          v
Application Storage Unavailable
```

### Five Whys

1. **Why did the EBS attachment fail?** The required KMS cryptographic operation could not complete.
2. **Why could the cryptographic operation not complete?** The customer-managed KMS key was disabled.
3. **Why didn't the previous attachment immediately fail?** The volume already had access to the data key for its existing attachment.
4. **Why did the issue appear during reattachment?** New attachment required fresh KMS access.
5. **How can the problem be prevented?** Protect KMS administrative operations, monitor key state, and validate encryption dependencies before maintenance.

### Prevention

- Restrict KMS administrative permissions.
- Separate key administrators from routine key users.
- Monitor `DisableKey` and key deletion activity.
- Use CloudTrail and EventBridge for relevant security notifications.
- Document encrypted EBS dependencies.
- Avoid disabling keys while dependent resources exist.
- Test disaster recovery procedures.
- Apply least-privilege IAM and KMS permissions.

---

## 14. Remediation — Restore KMS Access

### Step 1 — Verify the Target Key

```bash
echo "$KMS_KEY_ID"

echo "$KMS_KEY_ARN"

echo "$VOLUME_ID"
```

Confirm the values belong to the dedicated lab resources.

### Step 2 — Re-enable KMS Key

```bash
aws kms enable-key \
  --key-id "$KMS_KEY_ID" \
  --region "$AWS_REGION"
```

### Step 3 — Verify Key State

```bash
aws kms describe-key \
  --key-id "$KMS_KEY_ID" \
  --query 'KeyMetadata.{State:KeyState,Enabled:Enabled}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
State: Enabled
Enabled: True
```

Allow the change to propagate.

### Step 4 — Verify Volume State

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{State:State,Attachments:Attachments}' \
  --output json \
  --region "$AWS_REGION"
```

Confirm the failed attachment attempt has ended and the volume is `available` before retrying.

### Step 5 — Reattach Encrypted EBS

```bash
aws ec2 attach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdf \
  --region "$AWS_REGION"
```

Wait:

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
State: in-use
Attachment: attached
```

### Step 6 — Reconnect to EC2

```bash
aws ssm start-session \
  --target "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

### Step 7 — Identify Reattached NVMe Device

Inside EC2:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,SERIAL
```

Verify the disk using the EBS volume ID.

For example:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

After confirming the correct device:

```bash
export DATA_DEVICE="/dev/nvme1n1"
```

Replace the example path if necessary.

### Step 8 — Mount Existing Filesystem

```bash
sudo mount "$DATA_DEVICE" /data
```

**Do not run `mkfs` again.**

The original filesystem and data should still exist.

### Step 9 — Verify Filesystem

```bash
findmnt /data

df -hT /data
```

Expected: existing ext4 filesystem mounted.

### Step 10 — Verify Original Data

```bash
cat /data/important-data.txt

sha256sum /data/important-data.txt
```

Expected:

```text
EBS Lab 06 - Encrypted Application Data
```

Compare the checksum with the original baseline value.

### Step 11 — Test New File Writes

```bash
echo "KMS access restored - EBS working" \
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
| KMS Key | Disabled | Enabled |
| EBS Encryption | Enabled | Enabled |
| EBS Attachment | Failed | Attached |
| Linux Filesystem | Unavailable through failed attachment | Mounted |
| Original Data | Preserved | Accessible |
| Checksum | Not verified during failure | Matches baseline |
| New Writes | Unavailable | Successful |
| Incident | Active | Resolved |

---

## 15. IAM and KMS Key Policy Investigation

A disabled KMS key is only one possible cause of encrypted EBS failures.

In production, failures can also occur because an IAM principal lacks permission or the KMS key policy does not authorize the operation.

### Important KMS Permissions

| Permission | Purpose |
|---|---|
| `kms:DescribeKey` | Inspect key information |
| `kms:Decrypt` | Use a key for permitted decryption operations |
| `kms:GenerateDataKeyWithoutPlaintext` | Generate an encrypted data key |
| `kms:ReEncryptFrom` | Permit re-encryption from the key |
| `kms:ReEncryptTo` | Permit re-encryption to the key |
| `kms:CreateGrant` | Allow an integrated AWS service to use the key under permitted conditions |

Required actions vary depending on the actual EBS operation.

### Example Least-Privilege IAM Policy

The following example is designed for a dedicated EBS operations principal and must be reviewed before use.

Save as `ebs-kms-access-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UseSpecificEBSKey",
      "Effect": "Allow",
      "Action": [
        "kms:DescribeKey",
        "kms:Decrypt",
        "kms:GenerateDataKeyWithoutPlaintext",
        "kms:ReEncryptFrom",
        "kms:ReEncryptTo"
      ],
      "Resource": "REPLACE_WITH_LAB_KMS_KEY_ARN"
    },
    {
      "Sid": "AllowGrantForAWSResource",
      "Effect": "Allow",
      "Action": "kms:CreateGrant",
      "Resource": "REPLACE_WITH_LAB_KMS_KEY_ARN",
      "Condition": {
        "Bool": {
          "kms:GrantIsForAWSResource": "true"
        }
      }
    }
  ]
}
```

Replace `REPLACE_WITH_LAB_KMS_KEY_ARN` with your dedicated key ARN.

This is a reference policy, not an instruction to attach broad permissions to the EC2 Systems Manager role.

Other applicable permissions and controls must also be evaluated.

### IAM Policy vs. KMS Key Policy

| Policy | Responsibility |
|---|---|
| IAM Identity Policy | Grants or denies permissions to an IAM identity |
| KMS Key Policy | Controls access to a particular KMS key |
| KMS Grant | Delegates specific KMS operations |
| SCP | Limits permissions within affected organization accounts |
| Permission Boundary | Limits the effective permissions of an IAM principal |

### Security Best Practices

- Avoid unrestricted `kms:*` permissions.
- Restrict KMS actions to specific keys.
- Keep key administration separate from key usage.
- Review key policies and grants.
- Use explicit permissions for AWS service integrations.
- Monitor key changes through CloudTrail.
- Investigate explicit denies and organization-level restrictions.
- Never disable a key without reviewing dependent encrypted resources.

---

## 16. Final Verification

### AWS Verification

From WSL2:

```bash
aws kms describe-key \
  --key-id "$KMS_KEY_ID" \
  --query 'KeyMetadata.{State:KeyState,Enabled:Enabled}' \
  --output table \
  --region "$AWS_REGION"

aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,State:State,Encrypted:Encrypted,KmsKeyId:KmsKeyId,Attachments:Attachments}' \
  --output json \
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

cat /data/recovery-test.txt
```

### Verification Matrix

| Check | Before Fault | During Fault | After Fix |
|---|---|---|---|
| KMS Key | Enabled | Disabled | Enabled |
| EBS Encryption | Enabled | Enabled | Enabled |
| EBS Attachment | Attached | Failed to reattach | Attached |
| Filesystem | ext4 | Unavailable through new attachment | ext4 |
| `/data` | Mounted | Not mounted | Mounted |
| Application File | Accessible | Unavailable | Accessible |
| Checksum | Original | Not tested | Unchanged |
| New Writes | Successful | Unavailable | Successful |

### Success Criteria

- Customer-managed KMS key created.
- Key alias configured.
- Key enabled state verified.
- Encrypted EBS volume created.
- Encrypted EBS attached successfully.
- ext4 filesystem mounted.
- Test application data written.
- Original checksum recorded.
- Filesystem safely unmounted.
- EBS volume detached.
- Dedicated KMS key disabled.
- KMS-related reattachment failure reproduced.
- Key state and EBS metadata investigated.
- Root cause identified.
- KMS key re-enabled.
- EBS volume successfully reattached.
- Existing filesystem mounted without formatting.
- Original checksum verified.
- Application writes recovered.
- Temporary AWS resources cleaned up.

---

## 17. Production Best Practices

### EBS Encryption Troubleshooting Checklist

1. Verify AWS account and Region.
2. Identify the affected EBS volume.
3. Confirm whether the volume is encrypted.
4. Retrieve the KMS key ARN.
5. Check the key's enabled state.
6. Inspect the EBS attachment status.
7. Identify the principal performing the operation.
8. Review IAM permissions.
9. Review KMS key policy.
10. Investigate KMS grants if applicable.
11. Check for explicit denies.
12. Review relevant CloudTrail events.
13. Restore the necessary key access.
14. Retry the attachment only when safe.
15. Verify filesystem access and application data.

### Monitoring and Prevention

- Monitor KMS administrative API events.
- Alert on unexpected `DisableKey` actions.
- Alert on key deletion scheduling.
- Maintain an inventory of encrypted EBS dependencies.
- Protect key administrators with appropriate security controls.
- Use least-privilege IAM policies.
- Apply change management to KMS administration.
- Test encrypted volume recovery procedures.
- Maintain protected backups and tested recovery options.

### Security Considerations

- Do not disable AWS-managed service keys.
- Do not grant every role unrestricted key permissions.
- Do not remove EBS encryption to work around authorization issues.
- Do not expose sensitive KMS resource details unnecessarily.
- Keep access to encryption key administration tightly controlled.
- Preserve the original encrypted data during troubleshooting.
- Review key dependencies before scheduling deletion.

---

## 18. Cleanup

**Warning:** These commands permanently delete the dedicated encrypted EBS data volume and EC2 infrastructure. The KMS key will be scheduled for deletion after a mandatory waiting period.

### Step 1 — Unmount the Filesystem

Inside EC2:

```bash
cd /

findmnt /data

sudo fuser -vm /data
```

After ensuring no important processes are using it:

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

### Step 2 — Verify Resource Ownership

From WSL2:

```bash
aws sts get-caller-identity

aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,State:State,Encrypted:Encrypted,KmsKeyId:KmsKeyId,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"

aws kms list-resource-tags \
  --key-id "$KMS_KEY_ID" \
  --region "$AWS_REGION"
```

Confirm both resources belong to Lab 06.

### Step 3 — Detach the EBS Volume

```bash
aws ec2 detach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 4 — Delete Encrypted Data Volume

```bash
aws ec2 delete-volume \
  --volume-id "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 5 — Verify Root Volume Settings

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].{RootDevice:RootDeviceName,BlockDevices:BlockDeviceMappings}' \
  --output json \
  --region "$AWS_REGION"
```

Confirm the root volume's `DeleteOnTermination` setting.

### Step 6 — Terminate EC2

```bash
aws ec2 terminate-instances \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait instance-terminated \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

### Step 7 — Delete IAM Role and Profile

Verify the resources are dedicated to Lab 06.

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

### Step 8 — Verify Network Interface Cleanup

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'NetworkInterfaces[].{ID:NetworkInterfaceId,Status:Status}' \
  --output table \
  --region "$AWS_REGION"
```

Wait until the terminated instance's ENI is removed.

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

### Step 10 — Delete KMS Alias

Verify the alias belongs to the lab key:

```bash
aws kms list-aliases \
  --key-id "$KMS_KEY_ID" \
  --region "$AWS_REGION"
```

Delete the alias:

```bash
aws kms delete-alias \
  --alias-name "$KEY_ALIAS" \
  --region "$AWS_REGION"
```

Deleting the alias does not delete the KMS key itself.

### Step 11 — Schedule KMS Key Deletion

Before scheduling deletion, confirm that the key is dedicated to Lab 06 and no resources need it.

```bash
aws kms schedule-key-deletion \
  --key-id "$KMS_KEY_ID" \
  --pending-window-in-days 7 \
  --region "$AWS_REGION"
```

Verify:

```bash
aws kms describe-key \
  --key-id "$KMS_KEY_ID" \
  --query 'KeyMetadata.{ID:KeyId,State:KeyState,DeletionDate:DeletionDate}' \
  --output table \
  --region "$AWS_REGION"
```

Expected:

```text
State: PendingDeletion
DeletionDate: Future scheduled date
```

The key is not immediately deleted.

### Step 12 — Final Cleanup Verification

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

aws kms describe-key \
  --key-id "$KMS_KEY_ID" \
  --query 'KeyMetadata.KeyState' \
  --output text \
  --region "$AWS_REGION"
```

Expected:

```text
EBS Volumes: []
Lab VPC: []
KMS Key: PendingDeletion
```

Also verify that the lab EC2 instance, root volume, IAM role, and instance profile have been removed.

### Cleanup Summary

| Resource | Expected Result |
|---|---|
| Encrypted EBS Volume | Deleted |
| EC2 Instance | Terminated |
| Root EBS Volume | Deleted |
| IAM Role | Deleted |
| Instance Profile | Deleted |
| Security Group | Deleted |
| Route Table | Deleted |
| Public Subnet | Deleted |
| Internet Gateway | Deleted |
| VPC | Deleted |
| KMS Alias | Deleted |
| Customer-Managed KMS Key | Pending deletion |
| Default VPC | Unchanged |

---

## 19. Interview Questions and Answers

### Q1. What is Amazon EBS encryption?

Amazon EBS encryption protects stored block data using encryption keys managed through AWS KMS.

### Q2. What is a customer-managed KMS key?

A KMS key created and administered by an AWS customer, including control over permissions and its lifecycle.

### Q3. Why can an encrypted EBS volume fail to attach?

The KMS key may be disabled, scheduled for deletion, unavailable, or inaccessible because of IAM or key-policy restrictions.

### Q4. Why may an already attached EBS volume continue working after its KMS key is disabled?

The volume may continue using the data key obtained for its existing attachment.

### Q5. Why can reattachment fail?

The new attachment requires usable KMS key access to recover the encrypted data key.

### Q6. How do you check a KMS key's state?

Use `aws kms describe-key`.

### Q7. How do you re-enable a disabled KMS key?

Use `aws kms enable-key`, assuming the caller has the required administrative permissions.

### Q8. Which KMS permissions are commonly relevant to encrypted EBS operations?

Permissions may include `kms:CreateGrant`, `kms:Decrypt`, `kms:DescribeKey`, `kms:GenerateDataKeyWithoutPlaintext`, and re-encryption operations.

### Q9. What is the difference between an IAM policy and a KMS key policy?

An IAM policy grants permissions to an identity. A KMS key policy controls access to a particular key and whether IAM authorization can be delegated.

### Q10. What is a KMS grant?

A KMS grant allows designated principals or AWS services to perform specific cryptographic operations under controlled permissions.

### Q11. Can a disabled KMS key be restored?

Yes. An authorized key administrator can re-enable a disabled key. A permanently deleted KMS key cannot simply be re-enabled.

### Q12. How would you explain this lab in an interview?

I deployed an EC2 instance and created a dedicated customer-managed KMS key.

I used the key to provision an encrypted gp3 EBS volume, attached the volume, and created an ext4 filesystem.

After saving application test data and recording its checksum, I safely unmounted and detached the encrypted volume.

I intentionally disabled the KMS key and attempted to reattach the EBS volume.

The new attachment could not successfully complete because the key was unavailable.

I investigated the key state, EBS encryption metadata, attachment information, IAM permissions, and key policy.

After identifying the disabled key as the root cause, I re-enabled it, reattached the volume, mounted the existing filesystem, and verified that the original application data remained unchanged.

---

## 20. Repository Structure

```text
06-ebs-kms-access-error/
├── README.md
├── ec2-trust-policy.json
└── ebs-kms-access-policy.json
```

### File Descriptions

| File | Purpose |
|---|---|
| `README.md` | Complete EBS and KMS troubleshooting documentation |
| `ec2-trust-policy.json` | EC2 trust policy for Systems Manager access |
| `ebs-kms-access-policy.json` | Example least-privilege KMS access policy for an EBS operations principal |

Do not commit AWS credentials, private keys, or sensitive data.

The KMS access policy contains a placeholder ARN that must be replaced with the intended key ARN before deployment.

---

## 21. Lessons Learned

This lab demonstrates that Amazon EBS encryption introduces a dependency on AWS KMS.

An EBS volume can be healthy and properly provisioned but still fail to attach if its encryption key is unavailable.

The issue was not caused by filesystem corruption, a wrong Availability Zone, or an incorrect EBS device name.

The customer-managed KMS key was disabled.

We learned why already attached encrypted volumes may continue functioning while new attachments fail.

We also practiced inspecting KMS key state, EBS encryption metadata, key policies, and IAM authorization.

After re-enabling the key, we restored the EBS attachment and verified the original data without formatting the filesystem again.

**Key Lesson:** Encrypted EBS troubleshooting requires investigating both storage configuration and encryption-key availability. A disabled KMS key can prevent recovery even when the underlying EBS data remains intact.

---

## 22. Skills Demonstrated

- Amazon EBS encryption
- Customer-managed AWS KMS keys
- KMS key lifecycle management
- Encrypted EBS provisioning
- KMS aliases
- KMS key policies
- IAM authorization
- KMS grants
- AWS CLI v2
- EC2 volume attachment troubleshooting
- Linux NVMe device identification
- ext4 filesystem management
- SHA-256 integrity verification
- AWS Systems Manager
- CloudTrail investigation fundamentals
- Root cause analysis
- Least-privilege security principles
- Safe AWS resource cleanup

## Final Outcome

| Item | Expected Result |
|---|---|
| Incident | INC-EBS-006 |
| Problem | Encrypted EBS attachment failure |
| EBS Volume | 5 GiB gp3 |
| Encryption | Customer-managed KMS key |
| Healthy Key State | Enabled |
| Fault State | Disabled |
| Root Cause | KMS key unavailable |
| Remediation | Re-enable KMS key |
| Final EBS State | In-use |
| Final Attachment | Successful |
| Filesystem | ext4 |
| Mount Point | `/data` |
| Original Data | Preserved |
| Checksum Verification | Matches baseline |
| Cleanup | Resources removed; key pending deletion |
| Expected Incident Status | Resolved |

**Key Takeaway:** A secure EBS recovery process must account for both storage availability and AWS KMS authorization. Successful remediation restores the required key access without weakening encryption security or risking existing data.
