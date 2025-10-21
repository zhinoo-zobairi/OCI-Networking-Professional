# OCI Dynamic Routing Gateway (DRG)

## 1️⃣ What Is a DRG?

The **Dynamic Routing Gateway (DRG)** is OCI’s **virtual router** —  
the central hub for all **non-local traffic** (both North/South and East/West).

It handles:
- 🌍 **North/South** → between OCI and on-premises (via VPN or FastConnect) = crossing the boundary of the cloud network
- 🔁 **East/West** → between VCNs or regions (via Remote Peering) = staying inside your cloud boundaries but moving sideways between internal segments
- 🧭 **Cross-tenancy** → between VCNs in different OCI tenancies( = A tenancy in Oracle Cloud Infrastructure is the root account boundary — basically the organization’s private slice of Oracle Cloud.)

Think of the DRG as your **core router** inside Oracle’s cloud —  every network (VCN, VPN, FastConnect, or Peering link) plugs into it via an **attachment**.

## 2️⃣ DRG Attachments — How Networks Plug In

Each network resource connects to a DRG through an **attachment**.  
Attachments represent *physical/logical links* to the DRG.

| Attachment Type | Purpose |
|------------------|----------|
| **VCN Attachment** | Connects a Virtual Cloud Network (up to 300 per DRG). |
| **RPC (Remote Peering Connection)** | Links two DRGs (same or different region) → region-to-region communication. |
| **IPSEC_TUNNEL** | Each IPSec VPN creates **two tunnels**, each visible as a separate attachment. |
| **VIRTUAL_CIRCUIT** | FastConnect links (private fiber connections) attach here. |
| **LOOPBACK** | Used for *VPN over FastConnect* (the DRG’s local tunnel endpoint). |
| **CROSS-TENANCY** | Connects a VCN in another tenancy to this DRG. |

🧠 **Key Insight:** Every attachment behaves like a *port* on the virtual router.  
Traffic enters (ingress) and leaves (egress) through these attachments.

## 3️⃣ DRG Route Tables — The Heart of Routing Logic

Every **attachment** has a **DRG Route Table** associated with it.

- When a packet **enters the DRG**, it triggers a **route lookup** in *that attachment’s route table*.
- The lookup determines the **next hop attachment** (where the traffic should exit).
- ⚠️ **Only inbound lookups occur** — OCI does **not re-evaluate routing on egress**.

### Example

1. A packet leaves a VM in a VCN → the **VCN route table** points to the DRG.  
2. The DRG receives it via the VCN attachment → checks the **DRG route table** tied to that attachment.  
3. Finds route: `Destination = on-premises CIDR → Next hop = IPSec Tunnel Attachment`.  
4. DRG forwards the packet → VPN encrypts → sends to on-premises.  
5. ✅ **No second lookup** on the tunnel egress — routing is *one-directional per hop.*

## 4️⃣ Import Route Distributions — Dynamic Route Control

**Import Route Distributions** define **what routes** are automatically imported into a DRG route table.

They are like **filters and policies** for dynamic route learning.

