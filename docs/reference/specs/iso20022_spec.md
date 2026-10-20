---
slug: /reference/protocol/iso20022
title: ISO 20022 spec
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# ISO 20022 spec

An ISO 20022 spec is the configuration that tells the [ISO 20022 codec gear](../gears/codec_iso20022.md) what a message carries. It belongs to the same family as the [ISO8583 SDL](iso8583_sdl.md). It has the same header, message catalog, value sets, per-message rules and load-time checks. Two things differ:

1.  **The wire layer.** The ISO 20022 schema already defines the layout of each message in XSD. The spec does not restate it. It names a wire source, a library that parses and checks the messages of one profile.
2.  **The key of a field.** An ISO 8583 field has a number (DE 37). An ISO 20022 field has the path of its XML element (`CdtTrfTxInf.PmtId.EndToEndId`).

The rest of the spec adds what a schema cannot express. It says what a field means, gives it an alias, and says which messages require it and which values it can carry. It also says how the codec handles a sensitive field.

A spec that connects an ISO 20022 message to an ISO 8583 message is not a protocol spec. It is a translation spec, which the [translate gear](../gears/translate.md) reads.

---

## Schema structure

A spec has a header, a message catalog, value sets, fields and checks. This is an excerpt of the reference spec `examples/specs/iso20022-rtp.yaml`:

```yaml
spec:
  id: "iso20022-rtp"
  version: "1.0.0"
  wire:
    source: "moov-rtp20022"        # the parser, and the profile: rtp

  messages:
    catalog:
      "pacs.008.001.08":
        name: "FI to FI customer credit transfer"
        flow: request
        pairs_with: "pacs.002.001.10"
        meaning: "Moves funds from a debtor account to a creditor account."
      "pacs.002.001.10":
        name: "FI to FI payment status report"
        flow: response
        pairs_with: "pacs.008.001.08"
        meaning: "The status of a credit transfer that a participant received."

  enums:
    tx_status:
      closed: true
      values:
        "ACTC": {name: "Accepted technical validation", category: approved}
        "ACWP": {name: "Accepted without posting", category: approved}
        "RCVD": {name: "Received", category: in_progress}
        "RJCT": {name: "Rejected", category: decline_hard}
    status_reason:
      closed: false
      values:
        "AM02": {name: "Amount exceeds the allowed maximum", category: decline_hard}
        "AM04": {name: "Insufficient funds", category: decline_soft}
        "AC03": {name: "Invalid creditor account", category: decline_hard}

  fields:
    "CdtTrfTxInf.PmtId.InstrId":
      messages:
        - message: "pacs.008.001.08"
          usage: mandatory
      alias: "instr_id"
      meaning: "Identifies the instruction between the instructing and the instructed participant."
    "TxInfAndSts.OrgnlInstrId":
      messages:
        - message: "pacs.002.001.10"
          usage: mandatory
      alias: "instr_id"
      meaning: "The instruction identifier of the credit transfer that this report answers."
    "CdtTrfTxInf.IntrBkSttlmAmt":
      alias: "settlement_amount"
      format:
        kind: amount
        currency_field: "CdtTrfTxInf.IntrBkSttlmAmt@Ccy"
    "CdtTrfTxInf.CdtrAcct.Id.Othr.Id":
      alias: "creditor_account"
      sensitivity: pii
    "TxInfAndSts.TxSts":
      alias: "tx_status"
      values_ref: "tx_status"

  checks:
    - message: ["pacs.008.001.08"]
      name: "Amount is positive"
      assert: 'field("CdtTrfTxInf.IntrBkSttlmAmt") > 0'
      severity: reject
```

