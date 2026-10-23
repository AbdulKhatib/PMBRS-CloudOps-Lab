# Stage 2: Network Design

PMBRS-CloudOps-Lab, Stage 2 (custom VPC, public/private tiers). Region: us-east-2 (Ohio).

## Goal

Move the PMBRS MES stack out of the default VPC into a custom VPC with a public app tier and a private database tier, so that the DB has no path to or from the internet and is reachable only from the app tier.

## Address plan

| Item | Value |
|---|---|
| VPC | `pmbrs-vpc`, `10.20.0.0/16` (65,536 addresses) |
| DNS resolution / DNS hostnames | Both enabled (required for the Route 53 private zone in step 2.10) |
| Why 10.20.0.0/16, not the default VPC's 172.31.0.0/16 | Non-overlapping ranges keep peering, VPN and Transit Gateway possible later, and every IP identifies its VPC on sight. Overlapping CIDRs cannot be fixed after the fact (a VPC's primary CIDR cannot be changed). |

| Subnet | CIDR | AZ | Usable IPs | Route table | Purpose |
|---|---|---|---|---|---|
| `pmbrs-public-a` | 10.20.1.0/24 | us-east-2a | 251 | `pmbrs-public-rt` | APP-A, NAT Gateway (temporary, step 2.7) |
| `pmbrs-public-b` | 10.20.2.0/24 | us-east-2b | 251 | `pmbrs-public-rt` | APP-B and second ALB AZ (Stage 4) |
| `pmbrs-private-a` | 10.20.11.0/24 | us-east-2a | 251 | `pmbrs-private-rt` | Oracle DB, no public IP |

Usable IPs = 256 minus the 5 AWS reserves per subnet (network, VPC router, DNS, future use, broadcast). The console showed 251 for each subnet, matching the calculation.

Auto-assign public IPv4 is enabled on both public subnets and disabled on the private subnet.

## Routing

| Route table | Destination | Target | Associated subnets |
|---|---|---|---|
| `pmbrs-public-rt` | 10.20.0.0/16 | local | public-a, public-b |
| `pmbrs-public-rt` | 0.0.0.0/0 | Internet Gateway `pmbrs-igw` | public-a, public-b |
| `pmbrs-private-rt` | 10.20.0.0/16 | local | private-a |
| Main (default) | 10.20.0.0/16 | local | none |

Design decisions:
- A subnet is public only because its route table has `0.0.0.0/0 → IGW`. Nothing on the subnet itself makes it public.
- The private subnet is **explicitly** associated with its own route table instead of falling back to the main table. If someone later added an IGW route to the main table, every subnet relying on it would become public. The main table is kept local-only and unused.
- The most specific route wins (longest prefix). Traffic to anything inside 10.20.0.0/16 uses the `local` route and never touches the IGW or a NAT.

## Security groups

| Group | Inbound | Source | Reason |
|---|---|---|---|
| `pmbrs-app-sg` | TCP 80, TCP 8080, TCP 3389 | Administrator's current public IP (/32) | Browser test, Flask via IIS, and RDP to the jump host |
| `pmbrs-db-sg` | TCP 1521 (Oracle), TCP 3389 | `pmbrs-app-sg` | DB accepts traffic only from members of the app group |

Both groups keep the default outbound rule (all traffic).

Design decisions:
- The DB group references the app **group**, not an IP range, so rules survive IP changes.
- There is deliberately no 3389-from-administrator rule on the DB. The administrative path is laptop → APP-A (public IP) → DB (private IP), a jump host. Session Manager (Stage 5) replaces this later and needs no open RDP port.
- The "My IP" source is a snapshot. If the administrator's IP changes, the rule must be updated.

The old default-VPC groups were renamed `old-mes-lab-vm-sg` and `old-pmbrs-app-sg` so they cannot be mistaken for the new ones. They stay until Stage 2 cleanup because the old instances are still the fallback.

## Instances

| Instance | Type | Subnet / AZ | Private IP | Public IP | Security group |
|---|---|---|---|---|---|
| `pmbrs-db` | m7i-flex.large (8 GB, sized for Oracle) | `pmbrs-private-a` / us-east-2a | 10.20.11.48 | none | `pmbrs-db-sg` |
| `pmbrs-app-a` | t3.micro | `pmbrs-public-a` / us-east-2a | 10.20.1.158 | assigned at start, changes on every stop/start | `pmbrs-app-sg` |

Private IPs persist across stop/start. APP-A's public IP does not (no Elastic IP, to avoid the idle-address charge).

## Layers a connection must pass (troubleshooting order)

1. Is the test valid (correct IP, instance running)?
2. Route table
3. Security group (stateful)
4. Network ACL (stateless: needs inbound and return rules, ephemeral ports 1024-65535)
5. Windows Firewall on the host
6. Application listening on the port

A silent timeout points at layers 2-5. An immediate refusal means the packet arrived and nothing is listening (layer 6).

## Verification (baseline, step 2.6)

- RDP to APP-A on its public IP: works.
- RDP from APP-A to the DB on its private IP: works.
- `Test-NetConnection 10.20.11.48 -Port 1521` from APP-A: `TcpTestSucceeded : True`.
- `/health` from the public internet returns application OK and database OK; `/batches` returns the seeded batches.

This confirms the three requirements for internet reachability on the app side (IGW route, public IP, open security group) and that the DB is reachable only from inside the VPC.

## Planned additions

- Step 2.7: temporary NAT Gateway and `0.0.0.0/0 → NAT` on the private table, tested, then deleted.
- Step 2.9: S3 Gateway VPC Endpoint (a route table entry, no hourly cost).
- Step 2.10: Route 53 private hosted zone so the app connects to the DB by name.
