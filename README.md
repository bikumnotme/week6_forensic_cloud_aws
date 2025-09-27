# AWS Forensic CLI Toolkit — **Ubuntu/WSL Expanded** (with `coldsnap-user` IAM setup)

> Purpose: End‑to‑end DFIR workflow for EC2/EBS on Ubuntu/WSL, including **creating `coldsnap-user` and setting roles/policies** for EBS Direct API download with **coldsnap**.  
> This extends your original notes and preserves command parity with Windows where useful. 【7†source】

---

## 0) Prereqs (Ubuntu/WSL)

```bash
# Update base packages
sudo apt update && sudo apt install -y git curl build-essential jq

# AWS CLI already installed? If not:
#   https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
aws --version

# Verify profiles and identity
aws configure list-profiles
aws --no-cli-pager sts get-caller-identity --profile <your-admin-or-SSO-profile>
```

_Always use a separate admin/SSO profile for IAM changes; don’t create users with a low‑privileged profile._

---

## 1) Install **coldsnap** (Ubuntu/WSL)

```bash
# Install Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"

# Build and install coldsnap
git clone https://github.com/awslabs/coldsnap.git
cd coldsnap
cargo install --locked coldsnap
coldsnap --help
```

---

## 2) Create IAM user **coldsnap-user** (CLI)

> ⚠️ **Principle of Least Privilege**: Prefer a narrowly scoped inline policy over `AmazonEC2FullAccess`.

### 2.1 Create the user (no console access)

```bash
ADMIN_PROFILE=<your-admin-or-SSO-profile>

aws iam create-user   --user-name coldsnap-user   --profile "$ADMIN_PROFILE"
```

### 2.2 Create **access key** for programmatic use

```bash
aws iam create-access-key   --user-name coldsnap-user   --profile "$ADMIN_PROFILE"
```

This returns `AccessKeyId` and `SecretAccessKey`. Immediately store them in a dedicated AWS CLI profile:

```bash
aws configure --profile coldsnap-user
# AWS Access Key ID [None]: AKIA...
# AWS Secret Access Key [None]: ********************************************************
# Default region name [None]: ap-southeast-2
# Default output format [None]: json
```

_(You can use `aws configure set ... --profile coldsnap-user` to script this.)_

---

## 3) Attach **least‑privilege** inline policy for EBS Direct API

Create a local policy file:

```bash
cat > coldsnap-ebs-policy.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EbsDirectReadBlocks",
      "Effect": "Allow",
      "Action": [
        "ebs:ListSnapshotBlocks",
        "ebs:GetSnapshotBlock"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DescribeSnapshotsOptional",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSnapshots"
      ],
      "Resource": "*"
    }
  ]
}
JSON
```

Attach it as an **inline user policy**:

```bash
aws iam put-user-policy   --user-name coldsnap-user   --policy-name ColdSnapEBSAccessLeastPriv   --policy-document file://coldsnap-ebs-policy.json   --profile "$ADMIN_PROFILE"
```

### (Optional) Attach broader AWS‑managed policy

If your environment mandates it, you can also attach `AmazonEC2FullAccess`:

```bash
aws iam attach-user-policy   --user-name coldsnap-user   --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess   --profile "$ADMIN_PROFILE"
```

---

## 4) Test the **coldsnap-user** identity

```bash
aws --no-cli-pager sts get-caller-identity --profile coldsnap-user
```

If you’re using AWS SSO elsewhere, keep `coldsnap-user` as a plain access‑key profile only for the download operation.

---

## 5) Download a snapshot with **coldsnap** (EBS Direct APIs)

```bash
# Example variables
REGION=ap-southeast-2
SNAP_ID=snap-0d5f58de9f1981d45
OUT=/mnt/d/backupppp/backup/Digital\ Forensic/week6/forensic/disk.img

mkdir -p /mnt/forensic
coldsnap --profile coldsnap-user --region "$REGION" download "$SNAP_ID" "$OUT"
```

---

## 6) Traditional path (for reference): Create volume → Attach → `dd`

_(Kept for parity; execute from Ubuntu/WSL with an admin profile tied to the investigation instance.)_

