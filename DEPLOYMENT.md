# redStack: Boot-To-Breach Red Team Platform — Deployment Guide (Tunneled Access)

Step-by-step deploy of the workshop lab: from a clean AWS account to a running redStack with the OpenVPN tunnel up and the ShadowGate target subnet reachable. This is the setup runbook only. C2 stand-up and the attack path are covered in separate materials.

Deployment mode is Tunneled Access (OpenVPN to Hack Smarter Labs), not Direct Access. The redStack wiki stays the source of truth if any fact here drifts. Placeholders in `<angle brackets>` come from your own deploy; lab IPs, passwords, the Windows Administrator password, and the `X-Request-ID` token all print from `terraform output deployment_info` after apply.

Work top to bottom. Every step lists the command, what success looks like, and what to try if it fails.

| At a glance | |
|--|--|
| Workshop | DefCon, redStack boot-to-breach lab |
| Mode | Tunneled Access (OpenVPN + WireGuard to Hack Smarter Labs) |
| Target | ShadowGate (Hack Smarter Labs) |
| Pre-work time | ~15 to 30 min per AWS account (Phase 1) |
| Deploy time | ~5 to 10 min apply, plus ~8 to 12 min for the Windows password, plus ~5 min cloud-init |
| Cost | ~$0.21 to $0.25/hr compute. Destroy when done |
| Output | `deployment_info.txt` in `redStack/` with every IP, password, and the C2 header token |

---

## Directory model (read this first)

Two locations matter, and mixing them up is the most common cause of a broken deploy.

`redStack/` (repo root) holds only the SSH private key, `rs-rsa-key.pem`. `redStack/terraform/` holds the `.tf` files, `terraform.tfvars`, and the state files, and every `terraform` command runs from there.

Because Terraform runs from `redStack/terraform/` but the key lives one level up in the root, the key path in `terraform.tfvars` is relative to the `terraform/` directory:

```hcl
ssh_private_key_path = "../rs-rsa-key.pem"
```

Use `../rs-rsa-key.pem`, not `./rs-rsa-key.pem`. The `./` form resolves to `redStack/terraform/rs-rsa-key.pem`, which does not exist, and silently breaks the Windows Administrator password decrypt (it shows as "(not yet available)").

---

## Phase 1: Pre-work (do this before the workshop)

Everything in Phase 1 is one-time per AWS account and should not eat into the session clock.

### Step 1. Dedicated throwaway AWS account

Use a dedicated, throwaway AWS account, not your production account. redStack stands up public-facing hosts and the AWS Acceptable Use Policy applies.

If it fails: account signup needs a payment method even for lab use.

### Step 2. Install and configure the AWS CLI and Terraform

Install both, then configure AWS credentials for an IAM user with `AdministratorAccess` on the throwaway account (`aws configure`, region `us-east-1`, output `json`).

Success:

```bash
aws sts get-caller-identity   # returns your Account, Arn, and UserId
terraform --version           # v1.0 or higher
```

If it fails: `InvalidClientTokenId` or `SignatureDoesNotMatch` means the keys were mistyped, re-run `aws configure`. Terraform below 1.0, upgrade. The region here must match `aws_region` in `terraform.tfvars`.

### Step 3. Accept the Kali Marketplace EULA

One-time, no charge, per AWS account. Skip it and the apply fails on the Kali AMI with `OptInRequired`.

1. Visit https://aws.amazon.com/marketplace/pp/prodview-fznsw3f7mq7to signed in to the same account.
2. Continue to Subscribe, then Accept Terms.

Success: the Marketplace page shows the subscription active.

If it fails: if apply later errors `OptInRequired`, finish the subscription and re-run apply. No state cleanup needed.

### Step 4. Hack Smarter Labs access and .ovpn

Have a Hack Smarter Labs account with an active subscription covering the target machine, and download your `.ovpn` connection pack.

ShadowGate: https://www.hacksmarter.org/courses/e7586073-d447-41db-8f8e-6bd22576556d

Success: you have a `.ovpn` file saved locally and the ShadowGate machine launches in the HSL portal.

If it fails: the target IP shown at launch is inside the range and does not change your VPN config. The `.ovpn` pulls its routes from HSL at connect time.

### Step 5. Clone the repository

```bash
git clone https://github.com/BaddKharma/redStack.git
cd redStack
```

Success: you are inside `redStack/` and see `terraform/` with `terraform.tfvars.example` in it.

If it fails: no `git`, install it. Corporate proxy blocking GitHub, clone over SSH: `git clone git@github.com:BaddKharma/redStack.git`.

