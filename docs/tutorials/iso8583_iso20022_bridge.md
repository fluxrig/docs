---
slug: /tutorials/iso8583_iso20022_bridge
title: Bridge ISO 8583 and ISO 20022
---

# Bridge ISO 8583 and ISO 20022

This tutorial builds a bridge between a card terminal network and a real-time payment rail. A terminal sends an ISO 8583 financial request. The bridge sends an ISO 20022 credit transfer (`pacs.008`) to the rail. The rail answers with a payment status report (`pacs.002`). The bridge returns an ISO 8583 response to the terminal.

The worked example is a bill payment. A customer pays a bill at a payment terminal. The operator of the terminal network pushes the funds to the bank account of the biller over the rail, and the terminal shows the result. The terminal does not change: it speaks ISO 8583 as it always did.

The tutorial uses two new gears, the [ISO 20022 codec gear](../reference/gears/codec_iso20022.md) (`codec_iso20022`) and the [translate gear](../reference/gears/translate.md) (`translate`). Every other gear is one you already know: `io_iso8583`, `codec_iso8583`, `conductor`, `io_tcp` and `bento`.

> [!NOTE]
> **Scope of the ISO 20022 side.** The rail of this tutorial is the [RTP network](https://www.theclearinghouse.org/payment-systems/rtp) of The Clearing House. The ISO 20022 spec uses the wire source `moov-rtp20022`, which is the `rtp` profile of that network. This profile carries one transaction for each message, in US dollars, between participants that a 9-digit routing number identifies. It covers `pacs.008.001.08` and `pacs.002.001.10`. FedNow and Fedwire Funds use other profiles, which this release does not have. See [Where the wire layer comes from](../reference/specs/iso20022_spec.md#where-the-wire-layer-comes-from).
>
> The files of this tutorial are in `examples/`: the scenario in `examples/scenarios/billpay_rtp.yaml`, the specs in `examples/specs/`, and the records of the billers in `examples/lookups/billers.yaml`.

---

## Why a bridge needs more than a codec

ISO 8583 and ISO 20022 do not carry the same data. An ISO 8583 financial request identifies a card, an amount, a terminal and a merchant. An ISO 20022 credit transfer identifies two parties, two accounts and two banks. Most of that is not in the ISO 8583 message.

A bridge must therefore do three jobs. A different part of fluxrig does each one:

1. **Convert the format.** The codecs do this. `codec_iso8583` reads and writes ISO 8583 fields. `codec_iso20022` reads and writes XML. Each codec loads a **protocol spec**, which gives the fields their names (aliases), their meaning and their rules.
2. **Translate between the two protocols, and supply the missing data.** The `translate` gear does this. It loads a **translation spec**: a separate file that names the two protocol specs. The translation spec says which alias on one side becomes which alias on the other. The name, the bank and the account of the biller come from a directory that the translation spec names.
3. **Match the answer to the question.** The rail answers on its own connection, with its own identifiers. The `conductor` matches each `pacs.002` to the terminal that is waiting for it.

The three kinds of file are independent. You can update a protocol spec without a change to the translation, and change the translation without a change to either protocol spec.

The card number stays on the card leg. The PAN is not a field of the ISO 20022 spec, so `codec_iso20022` cannot put it in the `pacs.008`. The payer on the rail is the settlement account of the operator, not the card.

---

## The topology

Two Racks run the example. `rack-switch` is the bridge. `rack-rail` stands in for the rail, so you can run the tutorial on one machine. In production, `rack-rail` is not there: `rail` connects to the network of the rail operator.

<LikeC4 project="billpay-rtp" view="index" height={560} />

`fluxrig scenario viz` generates this diagram from the [scenario](#the-scenario) below, so the diagram and the wiring cannot drift apart. The diagram is interactive: hover a node for its description, drag to pan, scroll to zoom, and open a node to see its neighbours. Each internal arrow is one wire from the `wires:` section. The connection between `rail` and `rail-in` is one bidirectional TCP socket.

This view expands `rack-switch` down to its gears:

<LikeC4 project="billpay-rtp" view="of_rack_switch" height={560} />

The request path:

1. `terminals` accepts the `0200` from the terminal.
2. `decode-8583` decodes it. The fields get their aliases from the ISO 8583 spec: `pan`, `txn_amount`, `rrn`, `merchant_id` and the others.
3. `to-rail` applies the translation `payment_request`. It adds the ISO 20022 fields of a `pacs.008` to the message, generates the instruction identifier, and records that the target message is `pacs.008.001.08`.
4. `conductor` parks a ticket for the request under the instruction identifier, and sends the message on.
5. `encode-pacs008` builds the XML. It puts the XML in `RawPayload`.
6. `rail` writes the XML to the rail.

Each wire carries a `fluxMsg`. An encode does not replace the message with XML: it writes the XML into `RawPayload` and keeps the fields in `Data`. The conductor sees only structured messages, as on every path in fluxrig: it parks before the encode, and it matches the answer after the decode.

The answer path:

1. `rail` receives the `pacs.002`.
2. `decode-pacs002` decodes it.
3. `conductor` finds the ticket, restores the fields it parked, and sends the answer to `to-card`.
4. `to-card` applies the translation `payment_status`. It turns the status of the rail into an ISO 8583 response code and sets the MTI to `0210`.
5. `encode-8583` builds the `0210`, and `terminals` sends it to the terminal that asked.

---

## The protocol specs

Each codec loads a protocol spec. The ISO 8583 side uses the reference spec `iso8583-v87-ascii.yaml` from `examples/specs/`. The ISO 20022 side uses the reference spec `iso20022-rtp.yaml`, from the same directory.

The two specs use the same format. The difference is the key of a field. An ISO 8583 field has a number (DE 37). An ISO 20022 field has the path of its XML element. Both give the field an alias, and the translation uses only the aliases.

### How `codec_iso20022` represents a message

The codec decodes an ISO 20022 message into `fluxMsg.Data`. The raw path of a field is the chain of XML element names below the message root, joined by dots, under the `iso20022` key. The codec uses the element names of the ISO 20022 schema. Thus a field has the same path in every tool that reads the schema. An attribute is a sibling of its element: the element name, `@`, and the attribute name.

When the spec gives a field an alias, the codec writes the value two times: under the alias, and under the raw path. On encode, the alias has priority. `codec_iso8583` does the same.

These are the aliases of `iso20022-rtp.yaml` that this tutorial uses:

| Alias | Element path | Message |
| :--- | :--- | :--- |
| `sender_participant` | `AppHdr.Fr.FIId.FinInstnId.ClrSysMmbId.MmbId` | both |
| `receiver_participant` | `AppHdr.To.FIId.FinInstnId.ClrSysMmbId.MmbId` | both |
| `msg_id` | `GrpHdr.MsgId` | both |
| `creation_time` | `GrpHdr.CreDtTm` | both |
| `instr_id` | `CdtTrfTxInf.PmtId.InstrId` | `pacs.008` |
| `instr_id` | `TxInfAndSts.OrgnlInstrId` | `pacs.002` |
| `end_to_end_id` | `CdtTrfTxInf.PmtId.EndToEndId` | `pacs.008` |
| `tx_id` | `CdtTrfTxInf.PmtId.TxId` | `pacs.008` |
| `local_instrument` | `CdtTrfTxInf.PmtTpInf.LclInstrm.Prtry` | `pacs.008` |
| `category_purpose` | `CdtTrfTxInf.PmtTpInf.CtgyPurp.Prtry` | `pacs.008` |
| `settlement_amount` | `CdtTrfTxInf.IntrBkSttlmAmt` | `pacs.008` |
| `settlement_currency` | `CdtTrfTxInf.IntrBkSttlmAmt@Ccy` | `pacs.008` |
| `debtor_name` | `CdtTrfTxInf.Dbtr.Nm` | `pacs.008` |
| `debtor_account` | `CdtTrfTxInf.DbtrAcct.Id.Othr.Id` | `pacs.008` |
| `debtor_agent` | `CdtTrfTxInf.DbtrAgt.FinInstnId.ClrSysMmbId.MmbId` | `pacs.008` |
| `creditor_name` | `CdtTrfTxInf.Cdtr.Nm` | `pacs.008` |
| `creditor_account` | `CdtTrfTxInf.CdtrAcct.Id.Othr.Id` | `pacs.008` |
| `creditor_agent` | `CdtTrfTxInf.CdtrAgt.FinInstnId.ClrSysMmbId.MmbId` | `pacs.008` |
| `remittance_info` | `CdtTrfTxInf.RmtInf.Ustrd` | `pacs.008` |
| `instructing_agent` | `CdtTrfTxInf.InstgAgt.FinInstnId.ClrSysMmbId.MmbId` | `pacs.008` |
| `instructing_agent` | `TxInfAndSts.InstgAgt.FinInstnId.ClrSysMmbId.MmbId` | `pacs.002` |
| `instructed_agent` | `CdtTrfTxInf.InstdAgt.FinInstnId.ClrSysMmbId.MmbId` | `pacs.008` |
| `instructed_agent` | `TxInfAndSts.InstdAgt.FinInstnId.ClrSysMmbId.MmbId` | `pacs.002` |
| `orig_msg_id` | `OrgnlGrpInfAndSts.OrgnlMsgId` | `pacs.002` |
| `orig_msg_name` | `OrgnlGrpInfAndSts.OrgnlMsgNmId` | `pacs.002` |
| `orig_creation_time` | `OrgnlGrpInfAndSts.OrgnlCreDtTm` | `pacs.002` |
| `tx_status` | `TxInfAndSts.TxSts` | `pacs.002` |
| `status_reason` | `TxInfAndSts.StsRsnInf.Rsn.Cd` | `pacs.002` |

An alias can name two fields, but only if the two fields are never in the same message. `instr_id` is the important case. In a `pacs.008` it is the instruction identifier. In a `pacs.002` it is the original instruction identifier, which the rail copies from the `pacs.008`. The conductor uses `instr_id` to match the answer to the request, and it does not need a translation for this.

The fields under `AppHdr.` are in the business application header of the envelope, which the network puts around each message. The other fields are in the document.

The spec classifies `debtor_account` and `creditor_account` as personal data (`sensitivity: pii`). The codec does not mask them in its logs yet `[Roadmap]`.

### What the profile fills in

On encode, `codec_iso20022` writes two kinds of elements. No spec and no translation sets them.

The identity of the message. The codec always writes these elements, and replaces a value that the message already has:

- `GrpHdr.MsgId`, from the participant identifier and the current date, in the format of the profile.
- `GrpHdr.CreDtTm`.
- In the header: the sender (`sender_participant`), `BizMsgIdr`, `MsgDefIdr` and `CreDt`.

Thus the `pacs.002` that answers a `pacs.008` has its own identifiers, although `ack` builds it from the fields of the `pacs.008`.

The elements that have only one valid value in the profile. The codec writes these only when the message does not have them. For a `pacs.008`:

- `GrpHdr.IntrBkSttlmDt`, `GrpHdr.NbOfTxs` (`1`) and `GrpHdr.TtlIntrBkSttlmAmt` (the amount of the one transaction).
- `GrpHdr.SttlmInf.SttlmMtd` (`CLRG`) and `GrpHdr.SttlmInf.ClrSys.Cd` (`TCH`).
- `CdtTrfTxInf.PmtTpInf.SvcLvl.Cd` (`SDVA`) and `CdtTrfTxInf.ChrgBr` (`SLEV`).
- `CdtTrfTxInf.InstgAgt` and `CdtTrfTxInf.InstdAgt`, copied from `CdtTrfTxInf.DbtrAgt` and `CdtTrfTxInf.CdtrAgt`.

For a `pacs.002`, it writes `TxInfAndSts.AccptncDtTm` and `OrgnlGrpInfAndSts.OrgnlNbOfTxs`.

The codec does not write the receiver. The translation sets `receiver_participant`.

The codec reads the participant identifier from the `iso20022.participant_id` metadata, which `translate` writes from the translation spec. The instruction identifier is not on this list: `to-rail` generates it, before the conductor, so that the conductor can park the ticket under it.

---

## The translation spec

A translation spec says which alias of one protocol spec becomes which alias of another. It is data, not code. It has its own name and version, and you review it, version it and store it like a protocol spec.

Save this file as `specs/billpay-rtp.translation.yaml`:

```yaml
spec:
  kind: "translation"
  id: "billpay-rtp"
  version: "1.0.0"

  # The protocol specs that this translation connects.
  specs:
    card: "iso8583-v87-ascii:v3.0.0"
    rail: "iso20022-rtp:v1.0.0"

  # The participant identifier of the operator on the rail. It goes into the
  # identifiers that this translation and the codec generate.
  participant_id: "99000000101"

  lookups:
    # The directory of billers. The terminal sends the biller code in DE 42.
    billers:
      key: "card:merchant_id"
      fields: ["name", "routing", "account"]
      source: "billers:v1"

  tables:
    # TxSts, or TxSts/reason code, to an ISO 8583 response code (DE 39).
    tx_status_to_resp_code:
      "ACTC": "00"          # accepted
      "ACWP": "00"          # accepted without posting
      "RCVD": "09"          # received, in progress
      "RJCT/AM02": "13"     # amount over the limit
      "RJCT/AM04": "51"     # insufficient funds
      "RJCT/AC03": "14"     # invalid creditor account
      "RJCT/AC04": "14"     # closed account
      "RJCT/AC06": "14"     # blocked account
      default: "05"         # any other rejection: do not honour

  translations:
    payment_request:
      from: "card:0200"
      to: "rail:pacs.008.001.08"
      bindings:
        - { to: "instr_id", generate: "instr_id" }
        - { to: "end_to_end_id", from: "rrn" }
        - { to: "tx_id", from: "rrn" }
        - { to: "local_instrument", value: "STANDARD" }
        - { to: "category_purpose", value: "CONSUMER" }
        - { to: "settlement_amount", from: "txn_amount", format: "amount", currency: "txn_currency" }
        # The payer: the settlement account of the operator.
        - { to: "debtor_name", value: "Example Payments Operator" }
        - { to: "debtor_account", value: "5500001111" }
        - { to: "debtor_agent", value: "990000001" }
        # The payee: the biller, from the directory.
        - { to: "creditor_name", lookup: "billers.name" }
        - { to: "creditor_account", lookup: "billers.account" }
        - { to: "creditor_agent", lookup: "billers.routing" }
        - { to: "remittance_info", from: "terminal_id" }
        # The participant of the bank of the biller, on the rail.
        - { to: "receiver_participant", value: "99000000201" }

    payment_status:
      from: "rail:pacs.002.001.10"
      to: "card:0210"
      bindings:
        - { to: "resp_code", from: ["tx_status", "status_reason"], table: "tx_status_to_resp_code" }
```

The translation names the records of the billers by a store reference, `billers:v1`. Save them as `lookups/billers.yaml`. You upload them to the Mixer under that name and tag in [Run it](#run-it):

```yaml
"BILLER000000001":
  name: "Example Water Utility"
  routing: "990000002"
  account: "7700123456"
"BILLER000000002":
  name: "Example Power Company"
  routing: "990000003"
  account: "7700987654"
```

The routing numbers and accounts in these files are examples. They identify no real bank.

### How to read a translation

- **`from` and `to` name messages, not only protocols.** `card:0200` is the message `0200` of the spec that the translation calls `card`. The translator sets the identity of the target message itself: the MTI `0210` for `payment_status`, and the message `pacs.008.001.08` for `payment_request`. No binding sets them.
- **Each binding names an alias on each side.** `to` is an alias of the target spec. `from` is an alias of the source spec. A binding never names a DE number or an XML path. The same translation thus works for every ISO 8583 dialect whose spec declares `rrn`, `txn_amount` and `merchant_id`.
- **The binding kinds.** `from` reads a source alias. `value` is a constant. `lookup` reads a field of the record of the biller. `generate` makes a value: `instr_id` makes an instruction identifier in the format of the target profile, from `participant_id`. `now`, `today` and `uuid` make the obvious values. `table` translates a value. With a list of source aliases, the translator joins their values with `/` before it looks them up. `format: amount` converts an ISO 8583 amount in minor units, with its numeric currency at `currency`, to an ISO 20022 decimal amount and its alphabetic currency. The conversion works in both directions.
- **The translator adds fields and removes none.** The message keeps its ISO 8583 fields after `to-rail`. Only what the target codec encodes goes on the wire, and that is the fields of the target spec.
- **The records stay outside the spec.** The translation spec names the lookup, its key and its source. The records are a separate artefact, because they change more often than the translation. A new version of the records does not change the translation.
- **Every reference is a store reference.** The translation names its protocol specs and its records as `name:tag`. The Mixer sends all of them to the Racks with the scenario, so a Rack needs no file.

When the translation spec loads, fluxrig checks it against the two protocol specs. A binding that names an alias that the spec does not declare stops the load and names the alias. So does a message that is not in the catalog of its spec. The gear checks on each message, not at load, a field that the target message requires and that no binding writes. Another gear can supply such a field: the conductor restores the parked ISO 8583 fields of the answer before `to-card`.

---

## The scenario

Save this as `scenarios/billpay_rtp.yaml`:

```yaml
meta:
  name: "billpay_rtp"
  version: "1.0.0"

racks:
  - name: "rack-switch"
    labels:
      role: "the bridge"
  - name: "rack-rail"
    labels:
      role: "a simulated RTP rail; in production, the network of the rail operator"

gears:
  # ─────────────── rack-switch: the bridge ───────────────
  - name: "terminals"
    type: "io_iso8583"
    deploy: "rack-switch"
    labels:
      peer: "Payment terminals"
      peer_played_by: "iso8583-tool"
    config:
      mode: "server"
      bind: ":8583"
      frame_length_size: 2
      frame_length_endian: "big"
      encoding: "ascii"

  - name: "decode-8583"
    type: "codec_iso8583"
    deploy: "rack-switch"
    config:
      spec_path: "iso8583-v87-ascii:v3.0.0"
      direction: "decode"
      on_error: "reject"

  - name: "to-rail"
    type: "translate"
    deploy: "rack-switch"
    config:
      spec_path: "billpay-rtp:v1.0.0"
      translation: "payment_request"
      validation: "enforce"
      on_error: "reject"

  - name: "conductor"
    type: "conductor"
    deploy: "rack-switch"
    config:
      correlation_key: ["instr_id"]
      park_fields: ["proc_code", "txn_amount", "txn_currency", "txn_datetime",
                    "stan", "rrn", "terminal_id", "merchant_id"]
      routes:
        - name: "rail"
          destination:
            failover: ["out_rail"]
      valet:
        store: "memory"
        default_ttl: "15s"

  - name: "encode-pacs008"
    type: "codec_iso20022"
    deploy: "rack-switch"
    config:
      spec_path: "iso20022-rtp:v1.0.0"
      direction: "encode"
      validation: "enforce"
      on_error: "reject"

  - name: "rail"
    type: "io_tcp"
    deploy: "rack-switch"
    config:
      mode: "client"
      connect: "127.0.0.1:7020"
      framing: "length_prefix4"
      read_responses: true

  - name: "decode-pacs002"
    type: "codec_iso20022"
    deploy: "rack-switch"
    config:
      spec_path: "iso20022-rtp:v1.0.0"
      direction: "decode"
      validation: "enforce"
      on_error: "reject"

  - name: "to-card"
    type: "translate"
    deploy: "rack-switch"
    config:
      spec_path: "billpay-rtp:v1.0.0"
      translation: "payment_status"
      validation: "enforce"
      on_error: "reject"

  - name: "decline"
    type: "bento"
    deploy: "rack-switch"
    config:
      ports:
        inputs: ["in"]
        outputs: ["out"]
      bento:
        pipeline:
          processors:
            - mapping: |
                # The conductor emits the original request on `error`, and
                # to-rail emits on `error` a request that it refuses.
                root = this
                root.mti = "0210"
                # codec_iso8583 takes the MTI from this metadata key.
                meta "iso8583.mti" = "0210"
                root.resp_code = match meta("error.reason") {
                  "timeout"            => "68"   # the rail did not answer in time
                  "violation"          => "03"   # invalid merchant: the biller is not in the directory
                  "translation_failed" => "96"   # the bridge could not process the payment
                  _                    => "91"   # the rail is not available
                }

  - name: "encode-8583"
    type: "codec_iso8583"
    deploy: "rack-switch"
    config:
      spec_path: "iso8583-v87-ascii:v3.0.0"
      direction: "encode"
      on_error: "reject"

  # ─────────────── rack-rail: the simulated rail ───────────────
  - name: "rail-in"
    type: "io_tcp"
    deploy: "rack-rail"
    config:
      mode: "server"
      bind: ":7020"
      framing: "length_prefix4"

  - name: "decode-pacs008"
    type: "codec_iso20022"
    deploy: "rack-rail"
    config:
      spec_path: "iso20022-rtp:v1.0.0"
      direction: "decode"
      validation: "enforce"
      on_error: "reject"

  - name: "ack"
    type: "translate"
    deploy: "rack-rail"
    config:
      spec_path: "rail-sim:v1.0.0"
      translation: "acknowledge"
      on_error: "reject"

  - name: "bank"
    type: "bento"
    deploy: "rack-rail"
    config:
      ports:
        inputs: ["in"]
        outputs: ["out"]
      bento:
        pipeline:
          processors:
            - mapping: |
                # The bank of the biller accepts a payment up to 1000.00.
                root = this
                let over = this.settlement_amount.number() > 1000
                root.tx_status = if $over { "RJCT" } else { this.tx_status }
                root.status_reason = if $over { "AM02" } else { deleted() }

  - name: "encode-pacs002"
    type: "codec_iso20022"
    deploy: "rack-rail"
    config:
      spec_path: "iso20022-rtp:v1.0.0"
      direction: "encode"
      validation: "enforce"
      on_error: "reject"

wires:
  # Request path
  - { from: "terminals.out",      to: "decode-8583.in" }
  - { from: "decode-8583.out",    to: "to-rail.in" }
  - { from: "to-rail.out",        to: "conductor.in" }
  - { from: "conductor.out_rail", to: "encode-pacs008.in" }
  - { from: "encode-pacs008.out", to: "rail.in" }
  # Simulated rail
  - { from: "rail-in.out",        to: "decode-pacs008.in" }
  - { from: "decode-pacs008.out", to: "ack.in" }
  - { from: "ack.out",            to: "bank.in" }
  - { from: "bank.out",           to: "encode-pacs002.in" }
  - { from: "encode-pacs002.out", to: "rail-in.in" }
  # Answer path
  - { from: "rail.out",               to: "decode-pacs002.in" }
  - { from: "decode-pacs002.out",     to: "conductor.in_reply" }
  - { from: "conductor.out_response", to: "to-card.in" }
  - { from: "to-card.out",            to: "encode-8583.in" }
  - { from: "to-rail.error",          to: "decline.in" }
  - { from: "conductor.error",        to: "decline.in" }
  - { from: "decline.out",            to: "encode-8583.in" }
  - { from: "encode-8583.out",        to: "terminals.in" }
```

The simulated bank uses a second translation spec. It builds the `pacs.002` from the `pacs.008`, inside one protocol. Save it as `specs/rail-sim.translation.yaml`:

```yaml
spec:
  kind: "translation"
  id: "rail-sim"
  version: "1.0.0"
  specs:
    rail: "iso20022-rtp:v1.0.0"
  participant_id: "99000000201"   # the bank of the biller
  translations:
    acknowledge:
      from: "rail:pacs.008.001.08"
      to: "rail:pacs.002.001.10"
      bindings:
        - { to: "orig_msg_id", from: "msg_id" }
        - { to: "orig_msg_name", value: "pacs.008.001.08" }
        - { to: "orig_creation_time", from: "creation_time" }
        - { to: "instr_id", from: "instr_id" }
        - { to: "tx_status", value: "ACTC" }
        - { to: "receiver_participant", from: "sender_participant" }
        # The answer goes back: the two agents change places.
        - { to: "instructing_agent", from: "instructed_agent" }
        - { to: "instructed_agent", from: "instructing_agent" }
```

The answer goes to the participant that sent the request: `receiver_participant` takes the value of `sender_participant`. The last two bindings exchange the values of `instructing_agent` and `instructed_agent`. This works because every binding reads the message as it arrived, and the translator writes after all the reads. The order of the bindings does not change the result.

`bank` then makes the one decision that is not a translation: it rejects an amount over 1000.00.

Five details carry the bridge:

- **`instr_id` links the two legs.** `to-rail` generates the instruction identifier and writes it to `instr_id`. `conductor` parks the ticket under `instr_id`. The rail copies the identifier into `OrgnlInstrId`, and `decode-pacs002` writes it to `instr_id` too, because the spec gives both fields that alias. The conductor reads the same alias on both legs, so it finds the ticket.
- **The conductor comes between the translation and the encode.** It needs the instruction identifier, which exists after `to-rail`. It handles only structured messages, so it comes before `encode-pacs008`.
- **`to-card` comes after the conductor.** The conductor restores the parked ISO 8583 fields first. `to-card` then adds the response code to a message that already has the STAN, the RRN and the terminal, and `encode-8583` builds a complete `0210`.
- **Every refusal gets an answer.** `to-rail` and `conductor` send what they refuse on their `error` ports, and `decline` answers the terminal with a `0210`. `decline` sets the MTI in the `iso8583.mti` metadata, because `encode-8583` takes the MTI from there. The response code depends on `error.reason`: `03` for a biller that is not in the directory, `68` when the rail does not answer in time, `91` when the rail is not available.
- **`park_fields` keeps the ISO 8583 context.** The `pacs.002` does not carry the STAN, the RRN or the terminal. The conductor keeps them with the ticket. The PAN is not in the list: the `0210` does not need it, and the bridge does not keep it.

---

## Run it

Build the binaries:

```bash
make build
```

Copy the files of the tutorial into a working directory:

```bash
mkdir -p specs lookups scenarios
cp examples/specs/iso8583-v87-ascii.yaml examples/specs/iso20022-rtp.yaml \
   examples/specs/billpay-rtp.translation.yaml examples/specs/rail-sim.translation.yaml specs/
cp examples/lookups/billers.yaml lookups/
cp examples/scenarios/billpay_rtp.yaml scenarios/
```

The Mixer API needs a token. Choose one and give it to the Mixer in `fluxrig-mixer.toml`:

```toml
# fluxrig-mixer.toml
[api]
auth_token = "change-me"
```

Start the Mixer:

```bash
./bin/fluxrig-mixer -c fluxrig-mixer.toml --auto-adopt
```

Upload the specs and the records of the billers to the Mixer. Upload the protocol specs and the records first: the Mixer refuses a translation spec that names an artefact it does not have. A spec declares its name and version. The records declare neither, so give them with `--name` and `--tag`:

```bash
export FLUXRIG_API_TOKEN=change-me
./bin/fluxrig spec import --api specs/iso8583-v87-ascii.yaml
./bin/fluxrig spec import --api specs/iso20022-rtp.yaml
./bin/fluxrig spec import --api --name billers --tag v1 lookups/billers.yaml
./bin/fluxrig spec import --api specs/billpay-rtp.translation.yaml
./bin/fluxrig spec import --api specs/rail-sim.translation.yaml
```

The Mixer checks each artefact with the loader of its kind before it stores it. See [Uploading to the Mixer](../reference/spec_manager.md#upload).

Start the two Racks, each in its own terminal. The two Racks run on one machine. Thus each Rack needs its own name, state directory, store directory and log file:

```toml
# rack-switch.toml
[base]
name = "rack-switch"
state_dir = "./data/rack-switch"

[store]
dir = "./data/rack-switch"

[logging]
filename = "logs/rack-switch.log"
```

```toml
# rack-rail.toml
[base]
name = "rack-rail"
state_dir = "./data/rack-rail"

[store]
dir = "./data/rack-rail"

[logging]
filename = "logs/rack-rail.log"
```

```bash
./bin/fluxrig rack -c rack-switch.toml
./bin/fluxrig rack -c rack-rail.toml
```

Import the scenario and activate it. The Mixer sends it to the two Racks. It also sends every artefact that the scenario names: the protocol specs, the two translation specs and the records of the billers.

```bash
./bin/fluxrig scenario import --api --activate scenarios/billpay_rtp.yaml
```

Send a bill payment of 42.50 USD for the first biller:

```bash
./bin/iso8583-tool -mode auth \
  -auth-target 127.0.0.1:8583 \
  -auth-spec specs/iso8583-v87-ascii.yaml \
  -auth-mti 0200 \
  -auth-proc-code 500000 \
  -auth-amount 000000004250 \
  -auth-currency 840 \
  -auth-merchant-id BILLER000000001 \
  -auth-terminal-id TERM0001 \
  -auth-stan 000123 \
  -auth-rrn auto \
  -auth-expect-mti 0210 \
  -auth-expect-de39 00
```

The tool sends the `0200` and checks that the `0210` has response code `00`. `-auth-rrn auto` makes the RRN from the date and the STAN.

---

## Follow one payment

The `0200` arrives with these fields. This table masks the PAN, as the logs do:

| Alias | DE | Value |
| :--- | :--- | :--- |
| `mti` | 0 | `0200` |
| `pan` | 2 | `411111******1111` |
| `proc_code` | 3 | `500000` |
| `txn_amount` | 4 | `000000004250` |
| `stan` | 11 | `000123` |
| `rrn` | 37 | `628012000123` |
| `terminal_id` | 41 | `TERM0001` |
| `merchant_id` | 42 | `BILLER000000001` |
| `txn_currency` | 49 | `840` |

`to-rail` adds the ISO 20022 fields, and `encode-pacs008` builds this `pacs.008`. The network puts each message in its envelope, `<Message xmlns="urn:tch">`: a business application header (`AppHdr`), then the document. Each element has the prefix of its namespace: `head:` for the header and `ct:` for the credit transfer. An `identity` comment marks each value that the codec always writes, and a `profile` comment each value that it writes for the profile:

```xml
<Message xmlns="urn:tch"
         xmlns:head="urn:iso:std:iso:20022:tech:xsd:head.001.001.01"
         xmlns:ct="urn:iso:std:iso:20022:tech:xsd:pacs.008.001.08">
  <AppHdr>
    <head:Fr><head:FIId><head:FinInstnId><head:ClrSysMmbId><head:MmbId>99000000101</head:MmbId></head:ClrSysMmbId></head:FinInstnId></head:FIId></head:Fr>  <!-- identity -->
    <head:To><head:FIId><head:FinInstnId><head:ClrSysMmbId><head:MmbId>99000000201</head:MmbId></head:ClrSysMmbId></head:FinInstnId></head:FIId></head:To>
    <head:BizMsgIdr>B2026100799000000101BFLX77369368627</head:BizMsgIdr>      <!-- identity -->
    <head:MsgDefIdr>pacs.008.001.08</head:MsgDefIdr>                           <!-- identity -->
    <head:CreDt>2026-10-07T14:21:03</head:CreDt>                               <!-- identity -->
  </AppHdr>
  <CreditTransfer>
    <ct:FIToFICstmrCdtTrf>
      <ct:GrpHdr>
        <ct:MsgId>M2026100799000000101BFLX77369368626</ct:MsgId>               <!-- identity -->
        <ct:CreDtTm>2026-10-07T14:21:03</ct:CreDtTm>                           <!-- identity -->
        <ct:NbOfTxs>1</ct:NbOfTxs>                                             <!-- profile -->
        <ct:TtlIntrBkSttlmAmt Ccy="USD">42.50</ct:TtlIntrBkSttlmAmt>           <!-- profile -->
        <ct:IntrBkSttlmDt>2026-10-07</ct:IntrBkSttlmDt>                        <!-- profile -->
        <ct:SttlmInf>
          <ct:SttlmMtd>CLRG</ct:SttlmMtd>                                      <!-- profile -->
          <ct:ClrSys><ct:Cd>TCH</ct:Cd></ct:ClrSys>                            <!-- profile -->
        </ct:SttlmInf>
      </ct:GrpHdr>
      <ct:CdtTrfTxInf>
        <ct:PmtId>
          <ct:InstrId>2026100799000000101BFLXR77369368625</ct:InstrId>
          <ct:EndToEndId>628012000123</ct:EndToEndId>
          <ct:TxId>628012000123</ct:TxId>
        </ct:PmtId>
        <ct:PmtTpInf>
          <ct:SvcLvl><ct:Cd>SDVA</ct:Cd></ct:SvcLvl>                           <!-- profile -->
          <ct:LclInstrm><ct:Prtry>STANDARD</ct:Prtry></ct:LclInstrm>
          <ct:CtgyPurp><ct:Prtry>CONSUMER</ct:Prtry></ct:CtgyPurp>
        </ct:PmtTpInf>
        <ct:IntrBkSttlmAmt Ccy="USD">42.50</ct:IntrBkSttlmAmt>
        <ct:ChrgBr>SLEV</ct:ChrgBr>                                            <!-- profile -->
        <ct:InstgAgt><ct:FinInstnId><ct:ClrSysMmbId><ct:MmbId>990000001</ct:MmbId></ct:ClrSysMmbId></ct:FinInstnId></ct:InstgAgt>  <!-- profile -->
        <ct:InstdAgt><ct:FinInstnId><ct:ClrSysMmbId><ct:MmbId>990000002</ct:MmbId></ct:ClrSysMmbId></ct:FinInstnId></ct:InstdAgt>  <!-- profile -->
        <ct:Dbtr><ct:Nm>Example Payments Operator</ct:Nm></ct:Dbtr>
        <ct:DbtrAcct><ct:Id><ct:Othr><ct:Id>5500001111</ct:Id></ct:Othr></ct:Id></ct:DbtrAcct>
        <ct:DbtrAgt><ct:FinInstnId><ct:ClrSysMmbId><ct:MmbId>990000001</ct:MmbId></ct:ClrSysMmbId></ct:FinInstnId></ct:DbtrAgt>
        <ct:CdtrAgt><ct:FinInstnId><ct:ClrSysMmbId><ct:MmbId>990000002</ct:MmbId></ct:ClrSysMmbId></ct:FinInstnId></ct:CdtrAgt>
        <ct:Cdtr><ct:Nm>Example Water Utility</ct:Nm></ct:Cdtr>
        <ct:CdtrAcct><ct:Id><ct:Othr><ct:Id>7700123456</ct:Id></ct:Othr></ct:Id></ct:CdtrAcct>
        <ct:RmtInf><ct:Ustrd>TERM0001</ct:Ustrd></ct:RmtInf>
      </ct:CdtTrfTxInf>
    </ct:FIToFICstmrCdtTrf>
  </CreditTransfer>
</Message>
```

Where each value comes from:

| `pacs.008` alias | Source |
| :--- | :--- |
| `settlement_amount` `42.50`, `settlement_currency` `USD` | `txn_amount` `000000004250` and `txn_currency` `840`, with `format: amount` |
| `end_to_end_id`, `tx_id` | `rrn` |
| `instr_id` | Generated by `to-rail`, in the format of the profile, from `participant_id` |
| `receiver_participant` | A constant in the translation: the participant of the bank of the biller |
| `creditor_name`, `creditor_account`, `creditor_agent` | The records of the billers, with `merchant_id` as the key |
| `debtor_name`, `debtor_account`, `debtor_agent` | Constants in the translation |
| `remittance_info` | `terminal_id` |

On `rack-rail`, `ack` builds the answer from the `pacs.008`, and `bank` accepts the payment. `encode-pacs002` builds this `pacs.002`:

```xml
<Message xmlns="urn:tch"
         xmlns:head="urn:iso:std:iso:20022:tech:xsd:head.001.001.01"
         xmlns:ps="urn:iso:std:iso:20022:tech:xsd:pacs.002.001.10">
  <AppHdr>
    <head:Fr><head:FIId><head:FinInstnId><head:ClrSysMmbId><head:MmbId>99000000201</head:MmbId></head:ClrSysMmbId></head:FinInstnId></head:FIId></head:Fr>
    <head:To><head:FIId><head:FinInstnId><head:ClrSysMmbId><head:MmbId>99000000101</head:MmbId></head:ClrSysMmbId></head:FinInstnId></head:FIId></head:To>
    <head:BizMsgIdr>B2026100799000000201BFLX77369368629</head:BizMsgIdr>
    <head:MsgDefIdr>pacs.002.001.10</head:MsgDefIdr>
    <head:CreDt>2026-10-07T14:21:03</head:CreDt>
  </AppHdr>
  <MessageStatusReport>
    <ps:FIToFIPmtStsRpt>
      <ps:GrpHdr>
        <ps:MsgId>M2026100799000000201BFLX77369368628</ps:MsgId>
        <ps:CreDtTm>2026-10-07T14:21:03</ps:CreDtTm>
      </ps:GrpHdr>
      <ps:OrgnlGrpInfAndSts>
        <ps:OrgnlMsgId>M2026100799000000101BFLX77369368626</ps:OrgnlMsgId>
        <ps:OrgnlMsgNmId>pacs.008.001.08</ps:OrgnlMsgNmId>
        <ps:OrgnlCreDtTm>2026-10-07T14:21:03</ps:OrgnlCreDtTm>
        <ps:OrgnlNbOfTxs>1</ps:OrgnlNbOfTxs>
      </ps:OrgnlGrpInfAndSts>
      <ps:TxInfAndSts>
        <ps:OrgnlInstrId>2026100799000000101BFLXR77369368625</ps:OrgnlInstrId>
        <ps:TxSts>ACTC</ps:TxSts>
        <ps:AccptncDtTm>2026-10-07T14:21:03</ps:AccptncDtTm>
        <ps:InstgAgt><ps:FinInstnId><ps:ClrSysMmbId><ps:MmbId>990000002</ps:MmbId></ps:ClrSysMmbId></ps:FinInstnId></ps:InstgAgt>
        <ps:InstdAgt><ps:FinInstnId><ps:ClrSysMmbId><ps:MmbId>990000001</ps:MmbId></ps:ClrSysMmbId></ps:FinInstnId></ps:InstdAgt>
      </ps:TxInfAndSts>
    </ps:FIToFIPmtStsRpt>
  </MessageStatusReport>
</Message>
```

The answer has its own identifiers and goes back to the operator: `To` is the `Fr` of the request. `decode-pacs002` writes `OrgnlInstrId` to `instr_id`. The conductor finds the ticket under `instr_id` and restores the parked fields. `to-card` translates `ACTC` through the table to `resp_code` `00` and sets the MTI to `0210`. `encode-8583` builds the `0210`:

| Alias | DE | Value | Source |
| :--- | :--- | :--- | :--- |
| `mti` | 0 | `0210` | The target message of `payment_status` |
| `proc_code` | 3 | `500000` | Parked |
| `txn_amount` | 4 | `000000004250` | Parked |
| `stan` | 11 | `000123` | Parked |
| `rrn` | 37 | `628012000123` | Parked |
| `resp_code` | 39 | `00` | `tx_status` through the table |
| `terminal_id` | 41 | `TERM0001` | Parked |
| `merchant_id` | 42 | `BILLER000000001` | Parked |
| `txn_currency` | 49 | `840` | Parked |

---

## A declined payment

The simulated bank refuses a payment over 1000.00. Send 1250.00:

```bash
./bin/iso8583-tool -mode auth \
  -auth-target 127.0.0.1:8583 \
  -auth-spec specs/iso8583-v87-ascii.yaml \
  -auth-mti 0200 \
  -auth-proc-code 500000 \
  -auth-amount 000000125000 \
  -auth-currency 840 \
  -auth-merchant-id BILLER000000001 \
  -auth-terminal-id TERM0001 \
  -auth-stan 000124 \
  -auth-rrn auto \
  -auth-expect-mti 0210 \
  -auth-expect-de39 13
```

The bank answers `TxSts` `RJCT` with reason `AM02`. The table entry `RJCT/AM02` gives response code `13`. The terminal shows a decline, and the bridge sent no other message.

When the rail does not answer before `default_ttl`, the conductor sends the original request on `error`. `decline` sets response code `68`, and the terminal gets a `0210` for its `0200`. The conductor never answers the terminal by itself: a gear you can see in the scenario writes every response.

When the rail is not available, the conductor does not wait. `rail` reports its connection state to the conductor, and a request that arrives while the connection is down leaves on `error` at once, with `error.reason: no_destination`. `decline` sets response code `91`. Try it: stop `rack-rail`, then send the first payment again with `-auth-expect-de39 91`.

---

## What the gears refuse

With `validation: enforce`, each gear checks what it produces, and the codecs also check what they read:

- **`translate`** checks its output against the target message in the target spec. A field that the target message requires, and that is still empty after the translation, is a finding.
- **`codec_iso20022`** checks each message against the ISO 20022 schema, the rules of the `rtp` profile and the mandatory and forbidden fields of its spec. It checks on decode and on encode.

A message that fails a check does not leave the gear on `out`. The gear logs the findings, records them in `translate.violations` or `codec.violations`, and applies `on_error`. With `reject`, the message leaves on the `error` port with `error.reason: violation`.

Examples of what the gears refuse:

- A `merchant_id` that is not in the directory. The lookup gets no record, `creditor_name` stays empty, and `to-rail` refuses the message before `encode-pacs008` gets it.
- A currency other than `USD`. The profile refuses it.
- A routing number in `ClrSysMmbId.MmbId` that does not have exactly 9 characters.
- A `MsgId` or `InstrId` that does not have the format of the profile.

Try it: send the first command again with `-auth-merchant-id BILLER000000999` and `-auth-expect-de39 03`. `to-rail` refuses the message, and the request does not go to the rail. `to-rail` sends it on `error`, and `decline` answers the terminal with response code `03` (invalid merchant).

With `validation: warn`, a gear logs and records the same findings, but the message continues. Use `warn` to observe a new translation on real traffic before you enforce it.

`translate` also checks the source of each message. It compares the `codec.spec_id` and `codec.spec_version` that the codec wrote on the message with the protocol spec that the translation names. A message that a different version of the spec decoded is a finding. Thus a translation written for one version of a spec does not run unnoticed on traffic decoded with another version.

---

## Configuration reference

The configuration, the ports and the metadata of the two new gears are in their reference pages:

- [ISO 20022 codec gear](../reference/gears/codec_iso20022.md): the path convention, the `rtp` profile and the elements it fills, validation.
- [Translate gear](../reference/gears/translate.md): the format of a translation spec, the binding kinds, lookups, and what the gear checks.

---

## Run it as a test

The suite `test/robot/suites/billpay_rtp` runs this tutorial as written. It uses the files in `examples/` without a change, uploads them through the Mixer API, and checks the four response codes of this page: `00`, `13`, `03` and `91`.

```bash
make test-robot-billpay
```

The command prints a run root, `/tmp/fluxrig/robot_run_<timestamp>`. The suite works in `work/billpay_rtp/` under it, with one directory for each component: `mixer`, `switch` and `rail`. Each one keeps its logs in `logs/`: the process log (`mixer.log` or `fluxrig.log`) and `process_stdout.log`. The Robot report is in `results/`.

---

## Next steps

- [ISO 20022 codec gear](../reference/gears/codec_iso20022.md) and [translate gear](../reference/gears/translate.md): the reference for the two gears of this tutorial.
- [Payment switch with the conductor](payment_switch_conductor.md): routing, failover and timeouts on the conductor.
- [Robot Framework testing](iso8583_robot_suite.md): build a suite that sends traffic through a scenario and checks the answers.
