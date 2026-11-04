# AWS EBS Troubleshooting Labs

## Overview

This repository contains 10 hands-on Amazon Elastic Block Store (EBS) troubleshooting labs designed to simulate real-world cloud engineering and cloud security incidents.

Each lab focuses on a common storage problem that AWS engineers may encounter in production, including volume attachment failures, Linux mounting issues, full filesystems, resizing problems, snapshot restoration, KMS permissions, performance bottlenecks, and disaster recovery.

The goal is to practice identifying storage failures, investigating root causes, applying safe remediation, and verifying recovery using AWS CLI and Linux commands.

The labs progress from beginner to advanced and use isolated AWS resources wherever possible.

**Troubleshooting workflow:**

Provision → Healthy Baseline → Fault Injection → Investigation → Root Cause Analysis → Remediation → Verification → Cleanup

---

## 1. Technologies Used

| Technology | Purpose |
|---|---|
| Amazon EBS | Persistent block storage |
| Amazon EC2 | Linux compute environment |
| AWS CLI v2 | EBS and EC2 management |
| Amazon Linux 2023 | Backend operating system |
| Ubuntu WSL2 | Local command-line environment |
| Linux Bash | Troubleshooting and automation |
| AWS KMS | EBS encryption and key management |
| EBS Snapshots | Backup and recovery |
| Amazon CloudWatch | Storage performance monitoring |
| AWS IAM | Access control |
| AWS Systems Manager | Secure EC2 access |
| Git and GitHub | Version control and documentation |

---

## 2. Repository Structure

```text
aws-ebs-troubleshooting-labs/
│
├── README.md
│
├── 01-ebs-volume-not-attached/
│   └── README.md
│
├── 02-linux-volume-not-mounted/
│   └── README.md
│
├── 03-filesystem-full/
│   └── README.md
│
├── 04-ebs-resize-not-reflected/
│   └── README.md
│
├── 05-snapshot-restore-failure/
│   └── README.md
│
├── 06-ebs-kms-access-error/
│   └── README.md
│
├── 07-ebs-volume-stuck-in-use/
│   └── README.md
│
├── 08-ebs-performance-bottleneck/
│   └── README.md
│
├── 09-snapshot-permission-failure/
│   └── README.md
│
└── 10-volume-recovery-after-instance-failure/
    └── README.md
```

Each lab will include the AWS architecture, setup, fault injection, investigation commands, expected observations, remediation, verification, cleanup, and interview questions.

Supporting Bash scripts, JSON policies, and configuration files will be added when needed.

---

## 3. Troubleshooting Labs

| Lab | Scenario | Difficulty |
|---|---|---|
| [01](01-ebs-volume-not-attached/) | EBS Volume Not Attached | Beginner |
| [02](02-linux-volume-not-mounted/) | Linux Volume Not Mounted | Beginner |
| [03](03-filesystem-full/) | Filesystem Full | Beginner |
| [04](04-ebs-resize-not-reflected/) | EBS Resize Not Reflected in Linux | Intermediate |
| [05](05-snapshot-restore-failure/) | EBS Snapshot Restore Failure | Intermediate |
| [06](06-ebs-kms-access-error/) | EBS Encryption / KMS Access Error | Advanced |
| [07](07-ebs-volume-stuck-in-use/) | EBS Volume Stuck in Use | Intermediate |
| [08](08-ebs-performance-bottleneck/) | EBS Performance Bottleneck | Advanced |
| [09](09-snapshot-permission-failure/) | EBS Snapshot Permission Failure | Advanced |
| [10](10-volume-recovery-after-instance-failure/) | EBS Volume Recovery After Instance Failure | Advanced |

---

## 4. Lab Descriptions

### Lab 01 — EBS Volume Not Attached

**Scenario:** An EC2 instance requires additional storage, but an EBS volume is not attached successfully.

**Troubleshooting focus:**

- Inspect EBS volume state.
- Identify the volume and instance Availability Zones.
- Investigate attachment errors.
- Verify device names.
- Attach the volume to the correct instance.
- Confirm the volume becomes visible in Linux.

**Key commands:** `describe-volumes`, `attach-volume`, `lsblk`.

### Lab 02 — Linux Volume Not Mounted

**Scenario:** An EBS volume is attached to EC2, but the filesystem is not accessible through its expected mount directory.

**Troubleshooting focus:**

- Identify the block device.
- Inspect filesystem types.
- Create a filesystem on a verified empty lab volume.
- Mount the filesystem.
- Verify disk accessibility.
- Configure persistent mounting through `/etc/fstab`.