For example:
```text
Import all routes learned from FastConnect attachments into DRG Route Table A.
````

You then associate that route distribution with one or more DRG route tables.
Result → those tables automatically learn all the on-premises routes advertised over FastConnect.

🧩 Relationship summary:

```
Import Route Distributions  → populate →  DRG Route Tables  → used by →  DRG Attachments
```

* **One-to-many** relationships across all three objects.
* Enables fine-grained routing control (who learns what, from whom).
## 5️⃣ DRG Traffic Flow Overview

| Step | Component         | Function                                         |
| ---- | ----------------- | ------------------------------------------------ |
| 1    | VCN Route Table   | Forwards to DRG attachment                       |
| 2    | DRG Attachment    | Ingress to DRG                                   |
| 3    | DRG Route Table   | Determines next hop                              |
| 4    | Target Attachment | Forwards to VPN, FastConnect, or remote VCN      |
| 5    | Optional Import   | Adds learned routes dynamically (e.g., from BGP) |

🧠 **Rule of thumb:**
Routing **decision** = on ingress.
Routing **action** = on egress.
But no re-evaluation happens after the initial lookup.

## 6️⃣ ECMP (Equal-Cost Multi-Path Routing)

* DRG supports **ECMP**, meaning if multiple routes exist to the same destination with equal cost,
  traffic can be **load-balanced** across them (flow-based distribution).
* Enables **redundancy + bandwidth aggregation**.
* Example: two FastConnect circuits to the same on-prem router.

## 7️⃣ Monitoring & Metrics

DRG metrics track:

* **Inbound/Outbound Traffic** — bytes or packets
* **Drops** — categorized for quick diagnosis:

  * `No Available Route` → missing routing entry
  * `Throughput Drop` → bandwidth limit exceeded
  * `Other` → MTU exceeded, TTL expired, or unsupported transit routing (e.g., on-prem → on-prem via DRG)

Metrics can be filtered by:

* Route table
* Attachment type
* Peer region
* Drop type

🧭 Use OCI Monitoring dashboards or Alarms to visualize DRG health per attachment.

## 8️⃣ Design & Scalability Highlights

* **Up to 300 VCNs** can attach to a single DRG.
* Acts as a **single point of policy** for all routing decisions.
* Greatly reduces troubleshooting complexity — all paths converge on one gateway.
* Cross-region or cross-tenancy peering allows multi-region and hybrid designs.

## 9️⃣ Key Takeaways

| Concept                               | Summary                                                 |
| ------------------------------------- | ------------------------------------------------------- |
| **DRG = Virtual Router**              | Centralized control for all OCI routing beyond the VCN. |
| **Attachments = Interfaces**          | VCNs, VPNs, FastConnects, RPCs connect through these.   |
| **Route Tables = Rules**              | Define next hops per attachment.                        |
| **Import Distributions = Automation** | Control what dynamic routes are learned.                |
| **Lookup = Inbound Only**             | Routing decision made once per ingress.                 |
| **ECMP = Load Balancing**             | Efficient multi-path utilization.                       |
| **Metrics = Observability**           | Detect missing routes, bandwidth issues, MTU/TTL drops. |


**In one line:**

> The DRG is the **brain** of OCI hybrid and multi-VCN networking — a centralized router that unifies on-prem, inter-region, and intra-VCN communication under one scalable, policy-driven control plane.


# 🪐 OCI Realms — The Highest Level of Cloud Isolation

## 🧭 1️⃣ What Is a Realm?

A **Realm** is the *largest* administrative and physical boundary in Oracle Cloud Infrastructure (OCI).  
It’s a **completely separate cloud universe** — each realm has its own:
- Account and identity systems (tenancies)
- Networking backbone
- Control plane and data plane
- Compliance regime

> 🧩 You can think of a realm as a *completely separate cloud provider* —  
> even though Oracle operates all of them.

## 🌍 2️⃣ Realm → Region → Tenancy → VCN Hierarchy

```

[Realm]  → contains →  [Regions]  → each contains →  [Tenancies]
→ inside tenancy →  [VCNs, Subnets, DRGs, etc.]

```

| Level | Example | Scope | Interconnection |
|--------|----------|-------|-----------------|
| **Realm** | Commercial, Government, EU Sovereign | Global Isolation | ❌ Never interconnected |
| **Region** | `eu-frankfurt-1`, `us-phoenix-1` | Metro Area | ✅ Connected inside same Realm |
| **Tenancy** | Your OCI account | Organization Boundary | ✅ Can connect via RPC (if same realm) |
| **VCN** | Virtual network in a region | Cloud LAN | ✅ Connect via DRG / LPG |

## 🔒 3️⃣ Key Properties of Realms

| Property | Description |
|-----------|-------------|
| **Isolation** | No shared accounts, data, or backbone between realms. Each is a separate cloud. |
| **Private Backbone** | Regions *within* a realm are connected by Oracle’s private backbone — **but not across realms**. |
| **Dedicated Tenancy** | A tenancy exists *only within one realm*. If you need both commercial and gov access, you need two separate tenancies. |
| **No Native Peering** | DRG peering (RPC/LPG) only works within the same realm. |
| **Different Use Cases** | Realms are designed for **compliance separation** — e.g., commercial vs. sovereign government workloads. |

## 🏗️ 4️⃣ Real-World Analogy

Imagine Oracle runs **multiple parallel clouds**, each walled off like separate countries:

| Realm | Analogy | Purpose |
|--------|----------|---------|
| 🏢 **Commercial Realm** | Public Internet | Regular enterprise workloads |
| 🇺🇸 **US Government Realm** | Military-grade private cloud | FedRAMP / DoD compliance |
| 🇪🇺 **EU Sovereign Realm** | Schengen-law–compliant zone | Data residency and GDPR alignment |
| 🇷🇸 **Serbia Cloud Realm** | Localized sovereign deployment | National data control |
| 🏠 **Dedicated Region / Alloy Realm** | Private city | Oracle-managed cloud *inside* customer datacenter |

Each “country” has:
- Its own citizens (tenancies)
- Its own roads (regions and backbones)
- Its own laws (compliance rules)

There are **no highways** between countries — only **on-prem or carrier VPNs/FastConnect** can bridge them.

## 🚧 5️⃣ How to Connect Realms (If You Must)

Because **Remote Peering Connections (RPCs)** can’t cross realms, the only options are:
1. **Through On-Premises:**
   - Connect both realms to your corporate datacenter via **FastConnect or VPN**.
   - Route traffic **through your on-prem router/firewall**.
2. **Via Multi-Cloud Carrier:**
   - Some telecoms offer private inter-realm circuits.
   - Still behaves like on-prem routing — not native OCI interconnect.

📍 **Example:**
```

