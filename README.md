# Hyper-V CLI Cheat Sheet (Windows)

## 1. PowerShell Module: Hyper-V

### List VMs

```powershell
Get-VM
```

### Start / Stop VM

```powershell
Start-VM -Name "VMName"
Stop-VM -Name "VMName"
```

### Check VM State

```powershell
Get-VM -Name "VMName" | Select-Object Name, State
```

### Manage Checkpoints

```powershell
# List checkpoints
Get-VMSnapshot -VMName "VMName"

# Create checkpoint
Checkpoint-VM -Name "VMName" -SnapshotName "MyCheckpoint"

# Restore checkpoint
Restore-VMSnapshot -VMName "VMName" -Name "MyCheckpoint"
```

### Networking

```powershell
# List virtual switches
Get-VMSwitch

# Create a new external switch
New-VMSwitch -Name "ExternalSwitch" -NetAdapterName "Ethernet" -AllowManagementOS $true

# Connect VM to switch
Connect-VMNetworkAdapter -VMName "VMName" -SwitchName "ExternalSwitch"
```

### Virtual Hard Disks (VHD/VHDX)

```powershell
# Create new VHD
New-VHD -Path "C:\VMs\VMName\Disk.vhdx" -SizeBytes 50GB -Dynamic

# Attach VHD
Add-VMHardDiskDrive -VMName "VMName" -Path "C:\VMs\VMName\Disk.vhdx"

# Resize VHD
Resize-VHD -Path "C:\VMs\VMName\Disk.vhdx" -SizeBytes 100GB
```

## 2. CLI Utilities

### `vmconnect.exe` - Connect to a VM session

```cmd
vmconnect.exe localhost "VMName"
```

### `diskpart` - Manage virtual disks

```cmd
diskpart
DISKPART> select vdisk file="C:\VMs\VMName\VirtualDisk.vhdx"
DISKPART> attach vdisk
```

## 3. Quick Tips

* Always run PowerShell **as Administrator** when managing Hyper-V.
* Use `Get-Help <cmdlet>` to see more options.
* Combine with scripts to automate VM deployment and snapshots.
1. For windows

Download and install
<https://awscli.amazonaws.com/AWSCLIV2.msi>

2. For Ubuntu/Debian in WSL

# Update package index

sudo apt update

# Install prerequisites

sudo apt install curl unzip -y

# Download AWS CLI v2 installer

curl "<https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip>" -o "awscliv2.zip"

# Unzip the installer

unzip awscliv2.zip

# Run the installer

sudo ./aws/install

# Verify installation

aws --version

3. SSO configuration for aws CLIs

# Setup Browser(Otional if use WSL)

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb

sudo apt install ./google-chrome-stable_current_amd64.deb

# Run chrome
google-chrome

```

```bash
aws configure sso
```

# Name (set any name)

SSO session name (Recommended): <your name>

# SSO start URL check from your AWS IAM account (IAM Identity Center)

SSO start URL [None]: <https://mycompany.awsapps.com/start>

# SSO region (check your VMs region for fast setup), ex: us-east-1

SSO region [None]: <check your default region>

# SSO registration scopes (default [sso:account:access] )

SSO registration scopes [sso:account:access]

# SSO login

Redirect to your browser

# Set Default client Region: Check your VMs Region

Default client Region [None]: <your select VMs Region>

# Set output format

CLI default output format (json if not specified) [None]: json

# Set Profile name

# Verify your connection

aws configure list-profiles
aws --no-cli-pager sts get-caller-identity --profile <your profile>
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
