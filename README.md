# Segmented Privileged Access Gateway

A personal security lab demonstrating that privileged access to a protected server can be made **architecturally impossible** to bypass — not just discouraged by policy, but physically unreachable at the network layer unless routed through an enforced, credential-vaulting broker.

Built with **FortiGate** (network segmentation and policy enforcement) and **JumpServer** (privileged access broker), orchestrated in **GNS3** on top of **VMware Workstation**.

> 📄 Full write-up: [`report/report_english.pdf`](report/report_english.pdf) (French version also included)
> 🔧 Full FortiGate configuration trail: [`configs/fortigate-policies.md`](configs/fortigate-policies.md)

---

## Table of Contents

- [Project Summary](#project-summary)
- [Why This Project](#why-this-project)
- [Architecture](#architecture)
- [Tools Used](#tools-used)
- [What Was Built](#what-was-built)
- [Validation and Testing](#validation-and-testing)
- [Screenshots](#screenshots)
- [Known Limitations](#known-limitations)
- [Possible Extensions](#possible-extensions)
- [A Note on Wallix Bastion](#a-note-on-wallix-bastion)
- [Repository Structure](#repository-structure)

---

## Project Summary

This lab implements a three-zone network architecture where a FortiGate firewall sits as the **single enforcement point** between:

- an **administration zone** (a workstation an operator uses),
- a **PAM broker zone** (JumpServer, a privileged access management platform),
- a **server zone** (a protected target machine).

The firewall is configured so that **no policy exists** allowing direct traffic between the administration zone and the server zone. The only path from one to the other is through the broker — and the broker itself never exposes the server's real credentials to the operator. Every session brokered through it is fully recorded, keystroke by keystroke, and can be replayed like a video.

This isn't a theoretical diagram — every claim below was tested and verified with real traffic, real policies, and real screenshots.

---

## Why This Project

During a SOC internship, I worked daily with FortiGate, Forcepoint, and Wallix Bastion — enterprise tools for perimeter security and privileged access management. This project was built to deepen that understanding from the ground up: not just operating these categories of tools, but designing, breaking, debugging, and validating the architecture myself, end to end, in a fully self-built lab.

---

## Architecture

```
                         ┌─────────────────────┐
                         │      FortiGate      │
                         │  port1  port2 port3  │
                         └──────────┬───────────┘
              HTTP/HTTPS ───────────┼─────────── SSH
              (allowed)             │           (allowed)
                 │                  │                │
    ┌────────────▼───────┐         │      ┌──────────▼─────────┐
    │     AdminVM-1       │         │      │     Server_VM-1      │
    │   10.10.10.10        │         │      │    10.10.30.10        │
    │ Administration Zone  │         │      │      Server Zone      │
    └───────────────────────┘        │      └────────────────────────┘
              10.10.10.0/24          │                10.10.30.0/24
                         ⨯ NO POLICY — DENIED ⨯
                    (dashed path in diagram above)
                                     │
                         ┌───────────▼───────────┐
                         │      Broker_VM-1        │
                         │   JumpServer (PAM)       │
                         │     10.10.20.10           │
                         │   PAM Broker Zone           │
                         └───────────────────────────┘
                              10.10.20.0/24
```

A polished, vector version of this diagram — along with a step-by-step functional flow diagram (authentication → authorization check → credential vault → session brokering → recording) — is included in the full PDF report.

| Zone | FortiGate Interface | Subnet | Host |
|---|---|---|---|
| Administration | port1 | `10.10.10.0/24` | AdminVM-1 — `10.10.10.10` |
| PAM Broker | port3 | `10.10.20.0/24` | Broker_VM-1 (JumpServer) — `10.10.20.10` |
| Server | port2 | `10.10.30.0/24` | Server_VM-1 — `10.10.30.10` |

---

## Tools Used

| Tool | Purpose |
|---|---|
| [GNS3](https://www.gns3.com/) | Network topology orchestration, linking VMware VMs into a single virtual network |
| [VMware Workstation Pro](https://www.vmware.com/products/workstation-pro.html) | Hypervisor hosting every VM in the lab |
| [FortiGate](https://www.fortinet.com/products/next-generation-firewall) (VM64) | Firewall enforcing network segmentation and access policies |
| Ubuntu Desktop | Administration workstation (AdminVM) |
| Ubuntu Server | Protected target (Server_VM) and PAM broker host (Broker_VM) |
| [JumpServer](https://github.com/jumpserver/jumpserver) | Open-source Privileged Access Management (PAM) platform |

---

## What Was Built

1. **Three isolated network zones**, each behind its own FortiGate interface, with no default connectivity between them.
2. **Static IP addressing** on all three FortiGate interfaces, each acting as the gateway for its zone.
3. **A verified deny-by-default baseline** — confirmed with failed ping tests before any policy existed.
4. **A temporary proof-of-enforcement policy** — a direct Admin→Server rule was created, tested (succeeded), then deleted (failed again) — proving FortiGate's behavior is governed entirely by active policy, not implicit trust.
5. **Two permanent, least-privilege policies**:
   - `Admin-To-Broker` (port1 → port3, HTTP/HTTPS only)
   - `Broker-To-Server` (port3 → port2, SSH only)
   - No policy at all between Admin and Server — this is the architectural core of the project.
6. **JumpServer deployed** as the broker, with:
   - Server_VM-1 registered as an **Asset**
   - Its real SSH credentials stored as a vaulted **Account**, never exposed to end users
   - A dedicated operational user, `Operator1`, separate from the platform's `Administrator` account — enforcing separation of duties
   - An explicit **Authorization** rule linking `Operator1` → the Asset → the Account
7. **A live, browser-based SSH session** established through JumpServer, with zero knowledge of the real password required by the operator.
8. **Full session recording and audit trail**, including login logs, configuration change logs, per-command risk scoring, and full session video replay.

---

## Validation and Testing

Every claim in this project was tested, not assumed:

| Test | From → To | Expected | Result |
|---|---|---|---|
| Baseline (no policy) | Admin → Server | Denied | ✅ 100% packet loss |
| Temporary policy active | Admin → Server | Allowed | ✅ 0% packet loss |
| After policy removal | Admin → Server | Denied | ✅ 100% packet loss again |
| Final policy | Admin → Broker | Allowed | ✅ Reachable (HTTP/HTTPS) |
| Final policy | Broker → Server | Allowed | ✅ Reachable (SSH) |
| Final policy | Admin → Server | Denied | ✅ No path exists — no policy permits it |
| Session brokering | Operator1 → Server_VM-1 via JumpServer | Connects without exposing real password | ✅ Verified, session fully recorded |

Full command-level detail for every FortiGate step is in [`configs/fortigate-policies.md`](configs/fortigate-policies.md).

---

## Screenshots

All screenshots are in [`screenshots/`](screenshots/), organized by stage:

- **`architecture/`** — GNS3 topology, FortiGate dashboard
- **`fortigate-policies/`** — deny-baseline test, temporary validation policy, final policies
- **`jumpserver/`** — asset/account/user/authorization setup, live session connection, audit logs, session replay

---

## Known Limitations

This project is deliberately scoped, and its boundaries are documented rather than hidden:

- **FortiGate self-administration**: AdminVM-1 currently retains access to FortiGate's own configuration interface. In production, firewall management would sit on a dedicated, out-of-band management network — not reachable from any operational zone. See the report for the full discussion.
- **Compromised endpoint + legitimate broker session**: network segmentation prevents any *direct* bypass of the broker, but does not by itself prevent an attacker who has also obtained `Operator1`'s active JumpServer session from using it exactly as the legitimate user would. MFA, short session lifetimes, and command filtering (all discussed in the report) address this gap.
- **Not a complete Zero Trust architecture**: this project applies specific Zero Trust principles (deny-by-default, explicit enforcement, least privilege, separation of duties) to the privileged-access perimeter. It does not implement a centralized identity provider, continuous identity/context verification, endpoint posture checks, or a dynamic policy engine — the components required for a full, organization-wide Zero Trust model.
- **Zone-level, not host-level, policies**: policies are currently scoped per interface/zone. Adding a second host to an existing zone would inherit that zone's policy automatically. Tightening this further would require FortiGate address objects scoped to individual hosts.

---

## Possible Extensions

- Restrict policies to specific address objects instead of entire zones.
- Enable multi-factor authentication (MFA) on JumpServer accounts.
- Configure JumpServer Command ACLs to block or flag risky commands mid-session.
- Feed FortiGate and JumpServer logs into a SIEM for correlation and anomaly detection.
- Add a second target server with a distinct access tier, to demonstrate finer-grained authorization.
- Replace JumpServer with Wallix Bastion (see below) — the network architecture and FortiGate policies would require no changes.

---

## A Note on Wallix Bastion

This project was originally planned around **Wallix Bastion**, the commercial PAM solution used in my SOC internship. The evaluation license request required vendor-side commercial validation whose timeline didn't fit the project schedule, so the lab was built with **JumpServer**, an open-source PAM platform sharing the same core architectural principles (credential vaulting, session brokering, recording, granular authorization).

Importantly, the network segmentation and FortiGate policies designed here are **entirely independent of the specific PAM product** deployed. If a Wallix Bastion license becomes available, it can be substituted into the broker zone without any change to the network architecture — only the broker's internal configuration would need to be redone.

---

## Repository Structure

```
segmented-privileged-access-gateway/
├── README.md
├── report/
│   ├── report_english.pdf
│   └── rapport.pdf              # French version
├── configs/
│   └── fortigate-policies.md    # Full FortiGate CLI/GUI configuration trail
└── screenshots/
    ├── architecture/
    ├── fortigate-policies/
    └── jumpserver/
```

---

**Author:** Anas Ahrarache
Master's student, IT Security and Big Data — Faculté des Sciences et Techniques de Tanger, Université Abdelmalek Essaâdi
