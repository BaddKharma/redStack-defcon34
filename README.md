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
  <img src="https://img.shields.io/badge/C2-Mythic%20%7C%20Sliver%20%7C%20Havoc-C0392B?style=for-the-badge">
</p>

redStack is an open source AWS and Terraform project that stands up a full red team operator stack on demand: three C2 frameworks (Mythic, Sliver, Havoc), a Kali operator, a Windows operator, an Apache redirector with header and URI gating, and a Guacamole portal fronting the environment. One `terraform apply` brings it all up.

This is the landing page for the DEF CON 34 workshop. Slides, syllabus, and supporting docs live here. The platform itself lives in the redStack repo linked below.

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
| Adversary Village | Date TBD |
| Noob Village | Saturday, Aug 8, 1:00-2:50pm |

You deploy the stack yourself, walk the operator portal, stand up your C2s, and follow a full attack chain against a live range, landing a Sliver beacon and escalating to full control. The session closes with a clean teardown and cost management.

---

## redStack Platform

The workshop runs against the live redStack project. Read these before the session.

- **redStack repo:** https://github.com/BaddKharma/redStack
- **redStack README and wiki:** https://github.com/BaddKharma/redStack/wiki

The wiki is the technical source of truth. Deployment, architecture, and the OpenVPN tunnel setup (wiki page 15) are documented there.

Syllabus and slides will be posted here.

---

## Before You Arrive

Full prerequisites are in the syllabus. In short, arrive with:

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

**Michael Ortiz** is a Red Team Engineer and SME, U.S. Department of State Red Cell. Founder of devZero Security (SDVOSB, offensive security and security engineering) and developer of redStack. Marine Corps veteran. OSEP, OSCP, CRTO, CRTL. Speaker profile: https://sessionize.com/mike-ortiz

**Michael Kim** is a Senior Consultant in Offensive Security on Palo Alto Networks Unit 42's Proactive Services team, leading red team operations, adversary simulations, and penetration tests for global clients. He moved into cybersecurity during the pandemic and rose from boot camp graduate to senior consultant, bringing a multicultural background across seven countries to his approach to offensive work.

---

## Acknowledgments

Thanks to Tyler Ramsby at Hack Smarter Labs for authorizing the range for this workshop. The live attack portion runs against a Hack Smarter Labs range with their permission, on per-user instanced connections.
