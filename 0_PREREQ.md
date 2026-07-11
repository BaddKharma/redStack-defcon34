# redStack: Boot-To-Breach Red Team Platform (Prerequisites)

Everything you need in place before the workshop. All of it is one-time per AWS account, so do it ahead of the session; it should not eat into the workshop clock. When every box below is checked, you are ready for 1_DEPLOY.

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

## Step 2. Install and configure the AWS CLI and Terraform

Install both, then configure AWS credentials for an IAM user with `AdministratorAccess` on the throwaway account (`aws configure`, region `us-east-1`, output `json`).

Success:

```bash
aws sts get-caller-identity
```

Returns your Account, Arn, and UserId.

```bash
terraform --version
```

Reports v1.0 or higher.

If it fails: `InvalidClientTokenId` or `SignatureDoesNotMatch` means the keys were mistyped, re-run `aws configure`. Terraform below 1.0, upgrade. The region here must match `aws_region` in `terraform.tfvars`.

## Step 3. Accept the Kali Marketplace EULA

One-time, no charge, per AWS account. Skip it and the apply fails on the Kali AMI with `OptInRequired`.

1. Visit https://aws.amazon.com/marketplace/pp/prodview-fznsw3f7mq7to signed in to the same account.
2. Continue to Subscribe, then Accept Terms.

Success: the Marketplace page shows the subscription active.

If it fails: if apply later errors `OptInRequired`, finish the subscription and re-run apply. No state cleanup needed.

## Step 4. Hack Smarter Labs access and .ovpn

Have a Hack Smarter Labs account with an active subscription covering the target machine, and download your `.ovpn` connection pack.

ShadowGate: https://www.hacksmarter.org/courses/e7586073-d447-41db-8f8e-6bd22576556d

Success: you have a `.ovpn` file saved locally and the ShadowGate machine launches in the HSL portal.

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

If it fails: `InvalidKeyPair.Duplicate` means the key name already exists in AWS, either reuse the existing `.pem` or delete the AWS key (`aws ec2 delete-key-pair --key-name rs-rsa-key`) and recreate. Confirm the file is `redStack/rs-rsa-key.pem`, not inside `terraform/`. It is already covered by `.gitignore`.

## Step 7. Record your public IP

```bash
curl -4 -s ifconfig.me
```

Success: a single IPv4 address. You append `/32` to it in 1_DEPLOY.

If it fails: output with colons is IPv6, re-run with `-4`. The security groups only wire up IPv4.

---

## Next

Prerequisites done. Continue with 1_DEPLOY to configure `terraform.tfvars` and stand up the lab.