**Key commands:** `lsblk`, `blkid`, `file`, `mount`, `df -h`, `findmnt`.

### Lab 03 — Filesystem Full

**Scenario:** An application fails to write new files because its EBS-backed filesystem has insufficient free space.

**Troubleshooting focus:**

- Monitor disk usage.
- Identify large files and directories.
- Investigate inode exhaustion.
- Simulate storage pressure on a dedicated lab volume.
- Remove disposable test data.
- Verify normal application writes.

**Key commands:** `df -h`, `df -i`, `du`, `find`, `lsblk`.

### Lab 04 — EBS Resize Not Reflected in Linux

**Scenario:** The EBS volume size has been increased in AWS, but Linux still reports the previous filesystem capacity.

**Troubleshooting focus:**

- Check the volume size in AWS.
- Inspect partitions and filesystems.
- Identify ext4 versus XFS.
- Extend partitions when necessary.
- Grow the filesystem.
- Verify increased storage capacity.

**Key commands:** `modify-volume`, `describe-volumes-modifications`, `lsblk`, `growpart`, `resize2fs`, `xfs_growfs`.

### Lab 05 — EBS Snapshot Restore Failure

**Scenario:** An engineer creates a volume from an EBS snapshot but cannot access the expected data.

**Troubleshooting focus:**

- Verify snapshot completion.
- Create a new volume from a snapshot.
- Attach it in the correct Availability Zone.
- Identify the restored filesystem.
- Mount existing data without reformatting.
- Confirm file integrity and recovery.

**Key commands:** `create-snapshot`, `create-volume`, `attach-volume`, `blkid`, `mount`.

### Lab 06 — EBS Encryption / KMS Access Error

**Scenario:** An encrypted EBS volume cannot be attached or used because the required AWS KMS permissions or key access are missing.

**Troubleshooting focus:**

- Inspect EBS encryption.
- Identify the associated KMS key.
- Review IAM permissions and key policies.
- Investigate relevant KMS access failures.
- Correct key access in a controlled test environment.
- Verify encrypted volume operations.

**Key commands:** `describe-volumes`, `describe-key`, `get-key-policy`, `simulate-principal-policy`, `attach-volume`.

### Lab 07 — EBS Volume Stuck in Use

**Scenario:** An EBS volume remains in the `in-use` state and cannot be attached to the intended instance.

**Troubleshooting focus:**

- Inspect volume attachments.
- Identify the currently attached instance.
- Check filesystem and application activity.
- Unmount safely when appropriate.
- Investigate attachment and detachment state.
- Reattach the volume after safe release.

**Key commands:** `describe-volumes`, `findmnt`, `fuser`, `umount`, `detach-volume`.

**Safety:** Never use forced detachment as a routine fix. Forced detachment can cause filesystem corruption or data loss.

### Lab 08 — EBS Performance Bottleneck

**Scenario:** An application experiences slow disk I/O due to unsuitable EBS performance settings or workload saturation.

**Troubleshooting focus:**

- Investigate read/write latency.
- Check provisioned IOPS and throughput.
- Inspect CloudWatch EBS metrics.
- Generate controlled disk workloads.
- Compare workload demand with storage limits.
- Tune volume performance where appropriate.
- Verify improvement.

**Key commands:** `iostat`, `fio`, `describe-volumes`, `modify-volume`, CloudWatch metrics.

### Lab 09 — EBS Snapshot Permission Failure

**Scenario:** An engineer cannot create, copy, describe, or share a snapshot because of IAM, KMS, or snapshot sharing restrictions.

**Troubleshooting focus:**

- Inspect IAM identity and permissions.
- Investigate snapshot access controls.
- Review encrypted snapshot restrictions.
- Identify missing IAM or KMS permissions.
- Apply least-privilege policies.
- Verify the required snapshot operation.

**Key commands:** `create-snapshot`, `describe-snapshots`, `modify-snapshot-attribute`, `get-key-policy`.

### Lab 10 — EBS Volume Recovery After Instance Failure

**Scenario:** An EC2 instance becomes unusable, but important application data remains on its EBS volume.

**Troubleshooting focus:**

- Identify the affected EC2 instance and volumes.
- Preserve recoverable data.
- Create snapshots.
- Prepare a replacement EC2 instance.
- Restore or safely reattach storage.
- Mount existing filesystems.
- Validate recovered application data.

**Key commands:** `describe-instances`, `describe-volumes`, `create-snapshot`, `create-volume`, `attach-volume`, `mount`.

---

## 5. Common EBS Troubleshooting Commands

### AWS CLI

