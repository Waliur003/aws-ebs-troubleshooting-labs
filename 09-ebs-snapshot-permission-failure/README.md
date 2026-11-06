# Lab 09 — EBS Snapshot Permission Failure

## Overview

This lab demonstrates how to troubleshoot an AWS IAM permission error that prevents an engineer from creating an Amazon EBS snapshot.

We deploy an Amazon Linux 2023 EC2 instance with a dedicated 5 GiB gp3 EBS data volume. After creating an ext4 filesystem and storing sample application data, we prepare the volume for a consistent snapshot.

We then create a restricted IAM role with permission to inspect EBS resources but without permission to create snapshots.

Using temporary credentials from AWS STS, we assume the restricted role and attempt to create a snapshot.

AWS denies the operation with `UnauthorizedOperation`.

We investigate the calling IAM principal, attached policies, required EC2 actions, source volume ARN, snapshot resource authorization, and tag-on-create permissions.

After identifying the missing IAM permissions, we apply a narrowly scoped policy, retry snapshot creation, and verify that the snapshot reaches the `completed` state.

Finally, we confirm the original application data remains intact and clean up the temporary AWS infrastructure.

**Troubleshooting workflow:**

Provision → Create Application Data → Configure Restricted IAM Role → Fault Injection → Investigate Authorization → Root Cause Analysis → Least-Privilege Remediation → Verification → Cleanup

---

## 1. Lab Information

| Item | Configuration |
|---|---|
| Incident ID | INC-EBS-009 |
| Difficulty | Intermediate–Advanced |
| AWS Region | `us-east-1` |
| AWS Services | EC2, EBS, IAM, STS, VPC, Systems Manager |
| Operating System | Amazon Linux 2023 |
| EC2 Instance Type | `t3.micro` |
| EBS Volume Type | `gp3` |
| EBS Volume Size | 5 GiB |
| Filesystem | ext4 |
| Mount Point | `/data` |
| AWS Attachment Device | `/dev/sdf` |
| Linux Device | NVMe, such as `/dev/nvme1n1` |
| Restricted IAM Role | `cloud-ebs-lab09-snapshot-role` |
| Initial IAM Permissions | `ec2:DescribeVolumes`, `ec2:DescribeSnapshots` |
| Fault Injection | Attempt snapshot creation without permission |
| Expected Error | `UnauthorizedOperation` |
| Root Cause | Missing snapshot creation authorization |
| Remediation | Least-privilege IAM policy |
| Verification | Snapshot state `completed` |
| Data Verification | SHA-256 checksum |

---

## 2. Production Incident Scenario

### Incident Description

A company uses Amazon EBS snapshots to protect important application data.

A cloud operations engineer is responsible for creating backups of EBS volumes attached to production EC2 instances.

The engineer assumes a restricted IAM role and attempts to create a snapshot of an EBS data volume.

However, AWS rejects the operation:

```text
An error occurred (UnauthorizedOperation) when calling
the CreateSnapshot operation:

You are not authorized to perform this operation.
```

The engineer can successfully list and describe the EBS volume but cannot create its snapshot.

The infrastructure team must determine whether the issue is caused by missing IAM permissions, incorrect resource authorization, organizational restrictions, or snapshot tagging requirements.

### Incident Symptoms

- EC2 instance is running.
- EBS volume is attached.
- EBS source volume is healthy.
- The original filesystem contains valid application data.
- Restricted IAM role can describe the EBS volume.
- `CreateSnapshot` returns `UnauthorizedOperation`.
- The denied operation does not create a snapshot.
- Administrator access continues functioning normally.

### Business Impact

- Backup jobs may fail.
- Recovery points may not be created.
- Disaster recovery readiness may decrease.
- Compliance requirements may be affected.
- Operational teams may miss backup schedules.
- Incorrect permission changes may introduce security risks.

### Objective

Identify the missing permissions required for snapshot creation and restore authorized backup functionality without granting unnecessary administrative privileges.

---

## 3. AWS Architecture

### Healthy Infrastructure

```text
                   Amazon EC2
                Amazon Linux 2023
                        |
                        v
                 EBS Data Volume
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
```

### Broken Snapshot Architecture

```text
                  Cloud Engineer
                        |
                        v
                    AWS STS
                     AssumeRole
                        |
                        v
                 Restricted IAM Role
                        |
                        v
                 CreateSnapshot API
                        |
                        X
                UnauthorizedOperation
                        |
                        v
                 Snapshot Not Created
```

### Recovered Architecture

