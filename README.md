# redStack: Boot-To-Breach Red Team Platform Workshop

**DEF CON 34. Red team infrastructure on AWS, deployed live in two hours.**

<p align="center">
  <img src="https://img.shields.io/badge/DEF%20CON-34-B31942?style=for-the-badge&labelColor=0d1117">
  <img src="https://img.shields.io/badge/Format-Hands--On-B31942?style=for-the-badge&labelColor=0d1117">
  <img src="https://img.shields.io/badge/Duration-2%20Hours-B31942?style=for-the-badge&labelColor=0d1117">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazonaws&logoColor=FF9900">
  <img src="https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white">
  <img src="https://img.shields.io/badge/C2-Mythic%20%7C%20Sliver%20%7C%20Adaptix-C0392B?style=for-the-badge">
</p>

redStack is an open source AWS and Terraform project that stands up a full red team operator stack on demand: three C2 frameworks (Mythic, Sliver, Adaptix), a Kali operator, a Windows operator, an Apache redirector with header and URI gating, and a Guacamole portal fronting the environment. One `terraform apply` brings it all up.

This is the landing page for the DEF CON 34 workshop. The four workshop guides, slides, and supporting docs live here. The platform itself lives in the redStack repo linked below.

> **Note (2026-07-18): C2 backend change.** The third C2 is now AdaptixC2, not Havoc (Havoc was archived upstream). The four guides are updated; the slide decks still show Havoc and will be updated separately. The swap lives on the `dev` branch until it merges to `main`, so clone from `dev`:

```bash
git clone https://github.com/BaddKharma/redStack.git
cd redStack
git checkout dev
```

---

## Workshop at a Glance

| | |
|---|---|
| **Event** | DEF CON 34 |
| **Format** | Two-hour hands-on workshop, instructor-led. Attendees deploy live. |
| **Level** | Intermediate. Prior red team or pentest exposure expected. |
| **Deployment mode** | Tunneled Access (OpenVPN to Hack Smarter Labs) |
| **Target** | Live Hack Smarter Labs range, per-user instanced |

---

## Where to Catch It

| Village | Date and Time |
|---|---|
| Red Team Village | Date TBD |
| Noob Village | Saturday, Aug 8, 1:00-2:50pm |
| Adversary Village | Sunday, Aug 9, 10:00-11:55am |

You deploy the stack yourself, walk the operator portal, stand up your C2s, and follow a full attack chain against a live range, landing a Sliver beacon, escalating to SYSTEM, and also landing in beacons for Mythic and Adaptix. 

---

## Workshop Guides

A prerequisites checklist plus three guides run the workshop end to end, in order. They assume Tunneled Access and are written for this session, not general use. The stack stays up across all three guides and is destroyed at the end of ATTACK.

- [0_PREREQ.md](0_PREREQ.md): what to have ready before the session. Throwaway AWS account, AWS CLI and Terraform, Kali Marketplace EULA, Hack Smarter Labs `.ovpn`, repo clone, SSH key, and your public IP. Do it ahead of time.
- [1_DEPLOY.md](1_DEPLOY.md): initial deployment. Clean AWS account to a running stack with the OpenVPN tunnel up and ShadowGate reachable.
- [2_CONFIG.md](2_CONFIG.md): stand up the three C2 backends (Sliver, Mythic, Adaptix) behind the redirector and confirm a test beacon from each, using the Windows operator as the test platform. The beacons stay up as heartbeats.
- [3_ATTACK.md](3_ATTACK.md): the hands-on chain against ShadowGate. Recon over the tunnel, land a Sliver beacon via the redirector public IP, escalate to full control, then tear down.

## redStack Platform

The workshop runs against the live redStack project.

- redStack repo: https://github.com/BaddKharma/redStack
- redStack wiki: https://github.com/BaddKharma/redStack/wiki

The wiki is the public technical source of truth for the platform, including architecture and the OpenVPN tunnel setup. The four guides above are the operational runbooks for this workshop.

Slides will also be posted here.

---

## Before You Arrive

Full prerequisites and setup steps are in the deployment guide ([1_DEPLOY.md](1_DEPLOY.md), Phase 1). In short, arrive with:

- A dedicated, throwaway AWS account
- AWS CLI installed and configured
- Terraform 1.0 or later
- An SSH key pair created in AWS EC2
- The Kali Linux AMI subscribed in AWS Marketplace (EULA accepted)
- The redStack repo cloned
- An active Hack Smarter Labs account: https://www.hacksmarter.org
- Your provided `.ovpn` file for the assigned range
- A CloudWatch billing alarm set before your first deploy

Budget roughly $2 to $3 of AWS spend for the session. The full stack runs about $0.27 per hour of compute, so a two-hour session is around $0.55 in EC2. The rest is buffer for EBS storage, the Elastic IP, data transfer, and any time the environment stays up before you tear it down. Run `terraform destroy` when you finish to stop the meter.

---

## Instructors

**Michael Ortiz**: https://sessionize.com/mike-ortiz

**Michael Kim**: https://sessionize.com/michael-kim

---

## Acknowledgments

Thanks to Tyler Ramsby at Hack Smarter Labs for authorizing the range for this workshop. The live attack portion runs against a Hack Smarter Labs range with their permission, on per-user instanced connections.

Thanks to P3n3tr@t0r, Cr4ck3rj4ck5, and cyberbandit74 for testing the workshop and providing feedback that shaped these guides.
