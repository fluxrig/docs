---
slug: /reference/protocol/iso8583
title: ISO8583 SDL
---

# ISO8583 SDL

fluxrig carries no protocol of its own: a scenario wires gears, and a gear
speaks whatever its configuration says it speaks. The SDL is that
configuration for ISO 8583, which is where the description goes deepest,
because a payment dialect is a contract and not only a layout.

The ISO8583 SDL is a YAML-based format used to strictly define the dialect of ISO8583 protocols (e.g., Visa Base I, Mastercard IPM, Fiserv, Postilion).

An ISO 8583 dialect is usually described as a byte layout and nothing more. The
XML packager definitions the industry has used for decades do exactly that, and
so does [moov-io/iso8583](https://github.com/moov-io/iso8583), which fluxrig uses
directly: it is a dependency of this project, and its vocabulary *is* the SDL's
wire layer rather than something restated beside it. A byte layout says where a
field sits and how it is encoded, and stops there.

The fluxrig SDL adds what a layout cannot express: which elements a given message
requires, what values they may carry, and what a rule reads to decide. Four
things in total:

1.  **Physical layout**: encoding, length prefixes, composites. moov's vocabulary, consumed as it is.
2.  **Semantic aliases**: human-readable names (`card.pan` rather than `field 2`).
3.  **Rules per message**: usage, conditions, value domains and cross-field checks, applied to traffic.
4.  **Simulation macros**: instructions for generating messages. Declared, and not executed by the open product.

## Schema structure

A spec has two layers. The **wire layer** describes how bytes are laid out, in
the [moov-io/iso8583](https://github.com/moov-io/iso8583) declarative vocabulary.
The **semantic layer** describes what the fields mean, and is fluxrig's.

```yaml
spec:
  id: "acme-auth"
  version: "1.0.0"

  # Wire layer. `source` names where it comes from and `fields` is merged over
  # it, so a dialect states only its differences.
  wire:
    source: "moov:spec87ascii"
    fields:
      11: { padding: { type: Left, pad: "0" } }

  # Semantic layer. Wire attributes are not accepted here: a spec that sets
  # `length` or `enc` under `fields` fails to load, naming the key and where it
  # belongs.
  fields:
    2:
      alias: "card.pan"
      sensitivity: pan
    4:
      name: "Amount, Transaction"     # overrides the base's "Transaction Amount"
      alias: "amount"
    39:
      alias: "resp_code"
      values_ref: "response_code"
```

Neither DE 2 nor DE 39 says what it is called, and both are still labelled: the
label comes from the wire layer, where `moov:spec87ascii` already declares one
for every element it carries. `name` appears on DE 4 because there the base's
wording is not the one this protocol uses.

### Where the wire layer comes from

Three origins, and they compose. `wire.fields` is always merged over whatever
`source` produced, so the same key does the same job in all three.

| `wire` contains | What you get |
| :--- | :--- |
| `source` only | A base used as-is. |
| `source` and `fields` | A dialect: the base, with the differences this spec declares. |
| `fields` only | A spec with no upstream base, carrying its whole wire layer here. |

A `source` beginning `moov:` names a base resolved from the linked library, so it
cannot drift from upstream: `moov:spec87ascii` and `moov:spec87hex`. Anything
else is a path to a wire document beside the spec, relative to it. A path that
climbs out of the spec's own directory is refused, because a spec is deployed to
Racks and the path it names is not the operator's.

Naming a base is what lets the wire layer be *consumed* rather than restated.
Stripping the semantic keys from a spec yields a document the upstream library
accepts on its own, and that property is asserted in the test suite, so the two
layers cannot quietly fuse.

> [!WARNING]
> **Transport Headers & Offsets:** Ensure your SDL definition exactly matches the byte offset delivered by the I/O Gear. If the `io_iso8583` gear strips transport-level headers (like Visa V.I.P.), the SDL must begin defining at the MTI, NOT the proprietary header.

## Field definitions

Fields are defined by ID (0-128). ID 0 is reserved for MTI. ID 1 (Bitmap) is implicit.

### Wire attributes

These describe the bytes, and belong in the wire document or in
`wire.fields`. The vocabulary is the upstream library's, carried verbatim, so
these attributes are documented there rather than restated here:

- [Defining message specifications](https://github.com/moov-io/iso8583#defining-message-specifications)
  is the reference for what each attribute means. It names them as Go struct
  fields (`Enc`, `Pref`, `Pad`); the keys below are the same vocabulary written
  declaratively.
- [`examples/specs/spec87ascii.yaml`](https://github.com/moov-io/iso8583/blob/master/examples/specs/spec87ascii.yaml)
  is a complete spec in exactly the form used here, and is the fastest way to see
  the keys in use.
- [Composite fields](https://github.com/moov-io/iso8583/blob/master/docs/composite-fields.md)
  covers `subfields` in depth: TLV, positional parts, and unknown tags.

The vocabulary is consumed, not copied: an attribute moov adds is available here
with no change to fluxrig, and one it renames is a breaking change.

| Attribute | Description |
| :--- | :--- |
| `type` | `String`, `Numeric`, `Binary`, `Bitmap`, `Composite`, `Hex`, `Track2`. |
| `length` | Exact length for fixed fields, maximum for variable ones. |
| `description` | The element's label, in moov's vocabulary. A named base carries one for every field it declares, and the semantic `name` overrides it. See the note below. |
| `enc` | Value encoding: `ASCII`, `EBCDIC`, `BCD`, `Binary`, `HexToASCII`, `BerTLV`, and others. |
| `prefix` | Length prefix, as `<encoding>.<size>`: `ASCII.Fixed`, `ASCII.LL`, `BCD.LLL`, `Hex.LLLL`. |
| `padding` | `{ type: Left, pad: "0" }`. Required on any fixed-length field whose value may be shorter than its declared length; without it, packing a six-digit field with a two-digit value fails. |
| `subfields` | How the parts of a composite are framed: `from`/`to`, `tag`, `repeat`, `unknown_tags`. What they *mean* belongs under `fields`. See [Subfields](#subfields). |

> **`description` is the wire layer's word for a label**, and the semantic layer
> does not use it: `name` overrides that label, `meaning` is the prose. A
> `description` under `spec.fields` fails to load.

### Semantic attributes

These describe meaning, and belong under `fields`.

| Attribute | Description |
| :--- | :--- |
| `name` | **Overrides** the label the wire layer carries. Omit it and the element is still labelled; declare it where the upstream wording is not this protocol's, as DE 4 does. |
| `alias` | Dot-notation name the rest of the pipeline addresses the field by. |
| `meaning` | Prose for the generated protocol reference: what the element is. |
| `note` | What a reader has to know that is not the definition: a caveat, a consequence, a mistake that gets made. |
| `sensitivity` | Data classification: `pan`, `chd`, `sad`, `pii`, `none`. Drives masking on its own. |
| `log_mask` | Masks the value in logs and console output. Implied by any `sensitivity` other than `none`. |
| `scope` | `public` or `private`. Private fields are excluded from published documentation. |
| `values_ref` | Names a shared value set declared under `enums`. See [Value sets](#value-sets). |
| `messages` | What the field means in each message it has a rule for. See [The per-message matrix](#the-per-message-matrix). |
| `subfields` | What the parts of a composite mean. See [Subfields](#subfields). |

**`sensitivity` is the one to set.** A field classified `pan`, `chd`, `sad` or
`pii` is masked whether or not it also sets `log_mask`. Requiring both is how a
PAN reaches a log: someone sets the classification and reasonably trusts it.

### Example

```yaml
spec:
  id: "acme-auth"
  version: "1.0.0"
  wire:
    source: "moov:spec87ascii"
    fields:
      # The base leaves padding to the caller on most numerics. A STAN of "42"
      # in a six-digit fixed field has to become "000042" to pack at all.
      11: { padding: { type: Left, pad: "0" } }
  fields:
    2:
      name: "Primary Account Number"
      alias: "card.pan"
      sensitivity: pan          # masked in logs; never persisted in the clear
    4:
      name: "Amount, Transaction"
      alias: "amount.transaction"
    11:
      name: "System Trace Audit Number"
      alias: "stan"
```

## The per-message matrix

> [!NOTE]
> Two different things are checked, at two different times, and the distinction
> is worth keeping.
>
> **The spec is checked when it loads.** See [What is checked when a spec
> loads](#what-is-checked-when-a-spec-loads). A `when` that does not parse, or one
> reading a field the spec never declared, fails the whole load, at boot.
>
> **A message is checked against this matrix** when the codec is configured to do
> it. See [Enforcing the rules](#enforcing-the-rules): the default is off, so
> turning it on is a deployment decision rather than a consequence of upgrading.

A field does not mean one thing. DE 39 is issuer-originated in an authorization
response, absent from the request that caused it, and a network-management
disposition in an `0810`. `messages` on the field says so, one entry per message:

```yaml
  39:
    alias: "resp_code"
    values_ref: "response_code"
    messages:
    - mti: ["0110", "0210"]
      usage: mandatory
      response_value: new
    - mti: "0810"
      usage: mandatory
      response_value: new
    - mti: "0400"
      usage: forbidden
      note: "A reversal request states what is being reversed, not how it was answered."
    - mti: "0800"
      usage: forbidden
      note: "A network management request carries no disposition."
```

Four entries for one element, and none of them is redundant: it is required in an
authorization or financial response, required again in a network management
response, and forbidden in both of the requests that cause them.

An entry carries what varies by message, and `mti` takes a list because messages
that share a rule are common, and repeating an identical entry per message is how a
matrix rots.

**Two independent axes.** `usage` answers whether the field is there at all:
`mandatory`, `optional` or `forbidden`. `response_value` answers something else
entirely, namely how a response's value relates to the request's, and conflating the
two is the mistake to avoid, because "it must be present whatever its value" is
`usage: mandatory` and nothing more.

| `response_value` | Meaning | What a validator does |
| :--- | :--- | :--- |
| `echo` | The response must carry the request's value | Compare for equality; a difference is a fault |
| `new` | The responder originated it; there was nothing to echo | Do not compare |
| `modified` | It derives from the request's value and may legitimately differ | Do **not** compare for equality |

`echo` is the sharpest conformance check available: DE 11 arriving changed means
correlation is broken. `new` is DE 39: the issuer decides it. `modified` is DE 4
under partial approval, where the response returns less than the amount asked
for: a difference is expected and is the point of the message.

It is meaningless on a request, where there is no prior value to relate to, and a
spec that sets it there fails to load.

An entry also carries the value domain and a `note` scoped to that pairing.

**Cases within a message.** Two `0200`s differ by what the transaction is, so an
entry may carry a `when` that narrows it. Entries are evaluated in order and the
first match wins, which makes an entry without a `when` the message's default,
and therefore last:

```yaml
  55:
    name: "ICC Data"
    messages:
    - mti: ["0100", "0200"]
      usage: mandatory
      when: "field(22) == '05' || field(22) == '07'"   # chip and contactless
      note: "Chip read; the cryptogram travels here."
    - mti: ["0100", "0200"]
      usage: forbidden
      when: "field(22) == '02' || field(22) == '90'"   # magnetic stripe
    - mti: ["0100", "0200"]
      usage: optional                                  # any other entry mode
```

Put the last entry first and it wins every time: it has no condition, so it
matches any `0100`, and the two above it are never reached.

There is no `usage: conditional`. The entry's `when` is the condition; a usage
that only restated that one exists would say nothing about the field.

**Why on the field and not on the message.** A field's rules are what changes
together: a scheme bulletin says "DE 48 tag 9F5A now permits value X in
reversals": one field, several messages. The per-message view a reader wants is
a projection of this, derived by inverting it.

## Value sets

A value set says what a coded field can hold. It is declared under `enums` and
referenced by name, or written inline as `validValues`.

```yaml
enums:
  pos_entry_mode:
    closed: true              # a value outside this list is invalid
    values:
      "05": {name: "Integrated circuit card (chip)"}
      "07": {name: "Contactless (chip)"}
  response_code:
    closed: false             # this is what is documented, not what is permitted
    values:
      "00": {name: "Approved",      category: approved}
      "05": {name: "Do not honor",  category: decline_soft}
      "91": {name: "Issuer inoperative", category: system}
```

**`closed` is the difference between two claims a bare list cannot tell apart**:
whether a value outside the set is *invalid*, or merely *undocumented*. Both are
ordinary. `pos_entry_mode` is a fixed set from the standard, so an unknown entry
mode is a malformed message. Response codes are not: schemes add proprietary ones
continually, and a reader that rejected an unrecognised code would refuse live
traffic.

Open is the default, because refusing a value for being unfamiliar is the
expensive mistake in a payment switch. A closed set is the deliberate case and
says so.

**`name` and `category` are what make analytics possible.** `name` turns `39=05`
into "Do not honour". `category` groups values into the outcomes anyone actually
charts, and it is also what bounds an open set used as a telemetry dimension: a
code nobody listed still lands in a category that was.

## Subfields

A composite is described from both sides. The wire layer frames the parts, and the
field says what they are:

```yaml
  wire:
    fields:
      55:
        type: Composite
        subfields:
          layout: tlv
          unknown_tags: preserve        # round-trip tags not declared here
          parts:
          - { tag: 9F02, type: Numeric, length: 6 }
          - { tag: 5F2A, type: Numeric, length: 3 }
  fields:
    55:
      name: "ICC Data"
      subfields:
        layout: tlv
        parts:
        - tag: 9F02
          name: "Amount, Authorized"
        - tag: 5F2A
          name: "Transaction Currency Code"
          values_ref: currency_iso4217
```

`layout` is `tlv` (parts bound by `tag`), `positional` (bound by order, framed by
`from`/`to`, 1-based and inclusive), or `bitmapped` (presence by bitmap bit, used
by some dialects for private composites). All three compile to the upstream
library's native `Composite`.

Of the semantic side, `alias` is what the pipeline addresses a part by. `name`
and `meaning` reach the [generated protocol reference](#enforcing-the-rules),
which renders a composite's parts alongside the element that carries them.

`values_ref` on a part loads and resolves but is **not enforced on a message**:
validation applies value domains at the element level, not inside a composite. A
`when` may still read a part (`field(55.9F02)`, `field(3.1)`), and comparisons
there are on the characters the part carries, since a part declares no
`format.kind`.

## What is checked when a spec loads

A spec is mostly claims about itself: this field's values come from that enum,
this message pairs with that one, this condition reads that data element. Every
one of them is resolved when the spec loads, and a claim that points at nothing
**fails the whole spec**, and never becomes an error on a message.

That boundary is deliberate. A Rack that accepted a spec has already told the
Mixer it is serving that protocol, so a reference that only breaks on the right
transaction breaks in production.

What is rejected:

| | |
| :--- | :--- |
| `values_ref` naming a value set that is not declared under `enums` | |
| A special value set other than `@messages` | |
| A `when` that does not parse | reported with the position |
| A `when` reading a field, or a composite part, that the spec does not declare | |
| A message rule, a `check`, a `pairs_with` or a transition naming a message outside the catalog | |
| `source` on a request message | provenance is a property of a response |
| `log_mask: false` on a classified field | masking follows `sensitivity` and cannot be switched off |
| `format.currency_field` pointing at a missing field, or one that carries no currency codes | |
| An observability dimension naming nothing, or one whose values are unbounded | see below |
| A histogram whose subject or `by` names no field, or that groups an amount by something other than its currency | grouping by anything else adds up figures that were never comparable |
| Conditions that depend on one another in a cycle | generation resolves conditions to a fixpoint, and a cycle never reaches one |

**When is a dimension bounded?** A telemetry dimension must have a limited number
of values, or every distinct one becomes its own time series and the metrics
backend pays for it. A value set answers this only if it is `closed`, or if every
value carries a `category`, because categories are what give a value nobody listed a
bucket to land in.

That is also what naming a value is for. `name` turns `39=05` into "Do not
honour", a row someone can read; `category` turns seventeen response codes into
six outcomes (approved, referral, soft decline, hard decline, error, system)
which is the axis anyone actually charts. Without them a coded field is a
histogram of opaque strings.

Every problem is reported in a single load, not one per fix.

## Enforcing the rules

The `messages` matrix and `checks` are applied to traffic by the
[`codec_iso8583` gear](../gears/codec_iso8583.md), through its `validation`
setting:

| `validation` | What it does |
| :--- | :--- |
| `off` (default) | The rules document the protocol and nothing is applied to a message. |
| `warn` | Every violation is logged and counted, and the message goes on unchanged. |
| `enforce` | A violation that rejects fails the message, which then follows `on_error`. |

The default is `off`, so upgrading changes nothing about what traffic is
accepted. Use `warn` to find out whether your spec matches your traffic, then
`enforce`.

What is applied, per message type:

*   **Usage.** `mandatory` and absent, or `forbidden` and present, rejects.
*   **Conditions.** A rule carrying a `when` applies only where the condition
    holds, and the reason on a rejection says which condition made it apply.
*   **Value domains.** A value outside a **closed** value set rejects. An open
    set permits values it does not list, so it is not a rule.
*   **Checks.** Each carries its own `severity`. A check marked `warn` never
    rejects, even while the gear is enforcing.

Every violation is reported, not only the first. What was found travels on the
message as `codec.violations`, and is counted on `flux.iso8583.violations` by
severity, kind and MTI.

A message type the spec says nothing about breaks nothing.

### Turning it on, end to end

Every command below runs against the reference spec that ships with fluxrig.

**1. Read what the spec says.** Before enforcing rules, look at them:

```bash
fluxrig spec doc examples/specs/iso8583-v87-ascii.yaml --format html --out reference.html
```

That is the document [linked here](protocol_reference.md), rendered from this
spec.

**2. File it as an artefact.** A spec a fleet runs is deployed, not copied to each
Rack by hand:

```bash
$ fluxrig spec import examples/specs/iso8583-v87-ascii.yaml
Imported iso8583-v87-ascii:v2.2.0@256e09885b96 -> 256e09885b9681cfdb...

$ fluxrig spec list
NAME               VERSION          IMPORTED              SIZE     HASH          TITLE
iso8583-v87-ascii  v2.2.0 (latest)  2026-09-06 21:24 UTC  33.6 KB  256e09885b96  ISO 8583:1987 (ASCII)
```

**3. Watch, before enforcing.** Name the spec in the codec and set `warn`:

```yaml
gears:
  - name: codec-in
    type: codec_iso8583
    deploy: edge
    config:
      spec_path: "iso8583-v87-ascii:v2.2.0"
      direction: decode
      validation: warn
```

Traffic is untouched. Each violation is logged with the rule that fired and the
spec it was judged against:

```
Message breaks a spec rule | flux.name=codec-in type=codec_iso8583 direction=decode
  mti=0200 severity=warn kind=usage rule="DE 2: is mandatory and is not present"
  spec_id=iso8583-v87-ascii spec_version=2.2.0
```

Each is also counted on `flux.iso8583.violations`, labelled by severity, kind and
MTI.

**4. Enforce, once the warnings are quiet.** Change `warn` to `enforce`. A
rejecting rule now fails the message, which then follows `on_error`.

Do not skip step 3. Enforcing on traffic you have not watched first means
finding out what your spec says about it by dropping messages.

### How a value compares

Equality reads the characters an element carries, so a response code of `"00"` is
not `"0"`. An element that declares a `format.kind` of `amount`, `date`, `time`,
`datetime` or `numeric` is compared as a number instead, which is what lets a
rule write `field(4) == 1000` rather than the element's own zero padding.

`pan` is not among them: a leading zero makes it a different card.

## Roadmap: simulation

> [!WARNING]
> The simulation blocks are checked for consistency when a spec loads and are not
> executed: no engine generates a message from them or answers one. A spec may
> declare `x-fluxrig-simulation` and nothing in the open product will read it.

### Planned Simulation Macros:

*   `$PAN(SCHEME, LEN)`: Generates valid Luhn PAN.
*   `$RAND(MIN, MAX)`: Random integer.
*   `$UUID`: Unique ID.
*   `$NOW(FORMAT)`: Current timestamp.
*   `$ECHO(FIELD)`: Copies value from Request (for Response generation).

---

## PCI-DSS compliance integration

The SDL is the primary tool for **PCI Scope Reduction**. By defining security attributes directly in the spec, you ensure that sensitive data is isolated and protected across the entire data plane.

### Recommended configuration for sensitive fields

| Field | Description | `name` | `sensitivity` |
| :--- | :--- | :--- | :--- |
| **2** | Primary Account Number | `PAN` | `pan` |
| **35** | Track 2 Data | `Track 2` | `sad` |
| **45** | Track 1 Data | `Track 1` | `sad` |
| **52** | Personal Identification Number | `PIN Block` | `sad` |
| **55** | ICC / Chip Data | `EMV Data` | `chd` |

Classifying a field is enough; `log_mask` follows from it. The classes come from
the PCI vocabulary: `pan` is the account number, `sad` is sensitive
authentication data, `chd` is the rest of the cardholder data, `pii` is personal
data that is not card data.

> [!IMPORTANT]
> A classified field is replaced with asterisks in all **Rack** and **Mixer** operational logs. What a classification implies for persistence beyond logs is [Roadmap]: the classes are declared and drive masking today, and the storage policy that derives from them is not yet implemented.