```text
                 Restricted IAM Role
                         |
                         v
                  IAM Investigation
                         |
                         v
               Least-Privilege Policy
                         |
             -------------------------
             |                       |
             v                       v
      ec2:CreateSnapshot       ec2:CreateTags
             |                       |
             -------------------------
                         |
                         v
                  CreateSnapshot
                         |
                         v
                   Snapshot Pending
                         |
                         v
                 Snapshot Completed
                         |
                         v
                   Backup Verified
```

### Resource Configuration

| Resource | Configuration |
|---|---|
| VPC | `10.180.0.0/16` |
| Public Subnet | `10.180.1.0/24` |
| Internet Gateway | Attached |
| Route Table | Internet route |
| EC2 Security Group | No inbound rules |
| EC2 Management | Systems Manager |
| Data Volume | 5 GiB gp3 |
| Restricted IAM Role | Snapshot test role |
| Snapshot Tag | `Project=EBS-Troubleshooting-Lab09` |

---

## 4. Understanding IAM Authorization

AWS IAM controls which principals can perform actions on supported AWS resources.

An IAM role contains policies defining its permissions. A principal can assume a role and receive temporary credentials through AWS STS.

### Important IAM Concepts

| Concept | Description |
|---|---|
| IAM Role | AWS identity with assumable permissions |
| Trust Policy | Controls who may assume a role |
| Identity Policy | Defines actions the role can perform |
| STS AssumeRole | Provides temporary role credentials |
| Implicit Deny | Access denied when no applicable Allow exists |
| Explicit Deny | A Deny that overrides applicable Allows |
| Resource ARN | Identifies the AWS resource |
| Condition | Restricts when permissions apply |
| SCP | Organization-level maximum-permission guardrail |
| Permission Boundary | Restricts effective IAM permissions |

### Required Snapshot Permissions

For this lab, the IAM remediation addresses three authorization requirements.

| Permission | Resource | Purpose |
|---|---|---|
| `ec2:CreateSnapshot` | Specific source volume | Authorize snapshot of the intended volume |
| `ec2:CreateSnapshot` | New snapshot resource | Authorize creation of a correctly tagged snapshot |
| `ec2:CreateTags` | New snapshot resource | Permit tags during snapshot creation |

The exact required authorization depends on the API request and applicable IAM controls.

### Important Difference

```text
DescribeVolumes Allowed
          |
          v
Volume Metadata Visible
          |
          X
CreateSnapshot Not Allowed
          |
          v
Snapshot Request Denied
```

Being able to view a resource does not mean an IAM principal can modify or back it up.

---

## 5. Prerequisites and Safety

### Requirements

- AWS account
- AWS CLI v2
- Ubuntu WSL2
- Bash
- EC2 permissions
- EBS permissions
- IAM administration permissions
- STS AssumeRole permission
- Systems Manager
- Git and GitHub

### Security Requirements

- Use only a dedicated lab IAM role.
- Never modify your normal administrator role during fault injection.
- Use temporary STS credentials.
- Never commit AWS credentials.
- Restrict snapshot creation to the intended EBS volume.
- Avoid `ec2:*` or `AdministratorAccess` as a remediation.
- Use condition keys for snapshot tagging.
- Keep snapshot deletion separate from snapshot creation.
- Review IAM policies before applying them.
- Clean up unused roles and snapshots.

### Cost Considerations

EC2 instances, EBS volumes, snapshots, root EBS storage, and public IPv4 addresses may incur charges.

---

## 6. Prepare the Environment

### Step 1 — Configure Variables

Run in Ubuntu WSL2:

```bash
export AWS_REGION="us-east-1"
export PROJECT_TAG="EBS-Troubleshooting-Lab09"

export INSTANCE_NAME="cloud-ebs-lab09-ec2"
export VOLUME_NAME="cloud-ebs-lab09-source"

export EC2_ROLE_NAME="cloud-ebs-lab09-ssm-role"
export PROFILE_NAME="cloud-ebs-lab09-ssm-profile"

export TEST_ROLE_NAME="cloud-ebs-lab09-snapshot-role"
export SG_NAME="cloud-ebs-lab09-sg"

export AZ_A="us-east-1a"
export VOLUME_SIZE="5"
```

### Step 2 — Verify AWS Identity

```bash
aws sts get-caller-identity

aws --version
```

Retrieve the account ID:

```bash
export ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)
```

### Step 3 — Verify Instance Availability

```bash
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters "Name=instance-type,Values=t3.micro" \
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

Avoid duplicate resources.

---

## 7. Create AWS Infrastructure

### Step 1 — Create VPC

```bash
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block "10.180.0.0/16" \
  --tag-specifications \
    "ResourceType=vpc,Tags=[{Key=Name,Value=cloud-ebs-lab09-vpc},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --cidr-block "10.180.1.0/24" \
  --availability-zone "$AZ_A" \
  --tag-specifications \
    "ResourceType=subnet,Tags=[{Key=Name,Value=cloud-ebs-lab09-subnet},{Key=Project,Value=$PROJECT_TAG}]" \
  --query 'Subnet.SubnetId' \
  --output text \
  --region "$AWS_REGION")
