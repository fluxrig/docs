---
slug: /use_cases
title: fluxrig in action
---

# fluxrig in action

**fluxrig** treats every data stream the way a studio treats a signal. The setup routes it, transforms it and observes it, whatever it carries. The rig is the same in each domain below. Only the gears and the spec change. You patch the gears into the rig. You read the traffic against the spec.

That holds for a flow you design now and for one already in production. Nothing here requires the systems on either end to change.

## Payments
*Financial message switching*

Orchestrate payment flows between POS terminals and banking cores. The rig handles **ISO 8583** translation against a spec that describes the dialect. It can enforce what that spec says a message must carry. It parks correlation context at the edge. It switches and correlates replies through the **Conductor**. It persists durable audit metadata rather than clear-text cardholder data.

ISO 8583 is one rail, not the only one. A rail that speaks JSON or XML over HTTP needs no gear at all. Serving it and calling it are configuration. The same rig validates an integration before it carries money. It answers live traffic against the spec. It replaces the counterparty.

- **[Payment use cases](./payments.md)**: the operational patterns, a worked acquisition gateway, and how to test an integration before cutover
- **[Mobile network signals](./mobile_network_signals.md)**: ask a mobile operator what a payment message cannot know, through the GSMA Open Gateway APIs
- Reference: the **[ISO 8583 SDL](../reference/specs/iso8583_sdl.md)**, the **[protocol reference](../reference/specs/protocol_reference.md)** rendered from a spec, and the **[ISO 8583 utility](../reference/tools/iso8583_tool.md)**

## Industrial
*OT/IT bridging*

Bridge factory floor equipment and the systems that report on it. A Rack sits at the edge of the plant. It normalises what the equipment already publishes into events the rest of the business can read. It keeps doing this when the link to the centre is down.

The **Modbus** and **RS-485** gears are on the roadmap. Today the path runs over
the TCP I/O gear and Bento, behind whatever already terminates the fieldbus.

- **[Industrial use cases](./industrial.md)**: the Unified Namespace strategy, deployment patterns, and what runs at the plant edge
- Reference: the **[TCP I/O gear](../reference/gears/io_tcp.md)** and the **[Bento gear](../reference/gears/bento.md)**

## Internet of Things (IoT)
*Distributed edge intelligence*

Filter and correlate at the edge so the backhaul carries decisions rather than
noise. That is what a Rack is for. It keeps processing while the Mixer is
unreachable. This is the difference between a quiet link and a lost fleet.

Radio protocols reach a Rack over the link their network server already speaks,
TCP, HTTP or WebSocket. There is no **LoRaWAN**, **Zigbee** or **LTE** gear. The Rack sits behind one.

- **[IoT use cases](./iot.md)**: deployment patterns, edge filtering, and what the Rack does when the network is not there
- Reference: **[Operating fluxrig](../reference/operations.md)** for what a Rack does on its own