### Step 6. Create the SSH key pair (right after clone, lands in `redStack/`)

Create the key from inside `redStack/` so the `.pem` lands in the repo root next to `terraform/`. This is the file Terraform uses to decrypt the Windows password, and keeping it here is what makes `ssh_private_key_path = "../rs-rsa-key.pem"` resolve correctly.

Linux / macOS:

```bash
aws ec2 create-key-pair --key-name rs-rsa-key \
  --query 'KeyMaterial' --output text > ./rs-rsa-key.pem
chmod 400 ./rs-rsa-key.pem
```

Windows (PowerShell):

```powershell
aws ec2 create-key-pair --key-name rs-rsa-key --query 'KeyMaterial' --output text | Out-File -Encoding ascii rs-rsa-key.pem
icacls "rs-rsa-key.pem" /inheritance:r /grant:r "$($env:USERNAME):R"
```

Success:

```bash
ls -l rs-rsa-key.pem                          # present in redStack/, mode 400
aws ec2 describe-key-pairs --key-names rs-rsa-key
```

If it fails: `InvalidKeyPair.Duplicate` means the key name already exists in AWS, either reuse the existing `.pem` or delete the AWS key (`aws ec2 delete-key-pair --key-name rs-rsa-key`) and recreate. Confirm the file is `redStack/rs-rsa-key.pem`, not inside `terraform/`. It is already covered by `.gitignore`.

### Step 7. Record your public IP

```bash
curl -4 -s ifconfig.me
```

Success: a single IPv4 address. You append `/32` to it in Phase 2.

If it fails: output with colons is IPv6, re-run with `-4`. The security groups only wire up IPv4.

---

## Phase 2: Configure and deploy

### Step 8. Configure terraform.tfvars

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
notepad terraform.tfvars      # or: nano terraform.tfvars if on Linux
```

Set the three values personal to your deploy:

```hcl
localPub_ip          = "<YOUR_IPV4>/32"       # from Step 7, keep the /32
ssh_key_name         = "rs-rsa-key"           # matches the AWS key pair
ssh_private_key_path = "../rs-rsa-key.pem"    # key is in the repo root, not here
```

Set the Tunneled Access block:

```hcl
redirector_domain                    = ""     # no domain in tunneled mode
enable_vpn_tunnel                    = true   # OpenVPN client + WireGuard routing
enable_redirector_htaccess_filtering = false  # scanner/AV blocking is pointless inside an isolated lab
```

Tunnel CIDRs. Route only the specific target subnet you need, never a supernet that contains the lab VPCs (10.50.0.0/16, 10.60.0.0/16). Guacamole's WireGuard `AllowedIPs` is taken directly from this list and its tunnel endpoint is the redirector's 10.60.x IP, so a broad range like 10.0.0.0/8 swallows that endpoint and deadlocks the tunnel. ShadowGate sits on 10.1.x:

```hcl
vpn_tunnel_cidrs = ["10.1.0.0/16"]   # add other HSL /16s as needed; never 10.50/16 or 10.60/16
```

Leave Sliver on `t3.medium`. The Go cross-compiler exhausts a `t3.small` during implant generation and hangs SSH.

Success: `terraform.tfvars` saved with your IP, `ssh_private_key_path = "../rs-rsa-key.pem"`, the three tunnel values, and the RFC1918 CIDRs.

If it fails: nothing runs yet, but recheck the key path now. It is the single most common misconfiguration.

### Step 9. Init and plan

```bash
terraform init
terraform plan
```

Success: `init` reports success, and `plan` shows roughly 90 resources to add (7 EC2 instances, 2 VPCs, peering, security groups, ENIs, 2 EIPs), no errors.

If it fails: `InvalidKeyPair.NotFound`, `ssh_key_name` does not match AWS, list with `aws ec2 describe-key-pairs --query 'KeyPairs[].KeyName'`. `VPCLimitExceeded`, you are at the 5-VPC regional limit, delete unused VPCs or set `use_default_vpc = true`.

### Step 10. Apply

```bash
terraform apply
```

Type `yes`. Apply runs about 5 to 10 minutes. Expect the Windows instance to sit in "still creating" for roughly 8 to 12 minutes while it waits for the AWS-generated Administrator password. That wait is intentional (create now blocks until the password is decryptable), not a hang.

Cloud-init keeps running after apply returns. Linux hosts and Guacamole come up shortly after; the Windows workstation and the Mythic UI need roughly another 10 minutes.

If you plan to use Havoc this session, kick off its build now so it compiles in the background (~1 GB download, ~9 min). SSH to the Havoc host and run:

```bash
~/build_havoc.sh
```

Success: `Apply complete! Resources: NN added, 0 changed, 0 destroyed.`

If it fails: `OptInRequired`, finish the Kali subscription (Step 3) and re-run, no cleanup needed. If it stops on the Windows postcondition ("Windows password_data was not available before timeouts.create expired"), the AMI ran long, re-run `terraform apply` and it picks up the now-available password.

### Step 11. Capture deployment info

```bash
terraform output deployment_info
```

This holds the lab password, the Windows Administrator password, the redirector Elastic IP, and the `X-Request-ID` token. `deployment_info.txt` is also written to `redStack/`. Keep it open for the session.

Success: every host block is populated, and the Windows block shows a real password rather than "(not yet available)."

If it fails: if the Windows password still reads "(not yet available)," run `terraform refresh && terraform output deployment_info`. Still blank, confirm `ssh_private_key_path = "../rs-rsa-key.pem"` and that `redStack/rs-rsa-key.pem` exists (see the Directory model section).

---

## Phase 3: Verify the lab

### Step 12. Access the Guacamole portal

Open `https://<GUAC_PUBLIC_IP>/guacamole`, accept the self-signed cert warning, and log in as `guacadmin` with the lab password from `deployment_info`. The cert encrypts the operator session only and is not part of the C2 path.

