---
slug: /use_cases/payments
title: Payment use cases
---

# Payment use cases

**fluxrig** carries ISO 8583 traffic: it parses a message against a spec, routes
it, correlates the reply, and can enforce what the spec says a message must
carry. ISO 20022 is on the roadmap.

It is one rail, not the only one. A rail needs a codec only when its bytes need
one, and a rail that speaks JSON or XML over HTTP needs no gear at all: serving
it and calling it are configuration, which is what the
[acquisition gateway below](#example-iso-8583-acquisition-gateway) is made of.

What a node does is composed from gears rather than fixed, so the same runtime
serves a passive observer and a switch that answers.

The dialect is written in the [ISO 8583 SDL](../reference/specs/iso8583_sdl.md),
and `fluxrig spec doc` turns it into a
[protocol reference](../reference/specs/protocol_reference.md) that certification
and integration teams can read: the messages, what each carries, and the rules a
message is checked against, derived from the spec the engine loads rather than
maintained beside it.

---

## Operational patterns

Institutional payment infrastructure is categorized across a spectrum of operational patterns. **fluxrig** unifies these patterns into a single, cohesive architecture.

### Passive monitoring & observability
For environments where touching the transaction cycle is restricted, the **Rack** operates as a non-intrusive observer.

*   **Shadow mirroring**: a wire can fan out, so production traffic reaches a parallel Rack without the live path waiting on it.
*   **Differential Integrity** `[Roadmap]`: comparing an existing switch's responses against new logic before a cutover. This needs the [Correlator gear](../reference/gears/correlator.md), which is not built.

### Enrichment from outside the message

Some decisions need a fact the transaction does not carry. The Rack sits where it can obtain one, under a deadline it cannot exceed, and hand the authorizer an answer without the authorizer ever calling anyone.

*   **[Mobile network signals](./mobile_network_signals.md)**: comparing where a cardholder's handset is against where the merchant is, through the operator APIs standardised by GSMA Open Gateway.
*   **[A worked implementation](../tutorials/roaming_enrichment.md)**: the same case built end to end, with the enrichment, the degradation paths and the tests.

### Protocol orchestration gateway
As a orchestration gateway, **fluxrig** connects diverse systems, from cloud-native platforms to established financial networks.

*   **ISO 8583 Normalization**: The **[Codec Gear (ISO 8583)](../reference/gears/codec_iso8583.md)** performs deterministic translation of dense binary bitmaps into structured JSON/CBOR required by modern APIs.
*   **Sovereign Context Parking**: The **[Coat Check gear](../reference/gears/coatcheck.md)** parks correlation context (and, optionally, specific non-PAN fields) at the infrastructure boundary and restores it on the reply, keeping internal context off external legs. This is *parking*, not tokenization, surrogate substitution (replacing a PAN with a vault-backed token) is a separate, roadmap gear. Note a switch must still send the PAN to the scheme to authorize.

### Active orchestration and switching
In this pattern, **fluxrig** acts as the deterministic engine of the payment flow, routing transactions between originators (ATMs/POS) and processors.

*   **Switching and reply correlation**: The **[Conductor gear](../reference/gears/conductor.md)** routes each request across upstream connections and matches the reply over a "valet" ticket store (local by default). It generalizes the Coat Check for the switching path.
*   **Stand-in Processing (STIP)**: Composing a deterministic **[Logic Gear (STIP)](../reference/gears/wasm_logic.md)** allows the Rack to authorize transactions locally when the upstream host is unreachable, sustaining availability during host outages.
*   **Request/Response Matching**: The **[Correlator Gear](../reference/gears/correlator.md)** `[Roadmap]` (differential analysis) is a distinct capability from the Conductor's reply matching; use it for reconciliation against the immutable CBOR archives.

---

## Example: ISO 8583 acquisition gateway

This common scenario illustrates a modern transition: receiving standard Webhook/REST calls from a terminal and orchestrating them into a financial network.

<LikeC4 project="payments-gateway" view="flow" height={480} />

**Reading the diagram**: the interactive view traces one transaction as a numbered walkthrough: the request travels out (steps 1-5) and the authorization returns (steps 6-9). Click any gear to focus it, or pan/zoom for detail. Steps 5 and 6 are the two external connections, each a **single bidirectional socket**; every internal step is one **unidirectional wire**. Request and response wires are equals: the step order, not the arrow direction, is what distinguishes the legs. Each socket terminates at the gear that owns it, which bridges it onto those one-way wires.

### Step-by-step processing
1.  **REST Ingress**: The POS terminal sends a JSON payload. The **[Bento Gear](../reference/gears/bento.md)** acts as the HTTP server, mapping the request into an internal `fluxMsg`.
2.  **Business Logic**: The **Logic Gear** validates the transaction (e.g., checking minimum amount) and attaches routing metadata.
3.  **Protocol Encoding**: A **[Codec Gear](../reference/gears/codec_iso8583.md)** instance (`direction: encode`) packs the semantic message into the precise ISO 8583 binary bitmap expected by the external processor. The codec translates; it does not own a network connection.
4.  **Financial Egress**: The **[I/O Gear](../reference/gears/io_iso8583.md)** (client mode) owns the persistent, length-framed TCP connection to the Acquirer. It consumes the bitmap on its `in` port and writes it to the socket.
5.  **Authorization Response**: The Acquirer answers over the **same TCP connection**: the socket is bidirectional, and the I/O Gear bridges it back into the one-way world by emitting the response on its `out` port.
6.  **Response Decoding**: A second Codec instance (`direction: decode`) unpacks the response bitmap into structured fields.
7.  **REST Egress**: The Bento Gear correlates the response to the still-open HTTP request and answers the POS terminal.

> [!NOTE]
> **Paths are wired, not mirrored.** The response path deliberately skips the validation Logic Gear: each direction contains exactly the gears wired into it, and the acquirer's answer needs no request-side validation. When response-side processing is required (response-code mapping, journaling the authorization result, reversal bookkeeping), it is added by wiring a dedicated gear into the response pair, never implied by the request path.

> **Scaling this pattern**: A production switch needs more than one uplink: multiple acquirer or scheme connections, load balancing, failover, and reply correlation across all of them. That is the role of the **Conductor Gear**: see the [payment switch tutorial](../tutorials/payment_switch_conductor.md) for the full multi-region design.

---

## Testing and validation

The same rig is how you find out whether an integration is right before it
carries money.

**Answer the traffic against the spec.** Set the codec's `validation` to `warn`
and every message is checked without any being rejected, so the disagreements
between a certification document and what is actually on the wire arrive as
violations on the message rather than as a failed transaction. `enforce` comes
after that, once the spec and the traffic agree.

**Stand in for the counterparty.** The
[ISO 8583 utility](../reference/tools/iso8583_tool.md) generates load or echoes
it, so a scenario runs against a simulated scheme or host. A scenario records
which participants are simulated (`peer_played_by`) and the generated topology
shows it, so a reader deciding whether a suite proves anything can see which
side was real.

**Run the whole thing.** The
[Robot Framework suite](../tutorials/iso8583_robot_suite.md) assembles those
pieces into a rig that starts the Mixer and the Racks, drives real protocol
traffic through them and asserts on the result. The harness is not specific to
ISO 8583.

Before a production cutover, this is the loop worth running against the
counterparty's own certification cases.

## Institutional permissive freedom

**fluxrig** is a platform for institutional builders provided under the **Apache 2.0** license.

In an industry dominated by proprietary "Black Box" switches and restrictive licenses, **fluxrig** offers true commercial agility. Our model ensures you can build proprietary, mission-critical logic without the risk of legal contamination or forced disclosure of your commercial intellectual property.
