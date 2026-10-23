# Stage 2: Migration Notes (default VPC to pmbrs-vpc)

## Approach

Instances cannot move between VPCs, so each server was cloned through an AMI and launched into the new subnets. This kept the proven Oracle / IIS / Flask stack intact instead of rebuilding it, and it exercises a realistic migration path.

| Step | Action | Result |
|---|---|---|
| 1 | Both original instances (`mes-lab-vm`, `pmbrs-app-vm`) were stopped, so images were consistent | n/a |
| 2 | Created one AMI from each server (IDs: `ami-0a9564c074e3a098e` DB, `ami-0a1984988856e828a` app) | Both reached Available |
| 3 | Launched `pmbrs-db` (m7i-flex.large) into `pmbrs-private-a`, no public IP, `pmbrs-db-sg` | Private IP 10.20.11.48 |
| 4 | Launched `pmbrs-app-a` (t3.micro) into `pmbrs-public-a`, public IP enabled, `pmbrs-app-sg` | Private IP 10.20.1.158 |
| 5 | Baseline test (step 2.6) | Passed after the fixes below |

The original default-VPC instances stay stopped as a fallback until the end of Stage 2. They are stopped, not deleted. Their EBS volumes and the AMI snapshots keep billing a small storage cost until cleanup.

## Address changes

| Item | Before | After |
|---|---|---|
| DB private IP | 172.31.16.100 | 10.20.11.48 |
| App private IP | 172.31.22.87 | 10.20.1.158 |
| DB DSN in `config.py` | 172.31.16.100:1521/XEPDB1 | 10.20.11.48:1521/XEPDB1 |

Credentials are not recorded in this repository. `config.py` stays gitignored.

## Incident 11: Security group created in the wrong VPC

**Symptom.** `pmbrs-db-sg` showed VPC ID `vpc-06ee…` (default VPC) on its details page, and its rules referenced the old app group.

**Cause.** The VPC dropdown on the Create security group form defaulted to the default VPC even though the console's VPC filter showed `pmbrs-vpc`.

**Detection.** Checking the VPC ID field on the details page after creation, not just the rules.

**Fix.** A security group's VPC cannot be changed and a group can only reference groups in its own VPC, so the group was deleted (nothing was attached to it) and recreated with VPC set to `pmbrs-vpc`. Rules were re-added with the new `pmbrs-app-sg` as the source.

**Lesson.** A rule can look correct and still be wrong because the group sits in the wrong VPC. Verify the VPC ID of every new resource.

## Incident 12: Stale address in the Oracle configuration after the AMI move

**Symptom (1).** On the DB, `lsnrctl status` reported `TNS-12535: operation timed out` while connecting to `172.31.16.100`, the old address.

**Cause (1).** The AMI carried the old IP in the Oracle network configuration. The new instance no longer has that address, so the listener could not work with it.

**Fix (1).** Replaced the old IP with 10.20.11.48 in every `listener.ora` and `tnsnames.ora` under `C:\app` (four files, including the two `sample` copies; backups taken first as `.backup` files) and restarted the listener. The listener then showed `Listening on 10.20.11.48:1521`.

**Symptom (2).** The listener was up and `Test-NetConnection 10.20.11.48 -Port 1521` from the app server returned `TcpTestSucceeded : True`, but the listener's service list contained only `CLRExtProc`. No `XE` or `XEPDB1` (see `evidence/stage-2/2.6-listener-restart-terminal.txt`). The database instance was healthy (`XE` OPEN, `XEPDB1` READ WRITE) and the Oracle services were running.

**Finding.** A successful TCP test proves the port is reachable, not that the database is registered behind it. The database's `local_listener` parameter held the alias `LISTENER_XE` instead of an address. The alias in `OraDB21Home1\network\admin\tnsnames.ora` already pointed at the new IP after the edit, so the alias in that file was not obviously the broken part. The exact file the database resolved the alias from was not confirmed.

**Fix (2).** Set the parameter to the literal listener address and forced registration:

```sql
alter system set local_listener='(ADDRESS=(PROTOCOL=TCP)(HOST=10.20.11.48)(PORT=1521))' scope=both;
alter system register;
```

`lsnrctl status` then showed `XE` and `xepdb1` as READY. `scope=both` applies the change now and keeps it across restarts.

**Open point.** The root cause of why the alias did not resolve to the new address is not confirmed. A recursive search of `C:\app` for `172.31.16.100` in any remaining `tnsnames.ora` would close this out.

**Detection path.** Followed the layered checklist: route, security group and RDP hop already proven, then TCP test to the port, then listener status, then database registration.

**Lessons.**
- AMI clones carry addresses and hostnames inside application configuration, not just in the OS.
- Layer 4 (port open) passing does not prove layer 7 (service registered).
- Using a name instead of an IP removes this class of problem. A Route 53 private zone in step 2.10 addresses the app side.

## Observations worth keeping

- `ping` to the app server timed out. The security group allows only TCP 80, 8080 and 3389, so ICMP is dropped, and Windows Firewall blocks ICMP by default. A failed ping did not mean there was no route. Use `Test-NetConnection -Port` to test a path.
- `lsnrctl` exists only on the DB. "Not recognized" on the app server is the correct result.
- `Get-NetTCPConnection -LocalPort 8080 -State Listen` showed the System process (PID 4) bound to all interfaces. IIS registers its ports with the HTTP.sys kernel driver, so this is expected and confirms IIS is bound.
- APP-A's public IP changes on every stop/start (no Elastic IP). Private IPs persist.

## Verification

From the public internet, `/health` returned application OK and database OK, and `/batches` returned the three seeded batches. This proves the full path: internet → Internet Gateway → app server (public subnet) → local route → DB (private subnet) → Oracle `XEPDB1`.

## Remaining migration tasks

- Confirm nothing else (scripts, pipeline) still hard-codes the old DSN or IPs.
- At end of Stage 2: decide fallback retirement. Delete old instances, volumes, AMIs and the two `old-…` security groups together.