Success: the connection list shows Windows (RDP), SSH entries for Mythic, Sliver, Havoc, Redirector, Guacamole, and Kali, plus Havoc Desktop (VNC).

If it fails: give cloud-init the full 5 minutes. If the portal never loads, SSH to Guacamole and check `docker ps` for the `guacamole`, `postgres`, and `guacd` containers.

### Step 13. Access the Windows operator

In Guacamole, click Windows (RDP). Give it 10 to 30 seconds.

Success: the desktop loads with Chromium, VS Code, MobaXterm (redStack Lab session folder), and 7-Zip present.

If it fails: wait five more minutes; Windows is the slowest host and the decrypted Administrator password is applied late in cloud-init. If RDP rejects the password, the credential baked into Guacamole is stale (the password was not ready when the portal built its connection), fix the key path per the Directory model section and redeploy, or edit the Windows (RDP) connection in Guacamole and paste the password from `deployment_info`.

### Step 14. Confirm cross-host name resolution

From a Windows PowerShell prompt:

```powershell
ping mythic
ping sliver
ping havoc
ping redirector
ping guac
ping kali
```

Success: all hostnames resolve and respond.

If it fails: see the redStack wiki Troubleshooting page (Connectivity Checks). The tunnel and attack path assume cross-host DNS works.

---

## Phase 4: Bring up the tunnel and confirm reachability

The tunnel is a per-session manual start. Do it after the lab verifies. WireGuard between Guacamole and the redirector is already configured automatically during cloud-init.

> **Operational warning: do not rebuild Guacamole during the workshop.** A guac rebuild generates fresh WireGuard keys and pushes a new config to the redirector, but `guacamole_setup.sh` runs `systemctl start wg-quick@wg0` (not `restart`). Since wg0 is already up from the first deploy, `start` is a no-op and the redirector keeps the old keys. The handshake then fails and every internal host loses reachability to the target, while the redirector itself still reaches it, so it looks like a routing problem, not a key mismatch. Deploy and validate the lab before doors open and leave guac alone. If a rebuild is ever unavoidable, run `sudo systemctl restart wg-quick@wg0` on the redirector afterward to resync the tunnel.

### Step 15. Get your .ovpn onto the redirector

You already have your ShadowGate `.ovpn` on your laptop from pre-work (Step 4). Move it to the redirector in two hops: laptop to the Windows operator via GuacShare, then operator to the redirector via MobaXterm.

1. Get the `.ovpn` onto the Windows operator. In the Guacamole Windows (RDP) session, open the sidebar (`Ctrl+Alt+Shift`), click Devices, and upload your `.ovpn`. It lands in the `GuacShare` folder, visible in This PC in File Explorer.
2. In MobaXterm on the operator, open the **Apache Redirector (SSH)** bookmark under redStack Sessions. The left pane is the redirector's SFTP browser; it opens in `/home/admin`, so double-click into `vpn`.
3. With the `vpn` folder open in the SFTP pane, click the "Upload to current folder" button in the SFTP toolbar and pick the `.ovpn` from `GuacShare`.

