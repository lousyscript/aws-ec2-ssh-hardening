# Hardening SSH Access on an AWS EC2 Instance

Hands-on project: I launched an EC2 instance, deliberately exposed SSH to the whole internet, observed what happened, and then locked it down.

**Services used:** EC2, Security Groups, EBS, Docker
**Region:** eu-north-1 (Stockholm)
**OS:** Amazon Linux 2023

## 1. Setup
- Launched a free-tier instance with a key pair (key-only SSH login)
- Installed Docker and ran an nginx container on port 80
- Security group: SSH (22) from my IP only, HTTP (80) from anywhere

Screenshot: `screenshots/04-nginx-running.png`

## 2. Insecure configuration
I changed the SSH rule from **My IP** to `0.0.0.0/0` (open to the whole internet).

Screenshot: `screenshots/01-ssh-open-to-world.png`

## 3. What I observed
Checking the SSH logs (Amazon Linux 2023 uses the systemd journal, not `/var/log/secure`):

```bash
sudo journalctl -u sshd --no-pager | grep -E "Invalid user|Failed|Connection closed|Disconnected"
```

Result: an unknown IP connected to port 22 and dropped the connection without logging in, which is typical of internet scanners looking for open SSH servers.

```
sshd-session[...]: Connection closed by 45.14.109.141 port 51354
```

Screenshot: `screenshots/02-scanner-log.png`

I also checked my own logins:

```bash
sudo journalctl -u sshd --no-pager | grep "Accepted publickey"
```

All successful logins came from my own IPs and used my key. My home IP changed during the project, so I compared the IPs and key fingerprint to confirm there was no unauthorized access.

## 4. Fix
Changed the SSH source back to **My IP** (a single `/32` address).

Screenshot: `screenshots/03-ssh-restricted.png`

## 5. Before vs after

| | Insecure | Secure |
|---|---|---|
| SSH source | `0.0.0.0/0` | My IP (`/32`) |
| Reachable by scanners | Yes | No |
| Auth method | Key only | Key only |

## 6. What I learned
- Open management ports are found by scanners quickly
- Key-only login stops password guessing but doesn't stop the scanning, so restricting the network path is still needed
- Reading `Accepted` vs `Connection closed` log lines helps tell real access from noise
- Home IPs change, so IP-restricted rules need refreshing. For long-term use, AWS Systems Manager Session Manager avoids exposing port 22 at all
- Not every distro logs to `/var/log/secure`. Amazon Linux 2023 uses `journalctl`

## 7. Next steps
- Attach an IAM role so the instance can reach S3 without stored credentials
- Add the custom VPC and subnet practice
- Build a mini-CSPM in Python (boto3) that flags checks like SSH open to `0.0.0.0/0`
