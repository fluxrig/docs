---
slug: /use_cases/payments
title: Payment use cases
---

# Payment use cases

**fluxrig** carries ISO 8583 traffic. It parses a message against a spec. It routes it. It correlates the reply. It can enforce what the spec says a message must carry. ISO 20022 is on the roadmap.

It is one rail, not the only one. A rail needs a codec only when its bytes need one. A rail that speaks JSON or XML over HTTP needs no gear at all. Serving it and calling it are configuration. The [acquisition gateway below](#example-iso-8583-acquisition-gateway) uses this.

Gears compose what a node does rather than fixing it. The same runtime serves a passive observer and a switch that answers.

The [ISO 8583 SDL](../reference/specs/iso8583_sdl.md) writes the dialect. `fluxrig spec doc` turns it into a [protocol reference](../reference/specs/protocol_reference.md) that certification and integration teams can read. The reference lists the messages, what each carries, and the rules that check a message. The engine loads the spec. The reference derives from that spec. Nobody maintains it beside the spec.

---

## Operational patterns

Institutional payment infrastructure covers a spectrum of operational patterns. **fluxrig** unifies these patterns into a single, cohesive architecture.

### Passive monitoring & observability
For environments where you must not touch the transaction cycle, the **Rack** operates as a non-intrusive observer.

*   **Shadow mirroring**: a wire can split. Production traffic reaches a parallel Rack. The live path does not wait on it.
*   **Differential integrity** `[Roadmap]`: Compare an existing switch's responses against new logic before a cutover. This needs the [Correlator gear](../reference/gears/correlator.md), which nobody built.

### Enrichment from outside the message

Some decisions need a fact the transaction does not carry. The Rack sits where it can obtain one, under a deadline it cannot exceed. It hands the authorizer an answer. The authorizer never calls anyone.

*   **[Mobile network signals](./mobile_network_signals.md)**: compare where a cardholder's handset is against where the merchant is, through the operator APIs standardised by GSMA Open Gateway.
*   **[A worked implementation](../tutorials/roaming_enrichment.md)**: the same case built end to end, with the enrichment, the degradation paths and the tests.

### Protocol orchestration gateway
As an orchestration gateway, **fluxrig** connects diverse systems, from cloud-native platforms to established financial networks.

*   **ISO 8583 normalization**: The **[Codec Gear (ISO 8583)](../reference/gears/codec_iso8583.md)** deterministically translates dense binary bitmaps into structured JSON/CBOR for modern APIs.
*   **Sovereign context parking**: The **[Coat Check gear](../reference/gears/coatcheck.md)** parks correlation context (and, optionally, specific non-PAN fields) at the infrastructure boundary. It restores it on the reply. This keeps internal context off external legs. This is *parking*, not tokenization. Surrogate substitution (replacing a PAN with a vault-backed token) is a separate, roadmap gear. Note a switch must still send the PAN to the scheme to authorize.

### Active orchestration and switching
In this pattern, **fluxrig** acts as the deterministic engine of the payment flow. It routes transactions between originators (ATMs/POS) and processors.

*   **Switching and reply correlation**: The **[Conductor gear](../reference/gears/conductor.md)** routes each request across upstream connections. It matches the reply over a "valet" ticket store (local by default). It generalizes the Coat Check for the switching path.
*   **Stand-in processing (STIP)**: Compose a deterministic **[Logic Gear (STIP)](../reference/gears/wasm_logic.md)**. The Rack authorizes transactions locally when the upstream host is unreachable. This sustains availability during host outages.
*   **Request/response matching**: The **[Correlator Gear](../reference/gears/correlator.md)** `[Roadmap]` (differential analysis) is a distinct capability from the Conductor's reply matching. Use it for reconciliation against the immutable CBOR archives.

---

## Example: ISO 8583 acquisition gateway

This common scenario illustrates a modern transition. It receives standard Webhook/REST calls from a terminal. It orchestrates them into a financial network.

<LikeC4 project="payments-gateway" view="flow" height={480} />

**Reading the diagram**: The interactive view traces one transaction as a numbered walkthrough. The request travels out (steps 1-5). The authorization returns (steps 6-9). Click any gear to focus it. Pan or zoom for detail. Steps 5 and 6 are the two external connections, each a **single bidirectional socket**. Every internal step is one **unidirectional wire**. Request and response wires are equals: the step order, not the arrow direction, is what distinguishes the legs. Each socket terminates at the gear that owns it, which bridges it onto those one-way wires.

### Step-by-step processing
1.  **REST ingress**: The POS terminal sends a JSON payload. The **[Bento Gear](../reference/gears/bento.md)** acts as the HTTP server. It maps the request into an internal `fluxMsg`.
2.  **Business logic**: The **Logic Gear** checks the transaction (e.g., checking minimum amount). It attaches routing metadata.
3.  **Protocol encoding**: A **[Codec Gear](../reference/gears/codec_iso8583.md)** instance (`direction: encode`) packs the semantic message into the precise ISO 8583 binary bitmap. The external processor expects this bitmap. The codec translates. It does not own a network connection.
4.  **Financial egress**: The **[I/O Gear](../reference/gears/io_iso8583.md)** (client mode) owns the persistent, length-framed TCP connection to the Acquirer. It consumes the bitmap on its `in` port. It writes it to the socket.
5.  **Authorization response**: The Acquirer answers over the **same TCP connection**. The socket is bidirectional. The I/O Gear bridges it back into the one-way world. It emits the response on its `out` port.
6.  **Response decoding**: A second Codec instance (`direction: decode`) unpacks the response bitmap into structured fields.
7.  **REST egress**: The Bento Gear correlates the response to the still-open HTTP request. It answers the POS terminal.

> [!NOTE]
> **Wire paths. Do not mirror them.** The response path deliberately skips the validation Logic Gear. Each direction contains exactly the gears wired into it. The acquirer's answer needs no request-side validation. When you need response-side processing (response-code mapping, journaling the authorization result, reversal bookkeeping), wire a dedicated gear into the response pair. The request path never implies it.

> **Scaling this pattern**. A production switch needs more than one uplink: multiple acquirer or scheme connections, load balancing, failover, and reply correlation across all of them. That is the role of the **Conductor Gear**. See the [payment switch tutorial](../tutorials/payment_switch_conductor.md) for the full multi-region design.

---

## Testing and validation

The same rig shows whether an integration is right before it carries money.

**Answer the traffic against the spec.** Set the codec's `validation` to `warn`. The setup checks every message without rejecting any. Disagreements between a certification document and what is actually on the wire arrive as violations on the message rather than as a failed transaction. `enforce` comes after that, once the spec and the traffic agree.

**Replace the counterparty.** The
[ISO 8583 utility](../reference/tools/iso8583_tool.md) generates load or echoes
it, so a scenario runs against a simulated scheme or host. A scenario records
which participants it simulates (`peer_played_by`). The generated topology
shows it. A reader deciding whether a suite proves anything can see which
side was real.

**Run the whole thing.** The
[Robot Framework suite](../tutorials/iso8583_robot_suite.md) assembles those
pieces into a rig. It starts the Mixer and the Racks. It drives real protocol
traffic through them. It asserts on the result. The harness is not specific to
ISO 8583.

Before a production cutover, this is the loop worth running against the
counterparty's own certification cases.

## Institutional permissive freedom

**fluxrig** is a platform for institutional builders provided under the **Apache 2.0** license.

In an industry dominated by proprietary "Black Box" switches and restrictive licenses, **fluxrig** offers commercial agility. This model ensures you can build proprietary, mission-critical logic without the risk of legal contamination or forced disclosure of your commercial intellectual property.