[VCN A in Commercial Realm]
↕ FastConnect
[On-Prem Router]
↕ VPN/FastConnect
[VCN B in Government Realm]

```



### 🧠 In One Sentence:
> OCI **realms** are completely separate clouds —  
> regions inside a realm share Oracle’s backbone,  
> but **different realms are as isolated as AWS vs Azure.**

# Border Gateway Protocol (BGP) and OCI Connectivity

### The Big Picture

When your corporate network connects to Oracle Cloud (OCI), two independent worlds need to exchange routes and forward packets intelligently.  
- Your **on-premises environment**: switches, routers, firewalls — sometimes physical, sometimes virtual.  
- Oracle’s **cloud backbone**: a vast autonomous system hosting your Virtual Cloud Network (VCN).

To make these two worlds understand each other, they need:
1. A **protocol** to share routing information → **BGP (Border Gateway Protocol)**  
2. A **router on your side** to speak that protocol → **CPE (Customer Premises Equipment)**  
3. A **router on Oracle’s side** to speak back → **DRG (Dynamic Routing Gateway)**

These are the three actors in every hybrid network story.

### How Traffic Knowledge Is Shared

BGP is not about moving traffic; it’s about teaching routers **which prefixes exist** and **where they live**.

| Direction | Speaker | What It Announces |
|------------|----------|-------------------|
| On-prem → OCI | Your CPE router | Your corporate subnets, e.g. `192.168.1.0/24` |
| OCI → On-prem | Oracle’s DRG | Your cloud subnets, e.g. `10.20.0.0/16` |

So when you read “advertise 192.168.1.0/24 on both VC1 and VC2,”  
that means *your router* tells Oracle’s DRG:  
> “If you ever need to reach 192.168.1.0/24, use either of these circuits.”

Traffic only begins to flow *after* both sides have exchanged and installed these advertisements in their routing tables.
### The Two BGP Speakers

**CPE – Customer Premises Equipment**  
- The edge router that represents *your* network.  
- Could be a hardware appliance (Cisco, Juniper, Fortinet) or a **virtual router** running in a VM (VyOS, pfSense, CSR1000v).  
- Terminates your VPN or FastConnect and runs BGP toward Oracle.

**DRG – Dynamic Routing Gateway**  
- Oracle’s router inside OCI.  
- Terminates your VPN or FastConnect on the cloud side.  
- Speaks BGP to your CPE and advertises your VCN prefixes.  
- Without it, your VCN would be a closed bubble with no external routes.

Together they form a **BGP peering relationship**:
```

[CPE — your ASN 65001] ⇄ (BGP session) ⇄ [DRG — Oracle ASN 31898]

```

### Physical vs. Virtual Routers

A router is defined by *function*, not by form.  
It reads packet headers, consults routing tables, and forwards packets to the next hop.  

That logic can live:
- **In hardware** – purpose-built box with ASICs and physical interfaces.  
- **In software** – a VM or container performing the same functions.

Even a virtual router ultimately uses **real physical NICs** underneath — it just reaches them through software layers provided by a **hypervisor**.

### What Actually Happens Inside a “Virtual Router”

```

