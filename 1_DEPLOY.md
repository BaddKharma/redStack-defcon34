# redStack: Boot-To-Breach Red Team Platform (Deployment Guide, Tunneled Access)

Deploy the workshop lab from a clean AWS account to a running redStack: tunnel up, ShadowGate reachable. Setup only; C2 stand-up is in CONFIG, the attack path in ATTACK. Mode is Tunneled Access (OpenVPN to Hack Smarter Labs).

Before you start, complete everything in 0_PREREQ (AWS account, AWS CLI and Terraform, Kali Marketplace EULA, Hack Smarter Labs `.ovpn`, repo clone, SSH key pair, your public IP). This guide picks up from there.

Placeholders in `<angle brackets>` come from your own deploy and print from `terraform output deployment_info` after apply. The [redStack wiki](https://github.com/BaddKharma/redStack/wiki) is the source of truth if anything here drifts.

Work top to bottom. Every step lists the command, what success looks like, and what to try if it fails.

| At a glance   |                                                                                        |
| ------------- | -------------------------------------------------------------------------------------- |
| Workshop      | DefCon, redStack boot-to-breach lab                                                     |
| Mode          | Tunneled Access (OpenVPN + WireGuard to Hack Smarter Labs)                              |
| Target        | ShadowGate (Hack Smarter Labs)                                                          |
| Prerequisites | Complete 0_PREREQ first (~15 to 30 min per AWS account)                                 |
| Deploy time   | ~5 to 10 min apply, plus ~8 to 12 min for the Windows password, plus ~5 min cloud-init  |
| Cost          | ~$0.27/hr compute. Destroy when done                                                    |
| Output        | `deployment_info.txt` in `redStack/` with every IP, password, and the C2 header token  |

---

## Directory model (read this first)

> [!IMPORTANT]
> Terraform runs from `redStack/terraform/`, but `rs-rsa-key.pem` (used to decrypt the Windows password) sits one level up in the repo root. `terraform.tfvars` must set `ssh_private_key_path = "../rs-rsa-key.pem"`, or the decrypt silently fails and the Windows password shows "(not yet available)".

Full directory layout is in the [redStack wiki](https://github.com/BaddKharma/redStack/wiki).

---

## Phase 1: Configure and deploy

### Step 1. Configure terraform.tfvars

On Linux, swap `notepad` for `nano`.

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
notepad terraform.tfvars
```

Set the three values personal to your deploy:

```hcl
localPub_ip          = "<YOUR_IPV4>/32"       # from 0_PREREQ, keep the /32
ssh_key_name         = "rs-rsa-key"           # matches the AWS key pair
ssh_private_key_path = "../rs-rsa-key.pem"    # key is in the repo root, not here
```

Set the Tunneled Access block:

```hcl
redirector_domain                    = ""     # no domain in tunneled mode
enable_vpn_tunnel                    = true   # OpenVPN client + WireGuard routing
enable_redirector_htaccess_filtering = false  # scanner/AV blocking has no effect inside an isolated lab
```

Tunnel CIDRs. Route only the target subnet you need. Power ShadowGate on in the HSL portal first, read its IP, and set `vpn_tunnel_cidrs` to the matching /16: the first two octets of the IP, then `.0.0/16`.

| ShadowGate IP | `vpn_tunnel_cidrs` |
| ------------- | ------------------ |
| `10.0.28.224` | `["10.0.0.0/16"]`  |
| `10.1.5.40`   | `["10.1.0.0/16"]`  |

```hcl
vpn_tunnel_cidrs = ["X.X.0.0/16"]   # first two octets of ShadowGate IP + .0.0/16
```

> [!WARNING]
> Never route a supernet that contains the lab VPCs (`10.50.0.0/16`, `10.60.0.0/16`). Guacamole's WireGuard `AllowedIPs` comes straight from this list and its tunnel endpoint is the redirector's `10.60.x` IP, so a broad range like `10.0.0.0/8` swallows that endpoint and deadlocks the tunnel. Do not carry a hardcoded value over from a prior run.

> [!IMPORTANT]
> A ShadowGate reset or reboot in the HSL portal can reassign its IP, so re-check after any reset. If the new IP lands in a different /16, `terraform destroy` and redeploy with the updated CIDR: the WireGuard routing is baked into Guacamole at boot (guac ignores `user_data` changes), so a plain `terraform apply` will not re-route a new subnet.

**Success:** `terraform.tfvars` saved with your IP, `ssh_private_key_path = "../rs-rsa-key.pem"`, the three tunnel values, and the RFC1918 CIDRs.

**Failure:** nothing runs yet, but recheck the key path now. It is the single most common misconfiguration.

### Step 2. Init and plan

```bash
terraform init
terraform plan
```

**Success:** `init` reports success, and `plan` shows roughly 90 resources to add (7 EC2 instances, 2 VPCs, peering, security groups, ENIs, 2 EIPs), no errors.

> [!NOTE]
> `terraform plan` may warn: "You didn't use the -out option to save this plan..." This is expected and harmless. `-out` saves the plan to a file so `apply` executes exactly that plan rather than re-evaluating. For this workshop the gap between `plan` and `apply` is seconds, so it is not necessary.

**Failure:** `InvalidKeyPair.NotFound`, `ssh_key_name` does not match AWS, list with `aws ec2 describe-key-pairs --query 'KeyPairs[].KeyName'`. `VPCLimitExceeded`, you are at the 5-VPC regional limit, delete unused VPCs or set `use_default_vpc = true`.

### Step 3. Apply

```bash
terraform apply
```

Type `yes`. Apply runs about 5 to 10 minutes.

> [!NOTE]
> The Windows instance sits in "still creating" for roughly 8 to 12 minutes while it waits for the AWS-generated Administrator password. That wait is intentional (create blocks until the password is decryptable), not a hang.

Cloud-init keeps running after apply returns. Linux hosts and Guacamole come up shortly after; the Windows workstation and the Mythic UI need roughly another 10 minutes.

**Success:** `Apply complete! Resources: NN added, 0 changed, 0 destroyed.`

**Failure:** `OptInRequired`, finish the Kali subscription (0_PREREQ Step 3) and re-run, no cleanup needed. If it stops on the Windows postcondition ("Windows password_data was not available before timeouts.create expired"), the AMI ran long, re-run `terraform apply` and it picks up the now-available password.

### Step 4. Capture deployment info

```bash
terraform output deployment_info
```

This holds the lab password, the Windows Administrator password, the redirector Elastic IP, and the `X-Request-ID` token. `deployment_info.txt` is also written to `redStack/`. Keep it open for the session as you will need to refer to it throughout the workshop.

**Success:** every host block is populated, and the Windows block shows a real password rather than "(not yet available)."

**Failure:** if the Windows password still reads "(not yet available)," run `terraform refresh && terraform output deployment_info`. Still blank, confirm `ssh_private_key_path = "../rs-rsa-key.pem"` and that `redStack/rs-rsa-key.pem` exists (see the Directory model section).

---

## Phase 2: Verify the lab

### Step 5. Access the Guacamole portal

Open `https://<GUAC_PUBLIC_IP>/guacamole`, accept the self-signed cert warning, and log in as `guacadmin` with the lab password from `deployment_info`. The cert encrypts the operator session only and is not part of the C2 path.

**Success:** the connection list shows Windows (RDP), SSH entries for Mythic, Sliver, Adaptix, Redirector, Guacamole, and Kali.

**Failure:** give cloud-init the full 5 minutes. If the portal never loads, SSH to Guacamole and check `docker ps` for the `guacamole`, `postgres`, and `guacd` containers.

### Step 5a. Let the Adaptix teamserver finish building

The Adaptix teamserver compiles from source automatically during cloud-init (server plus beacon extenders, ~12 min on the default instance). It runs headless: there is no VNC desktop and nothing to kick off by hand, and it starts itself when the build finishes. Because it is the slowest backend to come up, just let it build in the background while you work Steps 6 through 10, then confirm it in CONFIG Phase C.

To watch it, SSH to the Adaptix host with the MobaXterm **Adaptix C2 (SSH)** bookmark and run:

```bash
cloud-init status --wait; systemctl status adaptix --no-pager
```

`active (running)` with port 4321 listening means the teamserver is ready for the operator client. If the build failed, run `~/build_adaptix_server.sh` once to retry.

### Step 6. Access the Windows operator

In Guacamole, click Windows (RDP). Give it 10 to 30 seconds.

**Success:** the desktop loads with Chromium, VS Code, MobaXterm (redStack Lab session folder), 7-Zip, and the Adaptix client present. Provisioning is still running while a `_SETUP-IN-PROGRESS.txt` file sits on the desktop; the box is ready when that is replaced by `_SETUP-COMPLETE.txt`. Open the `Setup Log` desktop shortcut to watch the live provisioning log finish.

If MobaXterm opens without the redStack Lab session folder, provisioning had not finished when it launched. Close MobaXterm, wait for `_SETUP-COMPLETE.txt` on the desktop, then reopen it and the session folder will be there.

> [!TIP]
> Use the MobaXterm **redStack Sessions** bookmarks for all SSH into the lab hosts. Guacamole's per-host SSH is a last resort, only if the Windows operator box is down or MobaXterm cannot reach the host.

**Failure:** wait five more minutes; Windows is the slowest host and the decrypted Administrator password is applied late in cloud-init. If RDP rejects the password (or `deployment_info` shows the Windows password as `(not yet available)`), the pem terraform used to decrypt does not match the key pair the instance launched with, so it could not decrypt the password and Guacamole baked a blank one. `deployment_info` will not have the password in this case, so pull it straight from AWS with the correct pem, then paste it into the Windows (RDP) connection in Guacamole:

```powershell
$winId = aws ec2 describe-instances --filters "Name=tag:Hostname,Values=windows" "Name=instance-state-name,Values=running" --query "Reservations[].Instances[].InstanceId" --output text
aws ec2 get-password-data --instance-id $winId --priv-launch-key ../rs-rsa-key.pem
```

In Guacamole settings, edit the Windows (RDP) connection and set its password to that value. To also fix the terraform output, correct `ssh_private_key_path` to the matching pem and run `terraform apply -refresh-only`; if the pem itself is wrong (its fingerprint does not match `ssh_key_name` in AWS), redeploy per the Directory model section.

### Step 7. Optional: Confirm cross-host name resolution

From a Windows PowerShell prompt:

```powershell
ping mythic
ping sliver
ping adaptix
ping redirector
ping guac
ping kali
```

**Success:** all hostnames resolve and respond.

**Failure:** see the redStack wiki Troubleshooting page (Connectivity Checks). The tunnel and attack path assume cross-host DNS works.

---

## Phase 3: Bring up the tunnel and confirm reachability

The tunnel is a per-session manual start. Do it after the lab verifies. WireGuard between Guacamole and the redirector is already configured automatically during cloud-init.

### Step 8. Get your .ovpn onto the redirector

You already have your ShadowGate `.ovpn` on your laptop from 0_PREREQ Step 4. Move it to the redirector in two hops: laptop to the Windows operator via GuacShare, then operator to the redirector via MobaXterm.

1. Get the `.ovpn` onto the Windows operator. In the Guacamole Windows (RDP) session, open the sidebar (`Ctrl+Alt+Shift`), click **Shared Drive**, then **Upload File**, and select your `.ovpn`. It lands in the `GuacShare` folder, visible in This PC in File Explorer.
2. In MobaXterm on the operator, open the **Apache Redirector (SSH)** bookmark under the redStack Lab session folder. 
	1. Accept the connection to the redirector
	2. Enter the admin password from the `deployment_info.txt`
		1. Optional: Choose to allow it to save your password, and enter a Master Password for the MobaXterm password vault
3. MobaXterm opens two panes: a terminal session on the right (you are logged in as `admin@redirector`) and an SFTP browser on the left side pane. The SFTP pane opens in `/home/admin` by default. Double-click into the `vpn` folder to navigate there.
4. With `vpn` open in the SFTP pane, click the "Upload to current folder" button in the SFTP toolbar and pick the `.ovpn` from `This PC>GuacShare`.

Keep this MobaXterm terminal open. Step 9 commands run here, in this same redirector session.

**Success:** in the same MobaXterm terminal, `ls ~/vpn/` shows exactly one `.ovpn` file.

**Failure:** the `vpn-tunnel` service picks the first file alphabetically if several exist, so keep only one in `~/vpn/`.

### Step 9. Start the tunnel

From redirector SSH session:

```bash
sudo systemctl start vpn-tunnel && sudo journalctl -u vpn-tunnel -f | grep -m1 "Initialization Sequence Completed"
```

This starts the service and tails the log only until the handshake lands. `grep -m1` exits on the first match, which drops the follow and returns you to the prompt with the `Initialization Sequence Completed` line printed. No manual quit, no getting stuck in the pager.

**Success:** the command prints `Initialization Sequence Completed` and hands you back the prompt within a few seconds.

If it does not return within ~30 seconds, the handshake has not completed. Ctrl-C out and check status (no pager, nothing to quit):

```bash
systemctl status vpn-tunnel --no-pager -l
```

**Failure:** the service will not start if `~/vpn/` is empty (Step 8). The status command above shows auth or route errors inline.

> [!IMPORTANT]
> Generate C2 agents only after the tunnel is up. Beacons always call back to the redirector public Elastic IP (in `deployment_info.txt`), never the `tun0` IP: HSL does not route VPN client IPs back to the target, so a `tun0` callback never returns. Full callback config is in CONFIG and ATTACK.

### Step 10. Confirm reachability to the target

ShadowGate's address is per-instance and changes by location, so this guide writes it as `<ShadowGate IP>` rather than a fixed value. Read your assigned address off the HSL portal for your instance and substitute it everywhere `<ShadowGate IP>` appears below. Confirm it falls inside your `vpn_tunnel_cidrs` (`10.1.0.0/16` by default); if your address sits in a different range, change the CIDR to contain it.

On the Windows workstation, open PowerShell (Start menu, type `powershell`, Enter). Run these one at a time, not as a block:

```powershell
ping <ShadowGate IP>
```

```powershell
Test-NetConnection <ShadowGate IP> -Port 445
```

ShadowGate answers ICMP, so a ping reply confirms the path; `TcpTestSucceeded : True` on 445 confirms the service is reachable.

**Success:** a reply from `<ShadowGate IP>` confirms the full path is live: internal host to Guacamole (WireGuard) to redirector (OpenVPN) to the HSL target network. The lab is ready for the attack path.

**Failure:** isolate the break from the redirector (SSH in over Guacamole), one command at a time:

```bash
ip route
```

Expect `X.X.0.0/16` (your target CIDR from `terraform.tfvars`) via ... dev tun0.

```bash
ping -c3 <ShadowGate IP>
```

Redirector to target, over OpenVPN.

If the redirector cannot reach the target either, the OpenVPN tunnel or HSL routing is the problem, not the internal path. Recheck Steps 8 and 9, and give the target the HSL boot window (about 5 minutes) before assuming a routing fault.

---

## Where this hands off

At the end of Phase 3 the lab is deployed, verified, tunneled, and ShadowGate is reachable. Next is CONFIG, which stands up the three C2 backends (Sliver, Mythic, Adaptix) and confirms a test beacon from each. ATTACK follows with the ShadowGate chain. Teardown and cost live at the end of ATTACK, since the stack stays up across all three guides; do not destroy between them.

Note: the tunnel does not auto-start after a stop or reboot. If you stop the stack between sessions rather than destroying it, re-run `sudo systemctl start vpn-tunnel` on the redirector when you resume (WireGuard comes back automatically).