Success: in the same MobaXterm terminal, `ls ~/vpn/` shows exactly one `.ovpn` file.

If it fails: the `vpn-tunnel` service picks the first file alphabetically if several exist, so keep only one in `~/vpn/`.

### Step 16. Start the tunnel

```bash
sudo systemctl start vpn-tunnel
journalctl -u vpn-tunnel -f
```

Wait for `Initialization Sequence Completed`.

Success: the service is active and the log shows the completed handshake.

If it fails: the service will not start if `~/vpn/` is empty (Step 15). Watch the live log for auth or route errors.

### Step 17. Record the tun0 IP

```bash
ip -4 addr show tun0 | grep -oP '(?<=inet\s)\d+(\.\d+)+'
```

This IP is dynamic and changes on every reconnect. If you generate C2 agents, do it after the tunnel is up, not before.

Callback address gotcha (for the attack path, not this guide): against Hack Smarter Labs, C2 implants call back to the redirector public Elastic IP, not the `tun0` IP. HSL does not route VPN client IPs back to the target, so a `tun0` callback never returns. Full callback config lives in the attack-path materials.

### Step 18. Confirm reachability to the target

From the Windows workstation (no nmap on Windows, so use a TCP probe):

```powershell
ping <TARGET_IP>
Test-NetConnection <TARGET_IP> -Port 445
```

ShadowGate answers ICMP, so a ping reply confirms the path; `TcpTestSucceeded : True` on 445 confirms the service is reachable.

Success: a reply from `<TARGET_IP>` confirms the full path is live: internal host to Guacamole (WireGuard) to redirector (OpenVPN) to the HSL target network. The lab is ready for the attack path.

If it fails, isolate the break from the redirector (SSH in over Guacamole). Work these three checks in order:

```bash
ip route                                              # expect: 10.1.0.0/16 (your target CIDR) via ... dev tun0
ping -c3 <TARGET_IP>                                  # redirector to target, over OpenVPN
sudo wg show                                          # Guacamole peer: recent handshake, nonzero transfer
```

Reading the result:

- No `tun0` route to the target CIDR, or ping fails from the redirector: the OpenVPN tunnel or HSL routing is the problem, not the internal path. Recheck Steps 16 and 17, and give the target the HSL boot window (about 5 minutes) before assuming a routing fault.
- Redirector reaches the target but the Windows operator does not: the guac-to-redirector WireGuard hop is down. Check `wg show` for a stale or missing handshake and run `sudo systemctl restart wg-quick@wg0`, then re-test from Windows. This is the Guacamole-rebuild key-mismatch failure noted in Phase 4.

---

## Phase 5: Teardown and cost

redStack deployments in the workshop are short-lived. Tear down when done, and run destroy from `redStack/terraform/`, the same directory you applied from:

```bash
cd redStack/terraform
terraform destroy
```

Type `yes`. It releases the Elastic IPs, terminates instances, and removes the VPCs (~5 min). Stopping instances instead of destroying leaves EBS volumes and Elastic IPs billing 24/7, so `destroy` is what actually stops all charges.

Confirm it worked. A real teardown ends with `Destroy complete! Resources: NN destroyed.` where NN is nonzero, and `terraform state list` then comes back empty:

```bash
terraform state list
```

Run from the wrong directory and Terraform finds no state, so it reports `No objects need to be destroyed` and `0 destroyed` while your infra keeps running and billing. `0 destroyed` is not a successful teardown. If you see it, `cd` into `redStack/terraform/` and run destroy again.

The tunnel does not auto-start after a stop or reboot. Re-run `sudo systemctl start vpn-tunnel` on the redirector when you resume. WireGuard comes back automatically.

---

## Where this hands off

At the end of Phase 4 the lab is deployed, verified, tunneled, and ShadowGate is reachable. From here the workshop moves into C2 stand-up (Sliver, Mythic, Havoc) and the ShadowGate attack path, covered in their own materials.

---

## Known issues and fixes (living log)

- Key path: the wiki Deploy page shows `ssh_private_key_path = "./rs-rsa-key.pem"`. From `redStack/terraform/` that resolves to a nonexistent file and breaks the Windows password decrypt. Correct value is `../rs-rsa-key.pem` (matches `terraform.tfvars.example`). Backport the fix to wiki page `04.-Deploying-Terraform.md`.
- Windows password timeout: `aws_instance.windows` now has `timeouts { create = "30m" }` and a `postcondition` on `password_data`, so apply blocks until the password is available and fails loudly instead of handing a blank credential to Guacamole. Expect the longer Windows