Physical Server
├── Physical NIC (eth0)  ← cable to switch
├── Hypervisor (ESXi, KVM, Hyper-V)
│    ├── Virtual Switch (vSwitch)
│    │    ├── vNIC: Router-VM eth0 → internal VLAN
│    │    └── vNIC: Router-VM eth1 → uplink VLAN
│    └── Bridges the vSwitch to the physical NIC
└── Router-VM (e.g. VyOS, CSR1000v)
├── Runs BGP, IPsec, NAT
└── Acts as your CPE

```

Packets leaving that VM still hit copper or fiber — they just travel through the hypervisor’s **virtual switch** before the real network card.

### The Hypervisor: The Layer Making Virtualization Possible

A **hypervisor** is the software that sits directly above the hardware and allows multiple virtual machines to share one server.  
It:
- Creates and manages VMs.  
- Allocates virtual CPUs, memory, and NICs.  
- Connects those vNICs via virtual switches.  
- Enforces isolation between guests.

| Type | Runs On | Examples | Where Used |
|------|----------|-----------|------------|
| **Type 1 (bare-metal)** | Directly on hardware | VMware ESXi, KVM, Xen, Hyper-V Server | Data centers, clouds |
| **Type 2 (hosted)** | On top of another OS | VirtualBox, Parallels | Desktops, labs |

Every cloud hypervisor (including Oracle’s) is Type 1 — that’s what lets them carve one physical server into many independent VMs, including your “virtual router.”
### The End-to-End View

```

On-Prem Network
192.168.1.0/24
│
│  CPE Router (physical or virtual via hypervisor)
│     ↳ advertises on-prem prefixes via BGP
│
└── FastConnect / VPN (two circuits VC1 & VC2)
│
▼
Oracle Cloud (OCI)
DRG (Dynamic Routing Gateway)
↳ advertises VCN prefixes (10.20.0.0/16)
↳ learns on-prem prefixes (192.168.1.0/24)
↳ installs best paths based on BGP attributes
│
└── VCN with public/private subnets

```

1. Your CPE advertises your local networks.  
2. Oracle’s DRG advertises your cloud subnets.  
3. Both sides learn each other’s routes dynamically through BGP.  
4. Packets flow symmetrically over the chosen FastConnect or VPN tunnels.

# Deep Dive: BGP Route Selection and OCI Routing Behavior

### The BGP Best Path Algorithm

When multiple paths exist to reach a destination, BGP uses a **best path selection** process.  
Oracle Cloud follows the same core logic used by the internet:

1. **Longest Prefix Match (Most Specific Wins)**  
   - `/32` is preferred over `/24` if both include the same IP.  
   - This happens *before* BGP attributes are evaluated.

2. **Shortest AS Path Wins**  
   - The path that crosses fewer autonomous systems is chosen.  
   - Each ASN in the path represents a hop between administrative domains.

3. **Highest Local Preference Wins (for outbound traffic)**  
   - A value manually set by the administrator to prefer one path over another.  
   - Only considered inside your own ASN.

### FastConnect vs. VPN

| Connection Type | Routing Method | Notes |
|------------------|----------------|-------|
| **FastConnect** | **BGP required** | Dedicated private link; always uses dynamic routing. |
| **IPsec VPN** | **BGP or static** | Two HA tunnels; each can be BGP or static. |
| **Best Practice** | Always use BGP for both tunnels. | Keeps routing dynamic and failover automatic. |

Oracle always prefers **FastConnect** over **VPN**, because:
- FastConnect provides private, direct connectivity.  
- VPN runs over the public internet and is less predictable.  

### How Oracle Ranks Paths Internally

When Oracle receives routes from you, it automatically **prepends ASNs** (adds artificial length) to ensure consistent preference:

| Connection Type | Oracle's Default Behavior | Result |
|------------------|---------------------------|---------|
| **FastConnect** | No prepend | Most preferred |
| **IPsec VPN (BGP)** | +1 prepend | Second preferred |
| **IPsec VPN (Static)** | +3 prepend | Least preferred |

This ensures FastConnect → VPN(BGP) → VPN(Static) order of priority.  
If you manually prepend extra ASNs on your FastConnect side, you can *accidentally reverse* this priority — Oracle would then prefer the longer path through VPN instead.
### AS Path Prepending — Controlling Inbound Preference

**AS Path Prepending** = adding fake entries of your ASN to make a route appear “longer” and therefore less preferred.

Example:
```

