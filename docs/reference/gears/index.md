---
slug: /reference/gears/overview
title: Catalog overview
---

# Catalog overview

This catalog details the modular processing units available in the **fluxrig** ecosystem. 

## Native built-ins
These gears are mature, optimized, and available for production use in the current core release.

*   **[Bento gear](bento.md)**: Universal I/O using the mature Bento engine (Pure Logic + Local I/O in the standard binary; institutional connectors via custom build).
*   **[ISO8583 I/O gear](io_iso8583.md)**: TCP socket with length-prefixed framing.
*   **[TCP I/O gear](io_tcp.md)**: Generic TCP I/O with delimiter-based framing (ingress/egress).
*   **[ISO8583 codec gear](codec_iso8583.md)**: Standardized parser for ISO8583 financial dialects.
*   **[Coat Check gear](coatcheck.md)**: Implements the "detached state" pattern for asynchronous correlation.
*   **[Wasm logic gear](wasm_logic.md)**: User-defined logic extensions via WebAssembly (Filter role available today; polyglot source gears are roadmap).
*   **[Conductor gear](conductor.md)**: Transaction switching, connection routing plus reply correlation over the valet ticket store; generalizes the Coat Check.

## Roadmap
These gears are currently in the **Architectural Proposal** or **Early Beta** phase.

*   **[Correlator gear [Roadmap]](correlator.md)**: Parallel shadow mastering / differential analysis (distinct from the Conductor's reply matching).
*   **[Modbus I/O gear [Roadmap]](io_modbus.md)**: Direct PLC integration (Modbus/TCP & RTU).
*   **[LoRaWAN I/O gear [Roadmap]](io_lorawan.md)**: Integrated LNS-less sensor connectivity.
*   **[Zigbee I/O gear [Roadmap]](io_zigbee.md)**: Industrial automation connectivity.
*   **[Network sniffer [Roadmap]](network_sniffer.md)**: Passive network tap (AF_PACKET).
*   **[Observability gears [Roadmap]](observability.md)**: Native connectors for external APM backends.

