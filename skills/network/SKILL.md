---
name: lark-uml:network
description: 飞书画板网络拓扑图：执行型 Skill。当用户要在飞书画板上绘制或修改网络拓扑图（网络区域、网关、交换机、主机、服务节点、链路连通关系）时使用。本 skill 直接驱动 lark-cli whiteboard 改写画板，不输出 Mermaid / SVG 代码。
---

# `lark-uml:network`

Specialist skill for **network topology diagrams** on a Feishu / Lark whiteboard. The agent reads, edits, and writes the board itself through `lark-cli whiteboard`. The final artifact is the updated whiteboard, not a code block.

## Inputs

- `board` — whiteboard URL or `wbcn...` token. Required.
- `task` — what to change this turn. Optional; if empty, this is a first-time initialization and the agent designs the topology from scratch.
- `language` — `zh-CN` (default) or `en-US`. Diagram-visible text only.

## Workflow

Follow [`../../references/workflow.md`](../../references/workflow.md) end to end. Stay inside the boundaries in [`../../references/boundaries.md`](../../references/boundaries.md). Apply the language rules in [`../../references/language.md`](../../references/language.md). Apply the native connector rules in [`../../references/connectors.md`](../../references/connectors.md).

**Execution route:** raw-first, native-only. Read the board as raw, edit native zone / device nodes and native connectors in place, then write raw back. Network links and traffic constraints are business relationships, so endpoints must bind to zone, gateway, firewall, switch, host, service, or peer node ids. **No Mermaid / PlantUML / SVG anywhere in the loop, not even as a private sketch.**

**Default mode is modify-in-place.** Duplicate and adapt the template's existing zones, device shapes, and link styles. Only redraw when the user explicitly asks, or the template does not match the actual topology at all.

## Diagram-specific rules

- **Zones first.** Every device sits inside a labeled zone: Internet, DMZ, public subnet, private subnet, VPC, on-prem, partner network. Zones are `subgraph`s with the CIDR range or zone name printed in the header. No "floating" devices outside any zone.
- **Device vocabulary.** Stable shape set:
  - Router / gateway → rectangle labeled `Router` / `GW`.
  - Firewall → rectangle with `FW` label (or a distinct border style if the renderer supports it).
  - Switch / load balancer → rectangle labeled `SW` / `LB`.
  - Host / VM / container → rectangle labeled `Host` / `VM` / `Pod`.
  - Service endpoint → rectangle named after the service.
- **Link semantics.** Lines represent **L2 / L3 connectivity**, not application calls. Annotate the link only with what is true at the network layer: protocol (`TCP/443`, `UDP/53`), port, VLAN, peering relationship. Do not label links with business-level call descriptions — those belong in `lark-uml:architecture`.
- **Direction.** Default to undirected lines for symmetric connectivity. Use arrows only for asymmetric flows (NAT direction, traffic-flow constraints) and label them with the constraint.
- **Uplink / downlink discipline.** External traffic (Internet, partner) sits at the top edge of the canvas; private hosts sit at the bottom. The vertical axis encodes "outside" → "inside". Side-to-side layout is acceptable only when zones are peers (e.g., region-to-region peering).
- **Address detail level.** Include addressing only when the user asked for it (subnet CIDRs, host IPs). Otherwise stay at the device / zone level.

## Forbidden mixings

- Business process steps — those belong in `lark-uml:flowchart` / `lark-uml:swimlane`.
- Use case actors and boundaries — those belong in `lark-uml:usecase`.
- Application-layer call graphs — those belong in `lark-uml:architecture`.
- Database tables — those belong in `lark-uml:er`.

## Native node composition

Build the network topology out of these native whiteboard primitives. Do not express any part of the diagram as Mermaid, PlantUML, or SVG.

- **Zone** — native group / container, one per Internet boundary, DMZ, public subnet, private subnet, VPC, on-prem segment, or partner network. Header carries zone name and CIDR if provided.
- **Internet / external cloud** — native circle or cloud-shape supplied by the template; placed at the top of the canvas.
- **Router / gateway** — native rectangle with `Router` / `GW` label.
- **Firewall** — native rectangle with `FW` label (or the dashed / accent style the template already uses).
- **Switch / load balancer** — native rectangle with `SW` / `LB` label.
- **Host / VM / container** — native rectangle with `Host` / `VM` / `Pod` label.
- **Service endpoint** — native rectangle named after the service.
- **Link connector** — native `type: "connector"` whose `connector.from` / `connector.to` bind to two device node ids. Undirected for symmetric connectivity; arrowhead only when the constraint is asymmetric (NAT, allowed direction). Label only with network-layer facts (`TCP/443`, `VLAN 10`, peering name) — never with business-layer call descriptions.

When extending, duplicate the closest existing zone, device, or link and adapt it.