```bash
# List EBS volumes
aws ec2 describe-volumes

# Inspect a particular volume
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID"

# Inspect volume status
aws ec2 describe-volume-status \
  --volume-ids "$VOLUME_ID"

# Attach an EBS volume
aws ec2 attach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdf

# List snapshots
aws ec2 describe-snapshots \
  --owner-ids self

# Inspect volume modification
aws ec2 describe-volumes-modifications \
  --volume-ids "$VOLUME_ID"
```

### Linux Storage Commands

```bash
# List block devices
lsblk

# Display filesystem types
lsblk -f

# Check disk usage
df -h

# Check inode usage
df -i

# Identify mounted filesystems
findmnt

# Inspect filesystem UUIDs
sudo blkid

# Inspect a block device
sudo file -s /dev/nvme1n1

# View disk I/O statistics (sysstat required)
iostat -xz 1
```

**Important:** Nitro-based EC2 instances often expose EBS devices using NVMe names such as `/dev/nvme1n1`, even when the attachment uses `/dev/sdf`. Always identify the correct volume before formatting, mounting, resizing, or detaching it.

---

## 6. EBS Troubleshooting Methodology

Every lab follows the same investigation process.

### Step 1 — Identify the Symptom

Understand what is failing: attachment, mounting, capacity, I/O, encryption, permissions, or recovery.

### Step 2 — Check AWS Resource State

Inspect EC2 instances, EBS volumes, snapshots, attachments, and associated KMS keys.

### Step 3 — Investigate Linux

Use operating system commands to inspect block devices, filesystems, mount points, and I/O activity.

### Step 4 — Determine the Root Cause

Identify the underlying configuration or operational problem using evidence.

### Step 5 — Apply Safe Remediation

Correct the issue without unnecessarily risking data loss.

### Step 6 — Verify Recovery

Confirm the EBS volume, filesystem, and application behave as expected.

### Step 7 — Clean Up

Remove dedicated test resources after verifying that no important data is required.

---

## 7. Safety and Data Protection

Storage troubleshooting can be destructive if incorrect commands are executed.

Follow these rules:

- Use dedicated disposable EBS volumes for fault injection.
- Never run `mkfs` on a volume containing data you need to preserve.
- Verify the device and volume ID before making filesystem changes.
- Never fill the production root filesystem to reproduce a disk-full incident.
- Create snapshots before risky modifications.
- Avoid force-detaching mounted volumes.
- Preserve original data when testing snapshot restoration.
- Use temporary test KMS keys and restrictive IAM permissions for encryption labs.
- Verify ownership before deleting snapshots and volumes.
- Remember that EBS volumes, snapshots, provisioned performance, and related AWS resources may incur charges.

---

## 8. Expected Learning Outcomes

After completing these labs, you should be able to:

- Diagnose EBS volume attachment errors.
- Mount and troubleshoot Linux block storage.
- Investigate filesystem capacity and inode problems.
- Expand EBS volumes and Linux filesystems.
- Restore data from EBS snapshots.
- Troubleshoot encrypted volumes and KMS access.
- Resolve common attachment-state problems safely.
- Investigate disk performance bottlenecks.
- Diagnose snapshot permission failures.
- Recover EBS data after EC2 failures.
- Document incidents using a structured troubleshooting process.

---

## 9. Production Engineering Practices

These labs also emphasize:

- Root cause analysis
- Least-privilege IAM and KMS access
- Backup and recovery
- CloudWatch monitoring
- Change management
- Linux administration
- Data integrity
- Safe infrastructure cleanup
- AWS CLI automation
- GitHub documentation

The goal is to build practical troubleshooting experience relevant to Cloud Engineer, Cloud Operations Engineer, Infrastructure Engineer, and Cloud Security Engineer roles.

---

## 10. References

Official AWS documentation:

- [Amazon EBS User Guide](https://docs.aws.amazon.com/ebs/latest/userguide/)
- [Attach an EBS volume](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-attaching-volume.html)
- [NVMe EBS volumes](https://docs.aws.amazon.com/ebs/latest/userguide/nvme-ebs-volumes.html)
- [Extend the filesystem after resizing](https://docs.aws.amazon.com/ebs/latest/userguide/recognize-expanded-volume-linux.html)

---

## Final Goal

Build a practical Amazon EBS troubleshooting portfolio that demonstrates the ability to diagnose, recover, secure, and maintain persistent cloud storage.

**Repository:** `aws-ebs-troubleshooting-labs`

**Total Labs:** 10

**Focus:** AWS Cloud Engineering, Linux Storage Troubleshooting, Cloud Operations, and Cloud Security.
