---
slug: /use_cases
title: fluxrig in action
---

# fluxrig in action

**fluxrig** treats every data stream the way a studio treats a signal: something
to be routed, transformed and observed, whatever it happens to carry. The rig is
the same in each domain below; what changes is the gears patched into it and the
spec the traffic is read against.

That holds for a flow being designed now as much as for one already in
production. Nothing here requires the systems on either end to change.

## Payments
*Financial message switching*

Orchestrate payment flows between POS terminals and banking cores. The rig
handles **ISO 8583** translation against a spec that describes the dialect, and
can enforce what that spec says a message must carry. It parks correlation
context at the edge, switches and correlates replies through the **Conductor**,
and persists durable audit metadata rather than clear-text cardholder data.

ISO 8583 is one rail, not the only one: a rail that speaks JSON or XML over HTTP
needs no gear at all, because serving it and calling it are configuration. And
the same rig is how an integration is validated before it carries money, by
answering live traffic against the spec and standing in for the counterparty.

- **[Payment use cases](./payments.md)**: the operational patterns, a worked acquisition gateway, and how an integration is tested before cutover
- **[Mobile network signals](./mobile_network_signals.md)**: asking a mobile operator what a payment message cannot know, through the GSMA Open Gateway APIs
- Reference: the **[ISO 8583 SDL](../reference/specs/iso8583_sdl.md)**, the **[protocol reference](../reference/specs/protocol_reference.md)** rendered from a spec, and the **[ISO 8583 utility](../reference/tools/iso8583_tool.md)**

## Industrial
*OT/IT bridging*

Bridge factory floor equipment and the systems that report on it. A Rack sits at
the edge of the plant, normalises what the equipment already publishes into
events the rest of the business can read, and keeps doing it when the link to
the centre is down.

The **Modbus** and **RS-485** gears are on the roadmap. Today the path runs over
the TCP I/O gear and Bento, behind whatever already terminates the fieldbus.

- **[Industrial use cases](./industrial.md)**: the Unified Namespace strategy, deployment patterns, and what runs at the plant edge
- Reference: the **[TCP I/O gear](../reference/gears/io_tcp.md)** and the **[Bento gear](../reference/gears/bento.md)**

## Internet of Things (IoT)
*Distributed edge intelligence*

Filter and correlate at the edge so the backhaul carries decisions rather than
noise. That is what a Rack is for: it keeps processing while the Mixer is
unreachable, which is the difference between a quiet link and a lost fleet.

Radio protocols reach a Rack over the link their network server already speaks,
TCP, HTTP or WebSocket. There is no **LoRaWAN**, **Zigbee** or **LTE** gear, and
none is needed to sit behind one.

- **[IoT use cases](./iot.md)**: deployment patterns, edge filtering, and what the Rack does when the network is not there
- Reference: **[Operating fluxrig](../reference/operations.md)** for what a Rack does on its own