```bash
# Create a volume from the snapshot
aws --no-cli-pager ec2 create-volume   --region "$REGION" --profile "$ADMIN_PROFILE"   --snapshot-id "$SNAP_ID"   --availability-zone ap-southeast-2a   --volume-type gp3 --encrypted   --tag-specifications "ResourceType=volume,Tags=[{Key=Purpose,Value=DFIR}]"   --query VolumeId --output text

# Attach to an analysis EC2
aws --no-cli-pager ec2 attach-volume   --region "$REGION" --profile "$ADMIN_PROFILE"   --volume-id vol-XXXXXXXX   --instance-id i-YYYYYYYY   --device /dev/sdf

# Image the block device (adjust device name as presented by the kernel)
sudo dd if=/dev/nvme1n1 of=/mnt/forensic/vol-XXXXXXXX.img bs=4M status=progress
```

---

## 7) Cleanup (Security hygiene)

```bash
# Deactivate or delete the access key (prefer deactivate first)
aws iam list-access-keys --user-name coldsnap-user --profile "$ADMIN_PROFILE"
aws iam update-access-key --user-name coldsnap-user --access-key-id <KEY_ID> --status Inactive --profile "$ADMIN_PROFILE"
aws iam delete-access-key --user-name coldsnap-user --access-key-id <KEY_ID> --profile "$ADMIN_PROFILE"

# Detach broad policy if attached
aws iam detach-user-policy   --user-name coldsnap-user   --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess   --profile "$ADMIN_PROFILE"

# Remove inline least-priv policy
aws iam delete-user-policy   --user-name coldsnap-user   --policy-name ColdSnapEBSAccessLeastPriv   --profile "$ADMIN_PROFILE"

# Delete user (only when fully done)
aws iam delete-user --user-name coldsnap-user --profile "$ADMIN_PROFILE"
```

---

## 8) Troubleshooting (Ubuntu/WSL)

- **WSL file paths**: Use Linux paths (e.g., `/mnt/d/forensics/disk.img`) not Windows `D:\` inside WSL.
- **Device naming**: On Nitro instances, EBS devices surface as `/dev/nvme*n*`. Confirm via `lsblk`.
- **403 AccessDenied on coldsnap**: Ensure the inline policy with `ebs:GetSnapshotBlock` and `ebs:ListSnapshotBlocks` is attached to `coldsnap-user` and that the snapshot is in the same account/region.
- **Rate/throughput**: Use local NVMe on the analysis host and avoid tiny instance types for faster `dd`/download.

---


# AWS Forensic Tasks — Snapshots, Images & Evidence Preservation (2025 Update)

_This guide focuses on **AWS-specific** forensic acquisition and analysis of EBS snapshots, AMIs, and related logs. It updates and extends your original multi‑platform guide【22†source】 with AWS‑native workflows suitable for incident response and teaching._

> **Legal & Chain of Custody** — Maintain authority, document every action, hash artifacts, and preserve originals (see core principles in your source guide【22†source】).

---

## 0) What’s new / why this matters (2025)

- **EBS Direct APIs** — Read **snapshot blocks directly** without creating a volume or instance. Great for low‑touch hashing, sampling, and diffing snapshots (`list-snapshot-blocks`, `list-changed-blocks`, `get-snapshot-block`). Not available on **archived** snapshots. citeturn1view0
- **Export AMIs to VM files** — Use **`export-image`** (VM Import/Export) to export an **AMI** to S3 as OVA/VMDK/VHD for offline tools. citeturn0search1turn0search7turn0search13turn0search18
- **CloudTrail Lake** — Query API activity with SQL for precise timeline/reconstruction. citeturn0search2turn0search8
- **Recycle Bin & Archive Tier** — Prevent/undo deletions and lower long‑term cost; be aware that **archived snapshots can’t be read via EBS Direct APIs**. citeturn0search3turn0search9turn0search10turn1view0
- **IAM Identity Center (ex‑SSO)** — Modern workforce access used in many orgs you’ll investigate. citeturn0search5turn0search11

---

## 1) Rapid triage: find who did what, when

### A. Enumerate snapshots & AMIs
```bash
# Snapshots you own
aws ec2 describe-snapshots --owner-ids self \
  --query "Snapshots[].{ID:SnapshotId, Start:StartTime, Encrypted:Encrypted, Desc:Description, KmsKeyId:KmsKeyId}"

