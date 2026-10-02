# Runbook: Prisma Access – Aggregate Remote Network (US-East, 200 Mbps)

| | |
|---|---|
| **Purpose** | Onboard a branch/site to Prisma Access as a Remote Network (RN) in the US-East compute location using **aggregate bandwidth** licensing, with 200 Mbps allocated. |
| **Managed by** | Strata Cloud Manager (SCM). Panorama-managed equivalents are noted inline. |
| **Risk** | Low–Medium. Config push affects tenant config; new RN does not impact existing RNs/mobile users. |
| **Est. time** | 45–90 min plus tunnel/BGP validation |

---

## 0. Key concepts (read first)

- **Aggregate bandwidth** is allocated **per compute location**, not per site. The 200 Mbps is a shared pool for *all* RNs homed to the US-East compute location. It is not a 200 Mbps cap on this one site.
- Allocation is enforced at the compute location level; individual tunnels are not rate-limited to the pool size, but a single IPSec tunnel has a practical throughput ceiling (~1 Gbps). Not a concern at 200 Mbps.
- Each RN needs a **primary** tunnel and (strongly recommended) a **secondary** tunnel to a different Prisma Access node for HA.
- Prisma Access assigns tunnel peer IPs and a service IP only after the first successful commit/push. Do not configure the on-prem side until you have them.

> **Verify before starting:** confirm in your license/CSP portal that the tenant has **Aggregate Bandwidth** licensing (not per-location "Location-based"), enough **unallocated Mbps** for 200, and check the current **minimum allocation per compute location** in the Palo Alto docs for your release. Minimums have changed over time. If the minimum exceeds 200, the request must be adjusted.

---

## 1. Prerequisites / inputs

Collect these before starting.

| Item | Value |
|---|---|
| Site name | `<SITE-NAME>` |
| Compute location | **US East** |
| Allocated bandwidth | **200 Mbps** |
| Branch device | `<vendor/model/OS>` (supports IKEv2 + IPSec, ideally BGP) |
| Branch public IP – primary | `<x.x.x.x>` |
| Branch public IP – secondary (if dual ISP) | `<x.x.x.x>` or same as primary |
| Branch subnets to advertise | `<10.x.x.0/24, ...>` |
| Routing | BGP (preferred) or static. If BGP: branch ASN `<ASN>`, Prisma Access ASN (default 65534) |
| IKE / IPSec parameters | Agreed crypto (see §4), PSK `<stored in vault>` |
| IKE ID type/value | e.g. FQDN/IP `<id>` |
| Change ticket | `<CHG#>` |
| Access | SCM role: Superuser or Prisma Access/Network admin |

**Prerequisite checks**
- [ ] Prisma Access tenant is activated; infrastructure subnet, BGP ASN and (if used) IP pool already configured under *Prisma Access Setup*.
- [ ] Branch subnets do not overlap with other RNs, mobile user pools, or the infrastructure subnet.
- [ ] Branch firewall allows UDP 500/4500 and ESP to/from Prisma Access peer IPs (obtained in §6).
- [ ] Change window approved.

---

## 2. Allocate 200 Mbps to US-East

**SCM:** *Workflows → Prisma Access Setup → Remote Networks → Bandwidth Management* (path may vary slightly by release)

1. Open **Bandwidth Management**.
2. Confirm licensing type is **Aggregate**.
3. Locate **US East** in the compute location list. Add it if not present.
4. Set allocated bandwidth to **200** Mbps.
   - If US-East already has an allocation, **add** to it rather than overwrite, so existing RNs are not starved. Record the previous value: `____ Mbps`.
5. Save. (Allocation takes effect with the config push in §6.)

**Panorama:** *Panorama → Cloud Services → Configuration → Remote Networks → Bandwidth Management (gear icon)* → set US East to 200 Mbps → OK.

**Check:** total allocated across locations must not exceed licensed aggregate Mbps.

---

## 3. Create IKE/IPSec crypto profiles (skip if reusing)

**SCM:** *Configuration → Network Services → Remote Networks → Templates / Profiles* (or *Prisma Access Setup → Remote Networks → IKE/IPSec Crypto*).

Recommended baseline (match to branch device capability):

| Profile | Setting |
|---|---|
| IKE Crypto | IKEv2, AES-256-CBC (or GCM if supported), SHA-256+, DH group 14/19/20, lifetime 8 h |
| IPSec Crypto | ESP, AES-256-GCM (or AES-256-CBC + SHA-256), PFS group matching, lifetime 1 h |

Prefer a custom profile over the "Suite-B" or default if the branch device requires it. Both ends **must match exactly**.

---

## 4. Create the Remote Network

**SCM:** *Workflows → Prisma Access Setup → Remote Networks → Add Remote Network*

1. **Name:** `<SITE-NAME>-USE`
2. **Region / Location:** the region that maps to **US East** compute location, then select the specific location.
3. **License type:** **Aggregate Bandwidth** (shown as aggregate; no per-RN Mbps field).
4. **Primary tunnel**
   - IPSec tunnel name: `<SITE>-pri`
   - Branch device public IP: `<primary IP>`
   - IKE gateway: authentication = Pre-shared key, enter PSK; local/peer ID per §1
   - IKE crypto profile / IPSec crypto profile from §3
5. **Secondary tunnel** (enable for HA)
   - Name: `<SITE>-sec`, peer IP `<secondary IP>`, same or distinct PSK/profiles