```

### Step 3 — Create Internet Gateway

```bash
export IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications \
    "ResourceType=internet-gateway,Tags=[{Key=Name,Value=cloud-ebs-lab09-igw},{Key=Project,Value=$PROJECT_TAG}]" \
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
    "ResourceType=route-table,Tags=[{Key=Name,Value=cloud-ebs-lab09-rt},{Key=Project,Value=$PROJECT_TAG}]" \
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
  --description "EBS Lab09 Snapshot Permissions" \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
    "ResourceType=security-group,Tags=[{Key=Name,Value=$SG_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query GroupId \
  --output text \
  --region "$AWS_REGION")
```

No inbound access is required.

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

### Step 7 — Create SSM Role and Profile

```bash
aws iam create-role \
  --role-name "$EC2_ROLE_NAME" \
  --assume-role-policy-document file://ec2-trust-policy.json \
  --tags Key=Project,Value="$PROJECT_TAG"

aws iam attach-role-policy \
  --role-name "$EC2_ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam create-instance-profile \
  --instance-profile-name "$PROFILE_NAME" \
  --tags Key=Project,Value="$PROJECT_TAG"

aws iam add-role-to-instance-profile \
  --instance-profile-name "$PROFILE_NAME" \
  --role-name "$EC2_ROLE_NAME"
```

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

Wait for EC2:

```bash
aws ec2 wait instance-running \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait instance-status-ok \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

### Step 10 — Retrieve Instance AZ

```bash
export INSTANCE_AZ=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' \
  --output text \
  --region "$AWS_REGION")
```

---

## 8. Create the Source EBS Volume

### Step 1 — Create 5 GiB gp3

```bash
export VOLUME_ID=$(aws ec2 create-volume \
  --availability-zone "$INSTANCE_AZ" \
  --volume-type gp3 \
  --size "$VOLUME_SIZE" \
  --tag-specifications \
    "ResourceType=volume,Tags=[{Key=Name,Value=$VOLUME_NAME},{Key=Project,Value=$PROJECT_TAG}]" \
  --query VolumeId \
  --output text \
  --region "$AWS_REGION")
```

Wait:

```bash
aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 2 — Verify Volume

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,Size:Size,State:State,AZ:AvailabilityZone}' \
  --output table \
  --region "$AWS_REGION"
```

### Step 3 — Attach to EC2

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

---

## 9. Prepare the Source Filesystem

### Step 1 — Connect Through SSM

```bash
aws ssm start-session \
  --target "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

The following commands run inside EC2.

### Step 2 — Identify the Correct Device

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,SERIAL
```

Compare its EBS serial with the source volume ID.

For example:

```bash
sudo /sbin/ebsnvme-id /dev/nvme1n1
```

After verification:

```bash
export DATA_DEVICE="/dev/nvme1n1"
```

Replace the example path with the correct device.

### Step 3 — Inspect the Blank Volume

```bash
lsblk -f

sudo blkid "$DATA_DEVICE" || true

sudo file -s "$DATA_DEVICE"
```

### Step 4 — Create ext4

**Destructive operation. Run only on the verified empty Lab 09 volume.**

```bash
sudo mkfs.ext4 "$DATA_DEVICE"
```

### Step 5 — Mount at `/data`

```bash
sudo mkdir -p /data

sudo mount "$DATA_DEVICE" /data
```

### Step 6 — Create Important Test Data

```bash
echo "EBS Lab 09 - Snapshot Permission Recovery Test" \
  | sudo tee /data/important-data.txt
```

Record checksum:

```bash
sha256sum /data/important-data.txt
```

### Step 7 — Verify Healthy Baseline

```bash
findmnt /data

df -hT /data

cat /data/important-data.txt
```

### Step 8 — Unmount Before Snapshot

```bash
cd /

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

The EBS volume remains attached, but its filesystem is unmounted.

---

## 10. Create the Restricted IAM Role

### Step 1 — Create a Test Role Trust Policy

Save as `snapshot-role-trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::REPLACE_WITH_ACCOUNT_ID:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Replace the account ID with the intended AWS account when deploying.

For a production environment, restrict trust to the specific authorized principal rather than broadly delegating to account principals.

### Step 2 — Create the Test Role

```bash
aws iam create-role \
  --role-name "$TEST_ROLE_NAME" \
  --assume-role-policy-document file://snapshot-role-trust-policy.json \
  --tags Key=Project,Value="$PROJECT_TAG"
```