# AMIs you own
aws ec2 describe-images --owners self \
  --query "Images[].{ID:ImageId, Name:Name, CreationDate:CreationDate, State:State, Encrypted:BlockDeviceMappings[0].Ebs.Encrypted}"
```

### B. Pull CloudTrail evidence (API actors & IPs)
- **CreateSnapshot/CopySnapshot/CreateImage/ExportImage/AttachVolume/RunInstances** are key events.
- Use **CloudTrail Lake** for SQL queries across large time windows. citeturn0search2

```bash
# Example: lookup CreateSnapshot calls in last 24h (CloudTrail Lake pseudo)
# In console: CloudTrail Lake -> Query -> SQL like:
# SELECT eventTime, eventName, sourceIPAddress, userIdentity.type, userIdentity.arn, requestParameters ...
# FROM <event_data_store>
# WHERE eventName IN ('CreateSnapshot','CopySnapshot','CreateImage','ExportImage')
#   AND eventTime BETWEEN '2025-09-24T00:00:00Z' AND '2025-09-25T00:00:00Z';
```

---

## 2) Low‑touch acquisition using **EBS Direct APIs**

> Goal: **hash and sample** snapshot contents without creating a volume or launching instances. _This complements the “Acquisition” section in your source guide【22†source】._

1. **List blocks** and **changed blocks** (between two snapshots in the same lineage).  
2. **Read blocks** to files; verify with checksums returned by the API.  
3. **Compute rolling or full hashes** client‑side for evidence logs.

```bash
# List blocks in a snapshot (returns BlockIndex + BlockToken; BlockSize typically 524288 bytes)
aws ebs list-snapshot-blocks --snapshot-id snap-0123456789abcdef0 --max-results 500

# List blocks changed between two lineage snapshots
aws ebs list-changed-blocks \
  --first-snapshot-id  snap-0aaa... \
  --second-snapshot-id snap-0bbb... \
  --starting-block-index 0 --max-results 500

# Get raw data for a given block (writes binary data to file)
aws ebs get-snapshot-block \
  --snapshot-id snap-0123456789abcdef0 \
  --block-index 6001 \
  --block-token AAAB... \
  /tmp/block-6001.bin
```
- **Notes:** EBS Direct APIs **do not work** on **archived** snapshots; use **Standard tier** for direct reads. Block size is 524,288 bytes (512 KiB). citeturn1view0

**Why this is forensically useful:** fast **comparison/diff** between two snapshot points; limited footprint; easy **parallel hashing** of blocks. See research/defense usage examples. citeturn0search17

---

## 3) Traditional acquisition paths (still valid)

### A. Clone & attach workflow
- **Copy snapshot** → **Create volume** (read‑only attach to a trusted forensic instance) → **image the device** to S3 (e.g., `dd` or `dcfldd`) → **hash**.  
- Keep originals immutable; document every API call (CloudTrail will record your actions). This aligns with your original acquisition guidance【22†source】.

```bash
# Copy snapshot cross‑Region/account (preservation)
aws ec2 copy-snapshot \
  --source-region ap-southeast-1 \
  --source-snapshot-id snap-0abcd... \
  --description "Forensic copy - Case 2025-09-25"

# Create volume from snapshot, then attach to forensic host
aws ec2 create-volume --availability-zone ap-southeast-1a --snapshot-id snap-0abcd...
aws ec2 attach-volume --volume-id vol-0aaaa... --instance-id i-0forensic...
# On the forensic host (Linux)
sudo dd if=/dev/xvdf of=/evidence/vol.raw bs=1M status=progress && sha256sum /evidence/vol.raw
```

### B. Export an **AMI** to a VM file for offline tools
- Create an **AMI** from the instance/snapshot (if not already), then **`export-image`** to S3 as **OVF/OVA/VMDK/VHD** for use in third‑party suites. citeturn0search1turn0search7turn0search13

```bash
aws ec2 export-image \
  --image-id ami-0abcd1234... \
  --disk-image-format VMDK \
  --s3-export-location S3Bucket=forensic-bucket,S3Prefix=exports/