6. **Routing**
   - **BGP:** enable, set peer AS = branch ASN, local address options per branch design; optionally enable *Don't advertise default route* / *Summarize mobile user routes* per design. Enter branch peer IP if required.
   - **Static:** add branch subnets under *Static Routes*. Static is less resilient; use only if the branch cannot do BGP.
7. **QoS (optional):** enable only if the QoS policy is already designed.
8. **Security:** attach intended security policy / zones via existing rulebase. No change needed for a new RN unless it needs a distinct rule.
9. Save.

**Panorama:** *Cloud Services → Configuration → Remote Networks → Add*. Same fields; Location, Aggregate license type, primary/secondary tunnels, routing.

---

## 5. Push configuration

1. **SCM:** *Push Config → Prisma Access (or Push Config at top right) → select Remote Networks / All → Push.*
   **Panorama:** *Commit → Commit and Push → Commit to Panorama, then Push to Devices → Prisma Access (Remote Networks).*
2. Wait for the job to reach **Success** (typically 5–20 min for a new RN).
3. If it fails, open the job details; common causes: overlapping subnets, invalid IKE ID, bandwidth over-allocation.

---

## 6. Get Prisma Access peer details

After a successful push:

- **SCM:** *Remote Networks → select `<SITE-NAME>-USE` → View details* (or *Insights → Remote Networks*).
- **Panorama:** *Cloud Services → Status → Network Details → Remote Networks.*

Record:

| Item | Primary | Secondary |
|---|---|---|
| Prisma Access peer (service) IP | | |
| BGP peer IP (if BGP) | | |
| Prisma Access ASN | | |

Allow these peer IPs in any upstream/edge firewall ACL.

---

## 7. Configure the branch device

Configure to mirror §3/§4 settings:

1. IKEv2 gateway pointing to the Prisma Access primary peer IP; PSK and IDs as set; matching crypto.
2. IPSec tunnel interface + proxy-ID / selectors (`0.0.0.0/0` ↔ `0.0.0.0/0` for route-based).
3. Second tunnel to the secondary peer IP (higher metric / lower BGP preference, e.g. AS-path prepend or lower local-pref).
4. **BGP:** peer with Prisma Access BGP IP, advertise branch subnets only; do **not** advertise default unless intended; filter inbound as needed.
5. **Static:** route toward the tunnel for destinations to send via Prisma Access (e.g. default 0.0.0.0/0 or specific prefixes).
6. Dead-peer detection on; tunnel monitoring if supported.

---

## 8. Validation

- [ ] **Tunnels up:** SCM shows primary (and secondary) tunnel **Connected/Green**. On branch: IKE SA and IPSec SA established.
- [ ] **BGP:** session Established; branch prefixes received by Prisma Access; Prisma Access routes (RN/MU/infra) received at branch.
- [ ] **Connectivity:** ping/traceroute from a branch host to an internet destination and to a known data-center/RN resource.
- [ ] **Egress IP:** confirm branch internet egress is through Prisma Access US-East (e.g. check public IP is a Prisma Access US-East egress IP).
- [ ] **Failover:** (in the change window if allowed) shut primary tunnel; confirm traffic moves to secondary and returns on restore.
- [ ] **Bandwidth:** *Bandwidth Management* shows US East allocated = previous + 200; Insights shows utilization for the compute location. Optionally run a throughput test and confirm traffic is not clipped below expectations.
- [ ] **Logs:** traffic/URL logs from the site appear in Strata Logging Service.

---

## 9. Rollback

1. Shut down tunnels on the branch device; restore previous routing/internet path.
2. In SCM, delete (or disable) the RN `<SITE-NAME>-USE`.
3. If this change added to the US East allocation, reduce it by 200 Mbps (back to `____ Mbps`).
4. Push config and confirm success.

---

## 10. Troubleshooting quick reference

| Symptom | Likely cause / action |
|---|---|
| Push fails: bandwidth | Total allocation exceeds license, or below the compute-location minimum. Adjust in Bandwidth Management. |
| IKE phase 1 down | PSK or IKE ID mismatch, crypto mismatch, UDP 500/4500 blocked, wrong peer IP, NAT in path (enable NAT-T). |
| Phase 2 down | IPSec crypto/PFS mismatch, proxy-ID/selector mismatch. |
| Tunnel up, no BGP | Wrong peer IP/ASN, BGP not allowed over tunnel interface, MD5/auth mismatch, tunnel interface IP/mask. |
| Tunnel up, no traffic | Missing routes, overlapping subnets, security policy not permitting the RN zone, asymmetric routing with the secondary. |
| Throughput below 200 Mbps | Check whether the pool is shared and consumed by other RNs in US East; check branch ISP and device crypto capacity; check MTU/MSS (try MSS 1350). |

---

## 11. Sign-off

| Step | Owner | Time | Done |
|---|---|---|---|
| Bandwidth allocated (200 Mbps, US East) | | | ☐ |
| RN created and pushed | | | ☐ |
| Branch configured | | | ☐ |
| Tunnels/BGP validated | | | ☐ |
| Failover tested | | | ☐ |
| Monitoring/alerts updated | | | ☐ |
| Documentation/CMDB updated | | | ☐ |

*Menu paths and defaults vary by Prisma Access / SCM release. Confirm against the current Palo Alto Networks documentation before executing.*