### Step 3 — Create Read-Only Policy

Save as `snapshot-readonly-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DescribeEBSResources",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVolumes",
        "ec2:DescribeSnapshots"
      ],
      "Resource": "*"
    }
  ]
}
```

### Step 4 — Attach Read-Only Policy

```bash
aws iam put-role-policy \
  --role-name "$TEST_ROLE_NAME" \
  --policy-name "Lab09SnapshotReadOnly" \
  --policy-document file://snapshot-readonly-policy.json
```

### Step 5 — Retrieve Test Role ARN

```bash
export TEST_ROLE_ARN=$(aws iam get-role \
  --role-name "$TEST_ROLE_NAME" \
  --query Role.Arn \
  --output text)

echo "$TEST_ROLE_ARN"
```

### Step 6 — Assume Restricted Role

```bash
read -r TEST_ACCESS_KEY TEST_SECRET_KEY TEST_SESSION_TOKEN \
  < <(aws sts assume-role \
    --role-arn "$TEST_ROLE_ARN" \
    --role-session-name "EBSLab09Troubleshooting" \
    --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
    --output text)
```

Create a helper:

```bash
lab09aws() {
  env -u AWS_PROFILE \
    AWS_ACCESS_KEY_ID="$TEST_ACCESS_KEY" \
    AWS_SECRET_ACCESS_KEY="$TEST_SECRET_KEY" \
    AWS_SESSION_TOKEN="$TEST_SESSION_TOKEN" \
    AWS_DEFAULT_REGION="$AWS_REGION" \
    aws "$@"
}
```

### Step 7 — Verify Restricted Identity

```bash
lab09aws sts get-caller-identity
```

Expected assumed-role ARN contains:

```text
assumed-role/cloud-ebs-lab09-snapshot-role/EBSLab09Troubleshooting
```

Do not print or commit the temporary access keys.

### Step 8 — Verify Read Access

```bash
lab09aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,Size:Size,State:State}' \
  --output table
```

Expected: successful volume description.

---

## 11. Fault Injection — Snapshot UnauthorizedOperation

### Step 1 — Attempt Snapshot Creation

Using the restricted role:

```bash
lab09aws ec2 create-snapshot \
  --volume-id "$VOLUME_ID" \
  --description "Lab09 snapshot permission test" \
  --tag-specifications \
    "ResourceType=snapshot,Tags=[{Key=Name,Value=cloud-ebs-lab09-snapshot},{Key=Project,Value=$PROJECT_TAG}]"
```

### Expected Error

```text
An error occurred (UnauthorizedOperation) when calling
the CreateSnapshot operation:

You are not authorized to perform this operation.
```

### Step 2 — Confirm No Snapshot Was Created

From the normal administrator CLI:

```bash
aws ec2 describe-snapshots \
  --owner-ids self \
  --filters "Name=tag:Project,Values=$PROJECT_TAG" \
  --query 'Snapshots[].{ID:SnapshotId,State:State}' \
  --output table \
  --region "$AWS_REGION"
```

Expected: no snapshot created by the denied request.

### Step 3 — Confirm Restricted Identity

```bash
lab09aws sts get-caller-identity
```

Verify that the failed request came from the restricted test role.

### Fault Summary

| Check | Expected Finding |
|---|---|
| EBS Source Volume | Healthy |
| EBS State | In-use |
| Filesystem | Unmounted |
| Test IAM Role | Assumed |
| DescribeVolumes | Allowed |
| CreateSnapshot | Denied |
| Error | UnauthorizedOperation |
| Snapshot Created | No |
| Root Cause | Missing IAM authorization |

---

## 12. Troubleshooting Investigation

### Check 1 — Inspect the Role Policy

Using normal credentials:

```bash
aws iam get-role-policy \
  --role-name "$TEST_ROLE_NAME" \
  --policy-name "Lab09SnapshotReadOnly"
```

Finding: the role only grants `DescribeVolumes` and `DescribeSnapshots`.

### Check 2 — Inspect Caller Identity

```bash
lab09aws sts get-caller-identity
```

Finding: the caller is the restricted role session.

### Check 3 — Review Required Permissions

The snapshot operation requires appropriate `ec2:CreateSnapshot` authorization.

If the request also creates tags, it requires the corresponding tag-on-create authorization.

### Check 4 — Decode Authorization Failure When Available

If AWS returns an encoded authorization message, an authorized administrator can investigate it:

```bash
aws sts decode-authorization-message \
  --encoded-message "ENCODED_MESSAGE_FROM_ERROR" \
  --query DecodedMessage \
  --output text
```