| Section | Content |
| :--- | :--- |
| `spec.id`, `spec.version` | The name and version of the spec. The spec store keeps it under `id:v<version>`, for example `iso20022-rtp:v1.0.0`. |
| `spec.wire.source` | The wire source. It sets the profile. See [Where the wire layer comes from](#where-the-wire-layer-comes-from). |
| `messages.catalog` | The message definitions the spec describes, with their pairs. |
| `enums` | Value sets, for codes such as `TxSts` and status reasons. |
| `fields` | The fields, keyed by element path. |
| `checks` | Named assertions over several fields. |

---

## Where the wire layer comes from

The wire source is a library that knows the XSD of each message in a profile. It decodes and encodes the XML, and it checks the rules of the profile. The spec names it and does not restate it, as the ISO8583 SDL names a Moov base for its wire layer.

A profile is the ISO 20022 usage of one payment network: the message definitions the network accepts, their versions, its envelope, and its rules. Each network publishes its own, so each profile has its own wire source.

| Wire source | Profile | Network | Library | Message definitions |
| :--- | :--- | :--- | :--- | :--- |
| `moov-rtp20022` | `rtp` | The [RTP network](https://www.theclearinghouse.org/payment-systems/rtp) of The Clearing House | [moov-io/rtp20022](https://github.com/moov-io/rtp20022) [v0.12.1](https://github.com/moov-io/rtp20022/releases/tag/v0.12.1) | `pacs.008.001.08`, `pacs.002.001.10` |

The `rtp` profile carries one transaction for each message, in US dollars, between participants that a 9-digit routing number identifies. Each message travels in the envelope of the network, `<Message xmlns="urn:tch">`, with a business application header (`head.001`) before the document. The [ISO 20022 codec gear](../gears/codec_iso20022.md#the-rtp-profile) lists the rules of the profile and the elements the codec fills for it.

The library version is fixed for each fluxrig release. A certification run tests against the message versions and rules of that library version.

A message definition that is not in the table above, for example `pain.001` or `camt.053`, is not available. FedNow and Fedwire Funds use other message sets, envelopes and rules. This release has no wire source for them.

---

## Field definitions

The key of a field is the path of its element below the message root, with the XSD element names joined by dots. An attribute is a sibling of its element: the element name, `@`, and the attribute name, for example `CdtTrfTxInf.IntrBkSttlmAmt@Ccy`.

A field carries semantic attributes only. A wire attribute (an encoding, a length, a prefix) in a field definition stops the load: the wire source owns the layout.

| Attribute | Description |
| :--- | :--- |
| `alias` | A name for the field. Gears, translations and rules use the alias, not the path. |
| `meaning` | What the field says, for the generated documentation. |
| `scope` | `public` (default) or `private`. A private field is left out of the public documentation. |
| `messages` | The per-message matrix: in which messages the field is mandatory, optional or forbidden. |
| `values_ref` | The value set that the field takes its values from. |
| `format` | The semantic kind of the field, for example `amount` with its `currency_field`. |
| `sensitivity` | The data classification: `pii`, `none` and the other classes of the SDL. The ISO 20022 codec does not mask a sensitive field in its logs yet `[Roadmap]`. It keeps every field value out of its validation findings. |
| `log_mask` | Masks the field in logs `[Roadmap]` for ISO 20022. It defaults from `sensitivity`. |

### Aliases across messages

An alias names one field in one message. It can name another field in another message, if the two fields are never in the same message. The reference spec uses this for `instr_id`. In a `pacs.008` it names the instruction identifier. In a `pacs.002` it names the original instruction identifier, which the rail copies from the `pacs.008`. A conductor can then match the answer to the request on one alias. The loader refuses an alias that names two fields of the same message.

---

## The per-message matrix

`messages` on a field says what the field is in each message, as in the SDL. The key is `message` instead of `mti`:

```yaml
"CdtTrfTxInf.Dbtr.Nm":
  messages:
    - message: "pacs.008.001.08"
      usage: mandatory
  alias: "debtor_name"
```

`usage` is `mandatory`, `optional` or `forbidden`. A field with no entry for a message follows `validation_policy.unlisted`.

---

## Value sets

Value sets work as in the SDL. A set can be `closed` (a value outside it is a finding) or open. Each value can carry a `category`, a bounded classification. The category is a telemetry dimension: an approval-rate panel reads `category="approved"` the same way for ISO 8583 response codes and for ISO 20022 statuses.

The ISO 20022 spec uses the categories of the SDL and adds one, `in_progress`, for a status such as `RCVD`. Such a status reports that a participant received the transaction, not that it accepted or refused it.

---

## Checks

`checks` are named assertions over the fields of a message, as in the SDL. The `when` and `assert` expressions use the path form `field("<element path>")` and `present("<element path>")`, because a field has no number. `severity: reject` is for a finding that fails the message under `enforce`, and `severity: warn` is for a finding that is only recorded.

The loader checks that each check is well formed. The codec does not apply them to traffic yet `[Roadmap]`. Until it does, put a deployment rule, such as a ceiling on the amount, in a gear of the scenario. The simulated bank of the [bridge tutorial](../../tutorials/iso8583_iso20022_bridge.md) does this.

---

## What is checked when a spec loads

The loader refuses a spec, and names the symbol that is wrong, when:

- A field key is not an element path of the wire source for the messages that use it.
- A field definition has a wire attribute.
- An alias names two fields of the same message.
- A `values_ref` names a value set that the spec does not declare.
- A `messages` entry names a message that is not in the catalog.
- A message in the catalog is not a message definition of the wire source.
- A `pairs_with` is not symmetric.

---

## Enforcing the rules

The [ISO 20022 codec gear](../gears/codec_iso20022.md#validation) applies the spec to traffic with its `validation` key: `off`, `warn` or `enforce`. It checks the rules of the wire source first and the mandatory and forbidden fields of the spec next, on decode and on encode.

---

## Protocol reference [Roadmap]

For the ISO8583 SDL, `fluxrig spec doc` generates a [protocol reference](protocol_reference.md) from the spec. The command does not render an ISO 20022 spec yet.
