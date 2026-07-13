# redStack: Boot-To-Breach Red Team Platform (Prerequisites)

Everything you need in place before the workshop. All of it is one-time per AWS account, so do it ahead of the session; it should not eat into the workshop clock. When every box below is checked, you are ready for 1_DEPLOY.

Run the deploy from your host machine or a dedicated VM with the AWS CLI and Terraform installed, not from inside the lab itself. The Terraform output writes `deployment_info.txt` to `redStack/` and you will need it open on the same machine you are deploying from.

| At a glance |                                            |
| ----------- | ------------------------------------------ |
| When        | Before the session                         |
| Time        | ~15 to 30 min per AWS account              |
| Cost        | $0 to complete (compute starts in DEPLOY)  |
| Next        | 1_DEPLOY                                    |

## Checklist

- [ ] Dedicated, throwaway AWS account (payment method attached)
- [ ] AWS CLI installed and configured (IAM user with `AdministratorAccess`, region `us-east-1`)
- [ ] Terraform 1.0 or higher installed
- [ ] Kali Linux Marketplace EULA accepted on that account
- [ ] Hack Smarter Labs account with an active subscription covering ShadowGate, and your `.ovpn` downloaded
- [ ] redStack repo cloned
- [ ] SSH key pair `rs-rsa-key` created in AWS, `.pem` in the repo root
- [ ] Your public IPv4 recorded

---

## Step 1. Dedicated throwaway AWS account

Use a dedicated, throwaway AWS account, not your production account. redStack stands up public-facing hosts and the AWS Acceptable Use Policy applies.

Success: a dedicated AWS account with a payment method attached, not used for production workloads.

## Step 2. Install and configure the Git, Terraform, and AWS CLI

**Install Git:**

| Platform | Command |
| -------- | ------- |
| macOS | `brew install git` or install Xcode Command Line Tools: `xcode-select --install` |
| Linux (Ubuntu/Debian) | `sudo apt install git` |
| Windows | Download and run the installer from https://git-scm.com/download/win |

**Install Terraform:**

| Platform | Command |
| -------- | ------- |
| macOS | `brew install terraform` |
| Linux (Ubuntu/Debian) | See https://developer.hashicorp.com/terraform/install |
| Windows | `choco install terraform` or download from https://developer.hashicorp.com/terraform/install |

**Install AWS CLI:**

| Platform | Command |
| -------- | ------- |
| macOS | `brew install awscli` |
| Linux (Ubuntu/Debian) | `sudo apt install awscli` |
| Linux (any) | `curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && unzip awscliv2.zip && sudo ./aws/install` |
| Windows | Download and run the MSI from https://aws.amazon.com/cli/ |

**Create an IAM user and attach permissions.** Skip this if you are using the root account or already have credentials configured. For the least-privilege policy option, see [Step 2 on the Prerequisites wiki page](https://github.com/BaddKharma/redStack/wiki/02.-Prerequisites#-step-2-aws-iam-permissions).

1. IAM Console > Users > Create user
2. Username: `redS-operator`
3. Permissions > Attach policies directly > search `AdministratorAccess`
4. Check `AdministratorAccess` > Next > Create user
5. Open the new user > Security credentials > Create access key
6. Select Command Line Interface (CLI) > acknowledge > Next
7. Copy the Access Key ID and Secret Access Key (the secret is shown only once)

**Configure the AWS CLI** with the IAM user's access key:

```bash
aws configure
```

When prompted:

| Prompt | Value |
| ------ | ----- |
| AWS Access Key ID | From IAM > your user > Security credentials |
| AWS Secret Access Key | Shown once at key creation; if lost, delete and recreate |
| Default region name | `us-east-1` (must match `aws_region` in `terraform.tfvars`) |
| Default output format | `json` |

Success:

```bash
aws sts get-caller-identity
```

Returns your Account, ARN, and UserId.

```bash
terraform --version
```

Reports v1.0 or higher.

If it fails: `InvalidClientTokenId` or `SignatureDoesNotMatch` means the keys were mistyped; re-run `aws configure`. Terraform below 1.0, upgrade. The region here must match `aws_region` in `terraform.tfvars`.

## Step 3. Accept the Kali Marketplace EULA

One-time, no charge, per AWS account. Skip it and the apply fails on the Kali AMI with `OptInRequired`.

1. Visit https://aws.amazon.com/marketplace/pp/prodview-fznsw3f7mq7to signed in to the same account.
2. Click on `View Purchase Options` 
3. Scroll down then click on `Subscribe`, then Accept Terms.

Success: the Marketplace page shows the subscription active.

## Step 4. Hack Smarter Labs access and .ovpn

Have a Hack Smarter Labs account with an active subscription covering the target machine, and download your `.ovpn` connection pack.

ShadowGate: https://www.hacksmarter.org/courses/e7586073-d447-41db-8f8e-6bd22576556d

Success: you have a `.ovpn` file saved locally (keep it somewhere safe since you will need this later) and the ShadowGate machine launches in the HSL portal.

If it fails: the target IP shown at launch is inside the range and does not change your VPN config. The `.ovpn` pulls its routes from HSL at connect time.

## Step 5. Clone the repository

```bash
git clone https://github.com/BaddKharma/redStack.git
cd redStack
```

Success: you are inside `redStack/` and see `terraform/` with `terraform.tfvars.example` in it.

If it fails: no `git`, install it. Corporate proxy blocking GitHub, clone over SSH: `git clone git@github.com:BaddKharma/redStack.git`.

## Step 6. Create the SSH key pair (right after clone, lands in `redStack/`)

Create the key from inside `redStack/` so the `.pem` lands in the repo root next to `terraform/`. This is the file Terraform uses to decrypt the Windows password, and keeping it here is what makes `ssh_private_key_path = "../rs-rsa-key.pem"` resolve correctly.

Windows (PowerShell):

```powershell
aws ec2 create-key-pair --key-name rs-rsa-key --query 'KeyMaterial' --output text | Out-File -Encoding ascii rs-rsa-key.pem
```

Linux / macOS:

```bash
aws ec2 create-key-pair --key-name rs-rsa-key \
  --query 'KeyMaterial' --output text > ./rs-rsa-key.pem
chmod 400 ./rs-rsa-key.pem
```

Success (the key is present in `redStack/`, mode 400):

```bash
ls -l rs-rsa-key.pem
aws ec2 describe-key-pairs --key-names rs-rsa-key
```

If it fails: `InvalidKeyPair.Duplicate` means `rs-rsa-key` already exists in AWS. Three options:

- **Reuse**: if you still have the original `.pem`, copy it to `redStack/rs-rsa-key.pem` and skip the create command.
- **Import**: if you have a local SSH key pair you want to use instead, import its public key: `aws ec2 import-key-pair --key-name rs-rsa-key --public-key-material fileb://~/.ssh/id_rsa.pub` and place the matching private key at `redStack/rs-rsa-key.pem`.
- **Delete and recreate**: `aws ec2 delete-key-pair --key-name rs-rsa-key`, then re-run the create command above.

Confirm the file lands at `redStack/rs-rsa-key.pem`, not inside `terraform/`. It is already covered by `.gitignore`.

## Step 7. Record your public IP

```bash
curl -4 -s ifconfig.me
```

Success: a single IPv4 address. You append `/32` to it in 1_DEPLOY.

If it fails: output with colons is IPv6, re-run with `-4`. The security groups only wire up IPv4.

---

## Next

Prerequisites done. Continue with 1_DEPLOY to configure `terraform.tfvars` and stand up the lab.
