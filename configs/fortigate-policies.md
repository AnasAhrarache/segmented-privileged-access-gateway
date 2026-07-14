# FortiGate Configuration — Segmented Privileged Access Gateway

This document records the actual FortiGate configuration used in this project: interface addressing, administrative access restrictions, and firewall policies enforcing the segmentation between the administration zone, the PAM broker zone, and the protected server zone.

## Network Zones

| Zone | Interface | Subnet | Host |
|---|---|---|---|
| Administration | port1 | 10.10.10.0/24 | AdminVM-1 — 10.10.10.10 |
| PAM Broker | port3 | 10.10.20.0/24 | Broker_VM-1 (JumpServer) — 10.10.20.10 |
| Server | port2 | 10.10.30.0/24 | Server_VM-1 — 10.10.30.10 |

FortiGate sits at the boundary of all three zones. No zone can reach another except through a policy explicitly defined below.

---

## 1. Interface Configuration

Each interface was assigned a static IP, acting as the gateway for its respective zone.

```
config system interface
    edit port1
        set mode static
        set ip 10.10.10.1/24
        set allowaccess ping https ssh
    next
    edit port2
        set mode static
        set ip 10.10.30.1/24
        set allowaccess ping https ssh
    next
    edit port3
        set mode static
        set ip 10.10.20.1/24
        set allowaccess ping https ssh
    next
end
```

## 2. Hardening: Restricting Administrative Access per Interface

By default, all three interfaces allowed `ping`, `https`, and `ssh` — meaning any host in any zone could reach FortiGate's own configuration GUI/CLI. This was identified as an unnecessary exposure and tightened: only `ping` is needed on the server and broker interfaces, since neither should ever need to administer the firewall itself.

```
config system interface
    edit port2
        set allowaccess ping
    next
    edit port3
        set allowaccess ping
    next
end
```

> **Note on port1:** AdminVM-1 retains `https`/`ssh` access to FortiGate in the current lab for configuration convenience. This is a documented, deliberate trade-off — see the "Known Limitations" section of the project report. In a production deployment, FortiGate administration would be moved to a dedicated out-of-band management interface, unreachable from any of the three operational zones.

---

## 3. Baseline Validation — Deny by Default

Before creating any policy, connectivity between zones was tested to confirm FortiGate's implicit-deny behavior:

```bash
# From AdminVM-1 (10.10.10.10)
ping 10.10.30.10   # Server_VM-1 — result: 100% packet loss, no policy exists
```

Result: **all traffic denied**, confirming a secure baseline before any rule was introduced.

---

## 4. Temporary Validation Policy (created, tested, then deleted)

To prove the firewall enforces policies explicitly rather than passing traffic by default, a temporary rule was created:

| Field | Value |
|---|---|
| Name | `TEST-Admin-to-Server` |
| Incoming Interface | port1 |
| Outgoing Interface | port2 |
| Source | all |
| Destination | all |
| Schedule | always |
| Service | ALL |
| Action | ACCEPT |

**Result while active:** `ping 10.10.30.10` from AdminVM-1 succeeded (0% packet loss).
**After deletion:** the same ping failed again, confirming FortiGate's behavior is fully governed by active policy, not by any cached or residual state.

This policy was **not** kept — it existed only to validate the enforcement mechanism.

---

## 5. Final Policies (permanent)

### Policy 1 — Admin-To-Broker

Allows AdminVM-1 to reach JumpServer's web interface.

| Field | Value |
|---|---|
| Name | `Admin-To-Broker` |
| Incoming Interface | port1 |
| Outgoing Interface | port3 |
| Source | all |
| Destination | all |
| Schedule | always |
| Service | HTTP, HTTPS |
| Action | ACCEPT |

### Policy 2 — Broker-To-Server

Allows JumpServer to open SSH sessions to Server_VM-1 on behalf of authorized users.

| Field | Value |
|---|---|
| Name | `Broker-To-Server` |
| Incoming Interface | port3 |
| Outgoing Interface | port2 |
| Source | all |
| Destination | all |
| Schedule | always |
| Service | SSH |
| Action | ACCEPT |

### Implicit Policy — Admin-To-Server (intentionally absent)

| Field | Value |
|---|---|
| Incoming Interface | port1 |
| Outgoing Interface | port2 |
| Action | **DENY (implicit — no policy created)** |

**This is the core of the architecture.** No rule permits port1 → port2 traffic. The only path from the administration zone to the server is through the broker (port1 → port3 → port2), never directly.

---

## 6. Post-Configuration Verification

| Test | From | To | Expected Result | Observed Result |
|---|---|---|---|---|
| Admin → Broker | 10.10.10.10 | 10.10.20.10 | Allowed | ✅ Success |
| Admin → Server | 10.10.10.10 | 10.10.30.10 | Denied | ✅ Failed (as designed) |
| Broker → Server | 10.10.20.10 | 10.10.30.10 | Allowed | ✅ Success |

---

## Summary Table — All Policies

| # | Name | Src Intf | Dst Intf | Service | Action |
|---|---|---|---|---|---|
| 1 | Admin-To-Broker | port1 | port3 | HTTP, HTTPS | ACCEPT |
| 2 | Broker-To-Server | port3 | port2 | SSH | ACCEPT |
| — | Implicit Deny | any | any | ALL | DENY |

---

## Known Limitations (see full report for details)

- AdminVM-1 retains administrative access to FortiGate itself (port1 allows `https`/`ssh`) — a real self-administration gap in this lab, discussed in the project report's scope chapter.
- Policies are currently scoped by **zone/interface**, not by individual host address objects. Adding a second host to any zone would inherit that zone's policy automatically — acceptable for this lab's single-host-per-zone design, but worth tightening with address objects if the topology grows.
- Services are restricted to what each flow actually needs (HTTP/HTTPS, SSH) rather than `ALL`, applying least privilege at the service level in addition to zone-level segmentation.