The caller needs `sts:DecodeAuthorizationMessage`.

Do not commit decoded authorization information containing sensitive account context.

### Check 5 — Inspect Source Resource Identity

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[0].{ID:VolumeId,State:State,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"
```

Confirm the correct volume is targeted.

### Investigation Summary

| Investigation | Finding |
|---|---|
| EC2 | Running |
| EBS Volume | Healthy |
| Test Role | Assumed successfully |
| Read Access | Allowed |
| CreateSnapshot Permission | Missing |
| CreateTags Permission | Missing |
| IAM Result | Implicit deny |
| Root Cause | Insufficient permissions |

---

## 13. Root Cause Analysis

### Root Cause

The restricted IAM role does not have permission to execute `ec2:CreateSnapshot`.

The initial role policy only allows EBS resource inspection.

Because AWS IAM denies operations without an applicable Allow, the snapshot request fails.

The request also specifies tags, which require additional authorization during resource creation.

### Root Cause Flow

```text
Restricted IAM Role
         |
         v
Read-Only EC2 Policy
         |
         v
DescribeVolumes Allowed
         |
         v
CreateSnapshot Request
         |
         X
No Applicable Allow
         |
         v
Implicit Deny
         |
         v
UnauthorizedOperation
```

### Five Whys

1. **Why was snapshot creation denied?** The IAM role lacked the required authorization.
2. **Why was the permission missing?** Its original policy only granted describe operations.
3. **Why did volume inspection succeed?** `ec2:DescribeVolumes` was allowed.
4. **Why are tags relevant?** Tag-on-create requests require separate permission checks.
5. **How can the issue be prevented?** Use correctly scoped IAM policies, test backup-role authorization, and monitor failed backup operations.

### Prevention

- Use narrowly scoped permissions.
- Validate backup roles before deployment.
- Require consistent snapshot tagging.
- Separate snapshot creation from deletion.
- Review IAM and organization-level restrictions.
- Monitor failed snapshot API requests.
- Test actual snapshot completion, not only IAM policy syntax.

---

## 14. Remediation — Least-Privilege IAM Policy

### Step 1 — Determine the Resource ARNs

```bash
export SOURCE_VOLUME_ARN="arn:aws:ec2:$AWS_REGION:$ACCOUNT_ID:volume/$VOLUME_ID"

export SNAPSHOT_RESOURCE_ARN="arn:aws:ec2:$AWS_REGION::snapshot/*"
```

### Step 2 — Create Remediation Policy

Save as `snapshot-create-policy.json`, replacing placeholders with the lab-specific values:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSnapshotOfLabVolumeOnly",
      "Effect": "Allow",
      "Action": "ec2:CreateSnapshot",
      "Resource": "REPLACE_WITH_SOURCE_VOLUME_ARN"
    },
    {
      "Sid": "AllowCreationOfTaggedSnapshots",
      "Effect": "Allow",
      "Action": "ec2:CreateSnapshot",
      "Resource": "arn:aws:ec2:us-east-1::snapshot/*",
      "Condition": {
        "StringEquals": {
          "aws:RequestTag/Project": "EBS-Troubleshooting-Lab09"
        }
      }
    },
    {
      "Sid": "AllowTagsOnlyDuringSnapshotCreation",
      "Effect": "Allow",
      "Action": "ec2:CreateTags",
      "Resource": "arn:aws:ec2:us-east-1::snapshot/*",
      "Condition": {
        "StringEquals": {
          "ec2:CreateAction": "CreateSnapshot",
          "aws:RequestTag/Project": "EBS-Troubleshooting-Lab09"
        }
      }
    }
  ]
}
```

The specific volume ARN restricts which source volume may be snapshotted.

The snapshot resource permission requires the designated Project tag.

The tag-on-create statement limits `ec2:CreateTags` to the snapshot creation context.

For an actual deployment, substitute the current account, Region, and volume identifiers.

### Step 3 — Validate JSON

```bash
python3 -m json.tool snapshot-create-policy.json
```

### Step 4 — Attach Remediation Policy

```bash
aws iam put-role-policy \
  --role-name "$TEST_ROLE_NAME" \
  --policy-name "Lab09SnapshotCreate" \
  --policy-document file://snapshot-create-policy.json
```

### Step 5 — Verify Policies

```bash
aws iam list-role-policies \
  --role-name "$TEST_ROLE_NAME"
```

Expected:

```text
Lab09SnapshotReadOnly
Lab09SnapshotCreate
```

Allow IAM changes to propagate.

### Step 6 — Retry Snapshot Creation

Using the restricted role helper:

```bash
export SNAPSHOT_ID=$(lab09aws ec2 create-snapshot \
  --volume-id "$VOLUME_ID" \
  --description "Lab09 successful snapshot after IAM fix" \
  --tag-specifications \
    "ResourceType=snapshot,Tags=[{Key=Name,Value=cloud-ebs-lab09-snapshot},{Key=Project,Value=$PROJECT_TAG}]" \
  --query SnapshotId \
  --output text)

echo "$SNAPSHOT_ID"
```

If the temporary credentials have expired, obtain a new STS role session first.

### Step 7 — Wait for Snapshot Completion

Using normal administrator credentials:

```bash
aws ec2 wait snapshot-completed \
  --snapshot-ids "$SNAPSHOT_ID" \
  --region "$AWS_REGION"
```

### Step 8 — Verify Snapshot

```bash
aws ec2 describe-snapshots \
  --snapshot-ids "$SNAPSHOT_ID" \
  --query 'Snapshots[].{ID:SnapshotId,State:State,Volume:VolumeId,Size:VolumeSize,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"
```

Expected: `completed`, correct source volume, and correct Project tag.

### Step 9 — Verify Tag Restrictions

Test unauthorized tagging without creating a snapshot:

```bash
lab09aws ec2 create-snapshot \
  --dry-run \
  --volume-id "$VOLUME_ID" \
  --tag-specifications \
    "ResourceType=snapshot,Tags=[{Key=Project,Value=WrongProject}]"
```

Expected: an authorization-denied response.

### Remediation Summary

| Check | Before Remediation | After Remediation |
|---|---|---|
| DescribeVolumes | Allowed | Allowed |
| CreateSnapshot | Denied | Allowed for lab volume |
| Required Project Tag | Not authorized | Enforced |
| Wrong Project Tag | Denied | Denied |
| Snapshot Creation | Failed | Succeeded |
| Snapshot State | None | Completed |
| Original Data | Preserved | Preserved |

---

## 15. Final Data Verification

### Step 1 — Reconnect to EC2

```bash
aws ssm start-session \
  --target "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

### Step 2 — Identify Existing Data Volume

Inside EC2:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,SERIAL
```

Verify that the intended device matches the source EBS volume.

### Step 3 — Mount Existing Filesystem

After identifying the correct device:

```bash
export DATA_DEVICE="/dev/nvme1n1"

sudo mount "$DATA_DEVICE" /data
```

Replace the example device path as necessary.

**Do not run `mkfs` again.**

### Step 4 — Verify Original Data

```bash
findmnt /data

cat /data/important-data.txt

sha256sum /data/important-data.txt
```

Compare the checksum with the value recorded before snapshot creation.

### Step 5 — Verify New Writes

```bash
echo "Lab09 IAM snapshot permissions fixed" \
  | sudo tee /data/recovery-test.txt
```

Verify:

```bash
cat /data/recovery-test.txt
```

### Verification Matrix

| Check | Expected Result |
|---|---|
| Restricted Role | Assumed |
| Snapshot Authorization | Corrected |
| Snapshot Created | Yes |
| Snapshot Status | Completed |
| Snapshot Project Tag | Correct |
| Source EBS Volume | Preserved |
| Source Filesystem | ext4 |
| Original File | Accessible |
| SHA-256 Checksum | Unchanged |
| Read-Write Test | Successful |

---

## 16. Additional Production Troubleshooting

### Common Snapshot Authorization Problems

| Problem | Possible Root Cause | Investigation |
|---|---|---|
| `UnauthorizedOperation` | Missing EC2 permission | Inspect role policies |
| Access denied during tagging | Missing `ec2:CreateTags` | Inspect tag-on-create conditions |
| Snapshot allowed for wrong volume only | Incorrect resource ARN | Verify source volume ARN |
| Explicit deny | IAM/SCP/boundary restriction | Review effective authorization |
| Wrong assumed role | Incorrect credentials | Run `sts get-caller-identity` |
| Encrypted volume error | KMS authorization or other encryption issue | Inspect encryption and key permissions |
| Snapshot remains pending | Snapshot processing issue | Check status and AWS events |

### Production Best Practices

1. Identify the exact caller identity.
2. Confirm the intended EBS volume.
3. Review all applicable IAM permissions.
4. Check for explicit denies.
5. Validate the snapshot resource authorization.
6. Validate tag-on-create permissions.
7. Use resource-specific ARNs where supported.
8. Avoid unrestricted `ec2:*` permissions.
9. Verify the snapshot reaches `completed`.
10. Test restoring snapshots regularly.

### Security Considerations

- Prefer temporary role credentials.
- Use least-privilege policies.
- Limit role trust to intended principals.
- Do not commit live AWS account identifiers unnecessarily.
- Protect backup resources from unauthorized deletion.
- Monitor snapshot creation and deletion activity.
- Maintain separate backup and restore procedures.

---

## 17. Cleanup

**Warning:** These commands permanently delete the dedicated Lab 09 snapshot, data volume, IAM roles, and EC2 infrastructure.

Verify every resource belongs to `EBS-Troubleshooting-Lab09` before deleting it.

### Step 1 — Unmount the Data Volume

Inside EC2:

```bash
cd /

sync

sudo umount /data

findmnt /data
```

Exit SSM:

```bash
exit
```

### Step 2 — Verify Snapshot Ownership

From WSL2 using normal credentials:

```bash
aws ec2 describe-snapshots \
  --snapshot-ids "$SNAPSHOT_ID" \
  --query 'Snapshots[].{ID:SnapshotId,Volume:VolumeId,Tags:Tags}' \
  --output json \
  --region "$AWS_REGION"
```

### Step 3 — Delete Snapshot

```bash
aws ec2 delete-snapshot \
  --snapshot-id "$SNAPSHOT_ID" \
  --region "$AWS_REGION"
```

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

### Step 5 — Delete EBS Volume

```bash
aws ec2 delete-volume \
  --volume-id "$VOLUME_ID" \
  --region "$AWS_REGION"
```

### Step 6 — Remove Temporary Credentials

```bash
unset TEST_ACCESS_KEY TEST_SECRET_KEY TEST_SESSION_TOKEN

unset -f lab09aws
```

### Step 7 — Delete Test IAM Policies

Verify the role's project tag first:

```bash
aws iam list-role-tags \
  --role-name "$TEST_ROLE_NAME"
```

Delete inline policies:

```bash
aws iam delete-role-policy \
  --role-name "$TEST_ROLE_NAME" \
  --policy-name "Lab09SnapshotReadOnly"

aws iam delete-role-policy \
  --role-name "$TEST_ROLE_NAME" \
  --policy-name "Lab09SnapshotCreate"
```

### Step 8 — Delete Test Role

```bash
aws iam delete-role \
  --role-name "$TEST_ROLE_NAME"
```

### Step 9 — Verify Root EBS Termination Setting

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings' \
  --output json \
  --region "$AWS_REGION"
```

Confirm the root volume uses `DeleteOnTermination=true`.

### Step 10 — Terminate EC2

```bash
aws ec2 terminate-instances \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"

aws ec2 wait instance-terminated \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION"
```

### Step 11 — Delete SSM IAM Resources

Verify the SSM role and instance profile belong to Lab 09.

```bash
aws iam list-role-tags \
  --role-name "$EC2_ROLE_NAME"

aws iam get-instance-profile \
  --instance-profile-name "$PROFILE_NAME"
```

Then:

```bash
aws iam detach-role-policy \
  --role-name "$EC2_ROLE_NAME" \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam remove-role-from-instance-profile \
  --instance-profile-name "$PROFILE_NAME" \
  --role-name "$EC2_ROLE_NAME"

aws iam delete-instance-profile \
  --instance-profile-name "$PROFILE_NAME"

aws iam delete-role \
  --role-name "$EC2_ROLE_NAME"
```

### Step 12 — Check Network Interfaces

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'NetworkInterfaces[].{ID:NetworkInterfaceId,Status:Status}' \
  --output table \
  --region "$AWS_REGION"
```

Wait until the instance network interfaces are removed.

### Step 13 — Delete Networking

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

### Step 14 — Verify Cleanup

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

Also verify the EC2 instance, root volume, and both dedicated IAM roles have been removed.

### Cleanup Summary

| Resource | Expected Result |
|---|---|
| EBS Snapshot | Deleted |
| EBS Data Volume | Deleted |
| EC2 Instance | Terminated |
| EC2 Root Volume | Deleted |
| Restricted IAM Role | Deleted |
| Restricted IAM Policies | Deleted |
| SSM IAM Role | Deleted |
| Instance Profile | Deleted |
| Security Group | Deleted |
| Route Table | Deleted |
| Subnet | Deleted |
| Internet Gateway | Deleted |
| Lab VPC | Deleted |
| Default VPC | Unchanged |

---

## 18. Interview Questions and Answers

### Q1. What causes `UnauthorizedOperation` during EBS snapshot creation?

The calling IAM principal may lack the required EC2 permission, or another applicable policy may explicitly deny it.

### Q2. Which IAM action allows snapshot creation?

`ec2:CreateSnapshot`.

### Q3. Why might CreateSnapshot require authorization on multiple resources?

Because the operation uses an existing EBS volume and creates a new snapshot resource.

### Q4. Why is `ec2:CreateTags` required?

When snapshot tags are supplied during resource creation, AWS performs a tag-on-create authorization check.

### Q5. What is an implicit deny?

An operation is denied because no applicable policy allows it.

### Q6. What is an explicit deny?

A policy specifically denies the operation, overriding applicable Allows.

### Q7. How do you identify the IAM principal making a request?

Run `aws sts get-caller-identity`.

### Q8. What does `sts:AssumeRole` do?

It allows an authorized principal to obtain temporary credentials for an IAM role.

### Q9. What is the purpose of IAM conditions?

They restrict the circumstances under which a policy grants permissions.

### Q10. What does `ec2:CreateAction` do?

It identifies the resource creation action during tag-on-create authorization.

### Q11. Why shouldn't you simply grant `ec2:*`?

Because it grants unrelated permissions and violates least-privilege principles.

### Q12. How would you explain this lab in an interview?

I deployed an EC2 instance with a dedicated EBS data volume and created sample application data.

I configured a restricted IAM role that could describe EBS resources but could not create snapshots.

After assuming the role through AWS STS, I reproduced an `UnauthorizedOperation` error when calling `CreateSnapshot`.

I investigated the active IAM identity, source volume ARN, role policies, and snapshot tagging requirements.

I identified missing `ec2:CreateSnapshot` and `ec2:CreateTags` permissions.

Instead of granting broad EC2 access, I created a narrowly scoped IAM policy that authorized snapshot creation for the intended volume and required the proper Project tag.

After applying the policy, I retried snapshot creation and verified that the snapshot reached `completed`.

Finally, I checked the original EBS filesystem and verified that the application data remained intact.

---

## 19. Repository Structure

```text
09-ebs-snapshot-permission-failure/
├── README.md
├── ec2-trust-policy.json
├── snapshot-role-trust-policy.json
├── snapshot-readonly-policy.json
└── snapshot-create-policy.json
```

### File Descriptions

| File | Purpose |
|---|---|
| `README.md` | Complete snapshot IAM troubleshooting documentation |
| `ec2-trust-policy.json` | EC2 trust policy for Systems Manager |
| `snapshot-role-trust-policy.json` | Trust policy for the restricted snapshot role |
| `snapshot-readonly-policy.json` | Initial policy reproducing the permission failure |
| `snapshot-create-policy.json` | Least-privilege remediation policy |

**Security note:** Store reusable placeholders in committed policies. Do not commit actual AWS credentials or session tokens. Replace the lab-specific ARN placeholders with real resource identifiers only when deploying.

---

## 20. Lessons Learned

This lab demonstrates that AWS snapshot creation can fail even when the EBS source volume is healthy and accessible.

The problem was not related to the filesystem, volume attachment, storage capacity, or AWS service availability.

The restricted IAM role simply lacked the required permissions to create a snapshot.

We learned how to reproduce an IAM `UnauthorizedOperation` error safely using a dedicated test role and temporary STS credentials.

We also practiced investigating principal identity, IAM policies, resource ARNs, and tag-on-create conditions.

After applying the least-privilege policy, snapshot creation succeeded without granting broad EC2 administration privileges.

**Key Lesson:** When EBS snapshot creation returns `UnauthorizedOperation`, investigate the caller identity, exact denied action, source volume, snapshot resource, and tagging permissions before changing IAM policies.

---

## 21. Skills Demonstrated

- Amazon EBS snapshot management
- AWS IAM role creation
- IAM trust policies
- IAM identity policies
- AWS STS AssumeRole
- Temporary credentials
- IAM implicit deny
- IAM explicit deny
- EC2 resource-level authorization
- Snapshot resource ARNs
- Tag-on-create permissions
- IAM condition keys
- Least-privilege remediation
- AWS CLI v2
- Linux ext4 filesystem management
- SHA-256 data verification
- Root cause analysis
- AWS infrastructure cleanup

## Final Outcome

| Item | Expected Result |
|---|---|
| Incident | INC-EBS-009 |
| Problem | Snapshot creation denied |
| Source EBS | 5 GiB gp3 |
| Initial IAM Access | Read-only |
| Fault | Missing snapshot creation permission |
| Error | UnauthorizedOperation |
| Root Cause | IAM implicit deny |
| Remediation | Scoped IAM policy |
| CreateSnapshot | Authorized for intended volume |
| Tag-on-create | Authorized with conditions |
| Snapshot State | Completed |
| Original Data | Preserved |
| Cleanup | Temporary resources removed |
| Expected Incident Status | Resolved |

**Key Takeaway:** Secure cloud troubleshooting requires fixing the specific authorization problem—not granting administrator permissions. Correctly scoped IAM policies restore EBS snapshot functionality while maintaining least-privilege security.