Primary FastConnect: advertise 192.168.1.0/24 with AS_PATH = 65001
Backup FastConnect:  advertise 192.168.1.0/24 with AS_PATH = 65001 65001 65001

```

Oracle’s DRG will prefer the shorter AS path (Primary) for inbound traffic.  
You now have deterministic primary/backup routing.

### Local Preference — Controlling Outbound Preference

Local preference influences **how your side chooses** a path when sending traffic to OCI.  
Higher = more preferred.

Example:
```

VC1: local-pref 200
VC2: local-pref 100

```
Your CPE will send packets through VC1 first, unless it fails.  
This is commonly used to mirror Oracle’s inbound preference for symmetric routing.

### Symmetric Routing and Why It Matters

In enterprise setups with firewalls or stateful inspection,  
traffic must return on the same path it left — otherwise the firewall will drop it as unsolicited.

- **Inbound (OCI → on-prem):** Controlled by AS path.  
- **Outbound (on-prem → OCI):** Controlled by local preference.  

Balancing both sides keeps the flow symmetric and predictable.

### ECMP – Equal Cost Multi-Path Routing

**ECMP** allows load balancing across multiple equal-cost routes.

- Works for both **FastConnect** and **VPN**, but not a mix of both.  
- Enabled per **DRG route table** in OCI.  
- Up to 8 equal paths supported.

It hashes traffic based on a **5-tuple**:
```

Source IP + Destination IP + Source Port + Destination Port + Protocol

```
Each flow (like one TCP session) always follows one specific path —  
so packets stay in order, but multiple sessions can be spread across links.

To use ECMP:
1. Advertise identical routes with identical AS path lengths.  
2. Configure equal local preferences on your CPE.  
3. Enable ECMP in the correct DRG route table.

### Deterministic Routing Example (Full Scenario)

```

On-Premises
│
│ Advertise 192.168.1.0/24 on:
│   VC1 → AS_PATH = 65001 (Primary)
│   VC2 → AS_PATH = 65001 65001 (Backup)
│
│ Local Preference:
│   VC1 = 200
│   VC2 = 100
│
▼
Oracle Cloud DRG
Receives two routes:
• 192.168.1.0/24 via VC1 → AS_PATH length 1 (preferred)
• 192.168.1.0/24 via VC2 → AS_PATH length 2 (backup)
DRG selects VC1 for inbound traffic.

```

Result:
- Both sides use VC1 for primary flow.  
- VC2 automatically becomes backup or active if ECMP is enabled.

### OCI Route Table Interaction

In OCI, you can visualize this as multiple route tables:
- **VCN route table** → knows destinations reachable through the DRG.  
- **DRG route table** → knows next hops (FastConnect, VPN).  
- **CPE** → holds your enterprise routing policies (local pref, AS path, etc.).

The DRG handles route lookups for traffic entering OCI from your on-premises side.  
That’s where ECMP and AS-path evaluation actually occur inside Oracle’s infrastructure.

### The Layered Summary

| Layer | Role | Example in This Story |
|-------|------|-----------------------|
| **Physical Layer** | Wires, optics, routers | Cables connecting your data center to Oracle’s edge |
| **Virtualization Layer** | Abstracts physical devices | Hypervisor creating vNICs for router-VMs |
| **Routing Layer (Network)** | Exchanges reachability | CPE ↔ DRG via BGP |
| **Policy Layer** | Decides which path wins | AS Path, Local Preference, ECMP |
| **Cloud Control Layer** | Maps this logic to OCI objects | DRG, Virtual Circuits, Attachments, Route Tables |

### Why This Matters

This architecture is what makes hybrid networking scalable and predictable.  
It lets you:
- Seamlessly extend your LAN into the cloud.  
- Control routing dynamically through policies instead of manual edits.  
- Balance or fail over between multiple physical or virtual circuits.  
- Keep routing symmetric for firewalls and compliance.  
- Scale horizontally through ECMP when bandwidth or redundancy grows.

So behind every “advertise 192.168.1.0/24 on VC1 and VC2” line  
is a coordinated dance between:
- **Real cables and NICs**,  
- **Virtualized routers**,  
- **Dynamic protocols (BGP)**,  
- **Cloud routing policies**,  
all aligning to make one consistent path between your physical and cloud networks.