```

---

## 4) Preservation controls (prevent tampering or loss)

- **Recycle Bin** for EBS Snapshots — set rules so “deleted” snapshots enter Recycle Bin and can be **recovered** during investigations. citeturn0search3turn0search9turn0search14
- **Archive Tier** for Snapshots — low‑cost long‑term storage (note: not readable via EBS Direct APIs; must restore first). citeturn0search10turn0search4turn0search20turn0search15
- **S3 Object Lock** for exported evidence (WORM); versioning and MFA‑delete (if using S3 evidence buckets). *(Use per your organization’s policy.)*
- **Cross‑account copies** to an evidence account with tight IAM and KMS controls (document **KMS key IDs** and key policies).

---

## 5) Timeline & correlation

- Build a **CloudTrail Lake** query set for: `CreateSnapshot`, `CopySnapshot`, `CreateImage`, `ExportImage`, `AttachVolume`, `RunInstances`, `StopInstances`. citeturn0search2
- Combine with: VPC Flow Logs, ELB/ALB logs, and guest OS logs (VSS/LVM), as covered in your source guide【22†source】.
- Record clock sources and timezones; convert to **UTC** in reports.

---

## 6) Quick scenarios (AWS-focused)

### A) Unknown snapshot appears
1. `describe-snapshots` + tags → capture IDs/KMS keys/owners.  
2. CloudTrail/CloudTrail Lake: filter on `CreateSnapshot` → extract principal & source IP. citeturn0search2  
3. **Preserve**: copy snapshot to evidence account; apply Recycle Bin rules. citeturn0search3  
4. **Acquire** via EBS Direct APIs or create a volume and image. citeturn1view0

### B) Suspected rollback / restore
1. Find `CreateImage`, `RunInstances`, `CreateVolume` from snapshot.  
2. Compare two snapshots via **`list-changed-blocks`** then fetch only changed blocks for triage. citeturn1view0  
3. Report file‑system diffs and timeline.

### C) Encrypted snapshots (KMS)
1. Record **KMS key ID** and snapshot encryption state during triage.  
2. Verify that your forensic role has **KMS decrypt** perms or coordinate with key custodians.  
3. When exporting AMIs, ensure target S3 bucket policy and KMS permissions allow writes/reads. citeturn0search13

---

## 7) Verification & hashing (examples)

```bash
# Hash exported raw image
sha256sum /evidence/vol.raw > /evidence/vol.raw.sha256

# Hash each block pulled via EBS Direct APIs (example loop; pseudo)
for i in $(seq 0 9999); do
  aws ebs get-snapshot-block --snapshot-id "$SNAP" --block-index "$i" --block-token "$(token_for $i)" "/tmp/$i.bin"
  sha256sum "/tmp/$i.bin" >> blocks.sha256
done
```

---

## 8) Reporting checklist (AWS flavor)
- [ ] Snapshot/AMI/Volume IDs, Regions, Accounts, **KMS key IDs** recorded.
- [ ] CloudTrail/CloudTrail Lake queries & exports saved to evidence. citeturn0search2
- [ ] Recycle Bin policy snapshot + current state captured. citeturn0search3
- [ ] Archive/restoration decisions documented (if moved to archive). citeturn0search10
- [ ] All artifacts hashed; hash manifests included.
- [ ] Cross‑account copy and S3 Object Lock settings logged.
- [ ] Limitations & assumptions (e.g., archived snapshots not readable via EBS Direct APIs). citeturn1view0

---

## 9) References
- **EBS Direct APIs (read snapshots, list & get blocks)** — docs & examples. citeturn1view0turn0search6turn0search12
- **Export AMI to VM file (`export-image`)** — CLI & API. citeturn0search1turn0search7turn0search13turn0search18
- **CloudTrail Lake** — overview & usage. citeturn0search2turn0search8
- **Recycle Bin for EBS Snapshots** — console & blog announcement. citeturn0search3turn0search9turn0search14
- **EBS Snapshot Archive & pricing** — docs & pricing page. citeturn0search10turn0search20turn0search4turn0search15
- **IAM Identity Center (formerly AWS SSO)** — rename & docs. citeturn0search5turn0search11

---

_This AWS-focused supplement is intended to be used alongside your broader “Snapshots & Artifacts” guide【22†source】._
