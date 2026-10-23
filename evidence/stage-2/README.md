# Stage 2 Evidence Index

Each row says what the file shows, checked against the screenshot itself. Captured 2026-10-22, us-east-2 (Ohio).

## Captured and verified

| File | What it proves |
|---|---|
| `2.0-credits-before-build.png` | Credits remaining $99.75, current month cost $2.92; the "Launch an instance using EC2" credit activity is still In progress. |
| `2.1-vpc-create-form.png` | `pmbrs-vpc` created with IPv4 CIDR `10.20.0.0/16`, no IPv6, default tenancy. This is the create form, not the finished VPC. |
| `2.1-subnets.png` | Three subnets in `pmbrs-vpc`: `pmbrs-public-a` 10.20.1.0/24 (us-east-2a), `pmbrs-public-b` 10.20.2.0/24 (us-east-2b), `pmbrs-private-a` 10.20.11.0/24 (us-east-2a). 251 available IPs each. The default-VPC 172.31.x subnets also appear in the list. |
| `2.2-public-rt-associations.png` | `pmbrs-public-rt` (rtb-0019e19b4f4c1d7c9) explicitly associated with public-a and public-b. At this point private-a still fell back to the main table. |
| `2.3-route-tables-list.png` | Four route tables. `pmbrs-private-rt` (rtb-06638e306bf7fe88d) is explicitly associated with `pmbrs-private-a`. `pmbrs-public-rt` has 2 subnets and no subnets left on the main table. |
| `2.4-app-sg-rules.png` | `pmbrs-app-sg` (sg-00c2f54dcf0eea95c) in `vpc-01e25cea6a93cfebc` (pmbrs-vpc). Inbound 3389, 80 and 8080 from `<admin-public-IP>/32` only. |
| `2.4-db-sg-rules.png` | `pmbrs-db-sg` (sg-0b73681cfd98c96e4) in `vpc-01e25cea6a93cfebc`. Inbound 1521 and 3389, source is the app security group only, no internet source. |
| `2.4-wrong-vpc-db-sg.png` | Incident 11. The first `pmbrs-db-sg` (sg-03e2633138ebee2eb) shows VPC `vpc-06ee39469668a22ad` (the default VPC) and its rules point at the old app security group. Deleted and recreated. |
| `2.4-old-app-sg-rules-before-rename.png` | Before state. The old app security group (sg-0fa0b5a86358e1202, default VPC) still allows 80, 8080 and 3389 from the old IP `<old-admin-public-IP>/32`. It has since been renamed `old-pmbrs-app-sg`. |
| `2.5-amis-available.png` | `pmbrs-db-ami` (ami-0a9564c074e3a098e) and `pmbrs-app-ami` (ami-0a1984988856e828a), both Available, created 2026/10/22 14:09 and 14:10. |
| `2.5-instances-running.png` | `pmbrs-db` (i-01f62c157a1c61a04, m7i-flex.large, us-east-2a, `pmbrs-db-sg`, no public IPv4) and `pmbrs-app-a` (i-09786272b9e4e350b, t3.micro, us-east-2a, `pmbrs-app-sg`, public IPv4 <app-a-public-IP>). The two old instances in us-east-2b are Stopped. Status checks were still Initializing. |
| `2.6-rdp-hop-app-to-db.png` | The administrative path works. On APP-A: private IP 10.20.1.158, mask 255.255.255.0, gateway 10.20.1.1. `lsnrctl` is not recognized there (Oracle is only on the DB). A Remote Desktop session titled `10.20.11.48` is open, and its desktop overlay shows instance i-01f62c157a1c61a04, private IP 10.20.11.48, no public IP, us-east-2a. |
| `2.6-listener-restart-terminal.txt` | Terminal log from the DB. Old IP replaced in four config files, `lsnrctl stop` (no listener), `lsnrctl start`, then `lsnrctl status` showing `Listening on 10.20.11.48:1521` but only the `CLRExtProc` service registered. Contains no credentials. |

## Still to capture (not in what you sent)

These are the gaps in the evidence. The instances are stopped, so grab them when you start the lab for step 2.7.

| Missing file | How to capture |
|---|---|
| `2.1-vpc-details.png` | VPC console, `pmbrs-vpc` details, showing DNS hostnames enabled and DNS resolution enabled. |
| `2.2-igw-attached.png` | Internet gateways page, `pmbrs-igw` (igw-0cc5afcf29cb874b2) state Attached to `pmbrs-vpc`. |
| `2.2-public-rt-routes.png` | `pmbrs-public-rt`, Routes tab: `10.20.0.0/16 local` and `0.0.0.0/0 → igw-0cc5afcf29cb874b2`. This is the screenshot that proves the public subnet definition. |
| `2.3-private-rt-routes.png` | `pmbrs-private-rt`, Routes tab: local only, no `0.0.0.0/0`. |
| `2.6-lsnrctl-stale-ip.png` | Not available. The `TNS-12535` timeout against 172.31.16.100 is described in chat but I have no screenshot or log of it. Either leave it out or reproduce only if easy. |
| `2.6-lsnrctl-registered.png` | `lsnrctl status` on the DB showing `XE` and `xepdb1` READY (after the `local_listener` fix). |
| `2.6-test-netconnection-true.png` | From APP-A: `Test-NetConnection 10.20.11.48 -Port 1521` with `TcpTestSucceeded : True`. |
| `2.6-health-ok.png` and `2.6-batches.png` | From the laptop: `http://<APP-A public IP>:8080/health` and `/batches`. |

## Before you commit

- Screenshots (`*.png`) are kept local and are not committed, because the console header and Owner columns show the AWS account ID and public IPs. This index has the account ID, public IPs and instance hostname redacted.
- The IPs `<admin-public-IP>` and `<app-a-public-IP>` are your current home IP and a temporary public IP. Mask the home IP if you prefer.
- Do not commit `config.py` or any `.backup` files.
