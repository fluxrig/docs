---
slug: /tutorials/roaming-enrichment
title: Enriching an authorization with a network signal
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Enriching an authorization with a network signal

> An issuing switch that asks a mobile operator, through a [CAMARA](https://camaraproject.org/) API, where the cardholder's handset is, compares it against the merchant's country, and hands its own authorizer the answer in a private ISO 8583 field. The authorizer makes no external call. Nothing new is compiled: the enrichment, the lookup and the operator simulator are all configuration.

## What you will build

A Rack sitting between a card scheme and an issuer's authorizer. An authorization arrives, is enriched, and continues.

The counterparties are not fluxrig, and they are played by different things because they do different jobs.

The **issuer's authorizer** answers whatever arrives, so it is `iso8583-tool`, the ISO 8583 host simulator the [ISO 8583 suite](iso8583_robot_suite.md) already uses. It is told to report the private field back in another field, which is how a test sees which outcome reached it.

The **card scheme** has to send a *specific* card for each case, which no responder can do. The suite packs each message itself through [`ISO8583Library`](../reference/robot_framework.md#sending-one-exact-message) and dials the switch, so the card is an argument to the test rather than a setting somewhere. Both talk ISO 8583 over their own TCP socket: a suite where every participant is also fluxrig would be a conversation with itself.

The operator is the exception, and deliberately so. There is no tool for it, and what it exercises is the thing this tutorial is about: an HTTP endpoint answering a published contract. It runs as a second Rack, so the call still crosses a process boundary rather than staying inside the one under test.

<LikeC4 project="roaming-enrichment" view="index" height={520} />

The topology above is generated from the scenario file by `fluxrig scenario viz`, so it cannot drift from what actually runs. Click any gear to drill into it.

One transaction through it, end to end:

<LikeC4 project="roaming-enrichment" view="flow_enriched" height={520} />

The case where the operator does not answer in time diverges at one step and rejoins at the next, so it is described below rather than drawn: only the value written into the private field differs, and everything after it is identical.

Sequences are hand-written rather than generated. Wires say what may talk to what; they never say what happens first, so the order of a transaction is editorial and lives in a file the generator does not overwrite.


When you are done, four things are true:

1. An authorization whose cardholder's handset is in the merchant's country reaches the authorizer with `RS00` in a private field.
2. One whose handset is elsewhere reaches it with `RS01`.
3. When the operator is slow, unavailable, or does not know the subscriber, the authorization still reaches the authorizer **within its time budget**, carrying which of those it was.
4. The cardholder's phone number appears in no outgoing message and no log.

The fourth is the one that fails silently when it is not asserted, which is why it is in the suite.

## Prerequisites

- A working Rack and Mixer (see the [quickstart](quickstart.md)).
- The full Rack build. This tutorial uses the `bento` gear, which is excluded by `-tags nobento`.
- No account with any operator or aggregator. The operator API is simulated locally.

**Everything here runs in clear text, and only here.** Both legs are plain: ISO 8583 over TCP to the scheme and the authorizer, and HTTP to the operator. That is a test running against loopback, where a certificate would add setup and prove nothing.

In production neither is. The ISO 8583 legs run over TLS, usually mutually authenticated, because that is what a scheme and an issuer's own authorizer require. The operator leg is REST over HTTPS, and an aggregator will hand you a credential that is worthless without it. Both are configuration rather than code: `io_iso8583` takes a `tls` block natively, the Bento `http` processor takes one for the outbound call, and the `http_server` input serves TLS from `cert_file` and `key_file`. What changes between this tutorial and a deployment is the certificates and where they come from, not the pipeline.

## Why the signal is worth having

A card transacting in Montevideo while its holder's handset is in Madrid is a fact the transaction does not contain. The merchant's country is in the message; the subscriber's is not, and only the mobile network knows it.

The reverse matters as much. A handset in the same country as the merchant is grounds for *removing* friction from a transaction that would otherwise be challenged.

## The contract this builds against

The CAMARA definition is pinned and vendored into the repository.

| | |
|:---|:---|
| Source | [camaraproject/DeviceStatus](https://github.com/camaraproject/DeviceStatus), `code/API_definitions/device-roaming-status.yaml`. As of **August 2026** CAMARA has split this API into [its own repository](https://github.com/camaraproject/DeviceRoamingStatus), whose latest release is a candidate; the pin stays on the last stable one and the contract below is unchanged between them |
| Tag | `r2.2` (release; the default branch carries `version: wip`) |
| API version | 1.0.0 |
| Operation | `POST /retrieve`, with a `device` object in the body (HTTPS in production; see [Prerequisites](#prerequisites)) |
| Vendored at | `examples/reference/camara/device-roaming-status.yaml` |
| Blob SHA | `7a0153d2ed1d8972397698fcd298c9c060987458` |

The copy is byte-identical to upstream. Both commands print that SHA, and that is what dates this pin: a tag can be moved, a blob hash cannot.

```bash
git hash-object examples/reference/camara/device-roaming-status.yaml
gh api /repos/camaraproject/DeviceStatus/contents/code/API_definitions/device-roaming-status.yaml?ref=r2.2 --jq .sha
```

The telco simulator serves this file rather than a hand-written mock, and the tests run without reaching the network.

## Comparing like with like

The merchant's country is the last two characters of `DE 43`, as ISO 3166 alpha-2.

The operator's answer has three fields, and only one of them is guaranteed:

| Field | What it is | Always there |
|:---|:---|:---|
| `roaming` | a boolean: is the handset roaming | yes |
| `countryName` | a **list** of alpha-2 country codes | no |
| `countryCode` | a **mobile country code**, a different registry | no |

The comparison therefore has to work from either country field, and cope with neither arriving.

`countryName` is a list because one mobile country code can cover several countries: 340 is BL, GF, GP, MF and MQ. So the test is membership, not equality, and reading the first entry alone would report a mismatch for the other four. An empty list means the code maps to no country, as 901 does for the global networks.

When only `countryCode` comes back, the merchant's own country is resolved to its code through [`examples/reference/iso3166_to_mobile_country_code.yaml`](https://github.com/jaab-tech/fluxrig/blob/main/examples/reference/iso3166_to_mobile_country_code.yaml), sourced from [ITU-T E.212](https://www.itu.int/pub/T-SP-E.212B), and the two codes are compared instead. A country can hold several codes, as the US does, so that test is membership as well.

> **The acronym is taken twice.** In payments *MCC* is the Merchant Category Code, `DE 18`. In telecommunications it is the Mobile Country Code. This tutorial abbreviates neither, including in the metric labels further down, which are read by whoever builds the dashboard rather than by whoever wrote them.

## The design in one paragraph

The issuer's authorizer never learns that a telco exists. fluxrig holds the integration, spends a strict time budget on it, and writes the outcome into a private field on the leg that goes to the authorizer. The scheme's message is not modified; the two legs are different dialects with different specs, which is already how a switch is configured here.

## Where the phone number comes from

The issuer already holds the mapping from an account to a customer to a mobile number. This tutorial does not read a customer database and neither should a deployment. The institution exports a nightly file instead:

```csv
key,msisdn
b21c...9f,+59899123456
7ae0...31,+34600111222
```

**The key is not the PAN, and not a bare hash of it either.** A PAN is sixteen digits with a known BIN and a check digit, so a table mapping every valid card number to its SHA-256 is something anyone can build.

What the derived key protects is the **file**, not the message: `DE 2` is in every authorization and no design choice changes that. But a nightly export pairing card numbers with mobile numbers is a cardholder-data store that gets backed up, copied and attached to tickets, and every copy drags whatever holds it into PCI DSS scope. Keying it by something that is not a PAN keeps the export out of that category.

Two ways to derive it:

| Option | When |
|:---|:---|
| **HMAC-SHA256 with a secret** | what the scenario uses. Deterministic, so it works as a lookup key |
| **Deterministic encryption (AES-SIV, or FPE where the format must survive)** | when the institution needs to reverse it |

Determinism is the requirement, which rules out ordinary encryption: AES-GCM produces a different ciphertext each time by design, which is what confidentiality wants and exactly what breaks a lookup key.

### Where the secret lives

**In this tutorial it is an environment variable, which is a simplification.** Right for getting it running, wrong for production: anyone who can read the process environment can read the key.

**In a real deployment the answer belongs to the bank**, and there is no single correct one. A payment HSM, a secret manager, a platform root of trust: which fits depends on an institution's existing controls and audit obligations, not on which is better.

**fluxrig fits all of them, because none is a change to fluxrig.** A key fetched from a vault over REST is an HTTP call in a pipeline, cached so it happens once per TTL window rather than once per transaction. Only an HSM reached over a host command port needs a gear rather than configuration.

Three things hold whichever is chosen.

**It protects the file, not the host.** An attacker on the Rack has live authorizations crossing it, PANs included, and no need of the export. The derived key narrows the blast radius of a leak; it is not a control against host compromise.

**Never in the scenario file.** The Mixer stores scenarios in its database and distributes them over the bus, so a literal secret in a scenario is a secret in the control plane and on the wire. Bento's secret scrubbing does not cover values used inside a Bloblang mapping, which is where this one is used.

**Rotation belongs to the file.** The export names the key it was derived with and the Rack resolves that identifier, so publishing a snapshot derived with a new key is the whole operation. A Rack that instead refreshes on a timer will eventually hold a key the active file was not derived with, and the failure is silent: every lookup misses, so every transaction looks like a card that was never enrolled.

### Where the file lives, and how big it gets

In memory on the Rack, not in its key-value store: the Snake backing that runs inside the Mixer, so a lookup there is a round trip to the control plane on every authorization.

| Cards | Snapshot on disk | In a Bento memory cache |
|---:|---:|---:|
| 100 thousand | 4 MB | 15 MB |
| 1 million | 45 MB | 153 MB |
| 5 million | 224 MB | 763 MB |
| 10 million | 448 MB | 1 GB |
| 70 million | 3 GB | 10 GB |

The second column is what a configuration-only deployment costs, because Bento's memory cache is a sharded map of strings. That puts the ceiling between one and five million cards. Above it, `cache_multilevel` puts a memory cache in front of the key-value store, read through, so only the cold tail pays the round trip.

The key is truncated to 128 bits, which halves the largest column. Not further: at 64 bits the chance that two of 10 million cards collide is 2.7e-06, and a collision here is not a miss but another customer's number.

The number is read, used for the call, and dropped. What continues to the authorizer is the outcome.

## The scenario

Ten gears, and the whole thing is [`examples/scenarios/roaming_enrichment.yaml`](https://github.com/jaab-tech/fluxrig/blob/main/examples/scenarios/roaming_enrichment.yaml). Nothing below is compiled.

| Gear | Type | Does |
|:---|:---|:---|
| `scheme-in` | `io_iso8583` | listens for the scheme, and writes the reply back on the same connection |
| `decode-scheme` | `codec_iso8583` | scheme dialect to fields |
| `enrich-roaming` | `bento` | the lookup, the call and the policy |
| `in-flight-daemon` | `coatcheck` | owns the correlation bucket and expires its entries |
| `stamp-request` | `coatcheck` | parks the arrival time under the transaction key |
| `encode-issuer` | `codec_iso8583` | issuer dialect, which declares the private field |
| `authorizer-out` | `io_iso8583` | dials the authorizer |
| `decode-issuer` | `codec_iso8583` | parses the reply, only so the key can be read |
| `match-reply` | `coatcheck` | hands the stamp back |
| `measure-authorizer` | `bento` | emits the round trip |

### The enrichment, in four steps

One `bento` gear does all of it. The number leaves this gear once, in the request to the operator, and never travels onward: it is held in metadata rather than in the message body, so it is not serialised to the authorizer or the scheme, and it is deleted before the gear emits.

**1. The enrolled number**, under a key derived from the PAN and truncated to 128 bits:

```yaml
- branch:
    request_map: 'root = this.iso8583.field."2".hash("hmac_sha256", env("ENROLMENT_HMAC_KEY")).encode("hex").slice(0, 32)'
    processors:
      - cache: {resource: enrolment, operator: get, key: '${! content() }'}
    result_map: 'meta msisdn = content().string()'
```

A card that is not enrolled misses here, and everything after it is skipped.

**2. Ask the operator**, inside the time budget, and give up on the signal rather than on the payment:

```yaml
- try:
    - branch:
        request_map: 'root.device.phoneNumber = meta("msisdn")'
        processors:
          - http: {url: '${ROAMING_URL:http://127.0.0.1:9500/retrieve}', verb: POST, timeout: 250ms, retries: 0}
        result_map: |
          meta device_alpha2 = (this.countryName | []).join(",")
          meta device_mcc = (this.countryCode | "").string()
          meta no_country = if this.exists("countryName") && (this.countryName | []).length() == 0 { "yes" } else { "" }
```

`retries: 0` is deliberate: a retry inside a fixed time budget is a second way to run out of it. `try` means a failure here leaves the message untouched rather than failing it.

**3. Fall back to codes**, only when there is a code and it maps to somewhere:

```yaml
- branch:
    request_map: 'root = if meta("device_alpha2").or("") == "" && meta("device_mcc").or("") != "" && meta("no_country").or("") == "" { this.iso8583.field."43".string().slice(-2).uppercase() } else { deleted() }'
    processors:
      - cache: {resource: iso3166_to_mobile_country_code, operator: get, key: '${! content() }'}
    result_map: 'meta merchant_mcc = content().string()'
```

All three conditions matter. Without the check on `no_country`, an empty country list would be compared as if it named a country; without the check on `device_mcc`, a failed operator call would look one up under an empty key and mark the message as errored.

**4. Decide, and drop the number:**

```yaml
- mapping: |
    let device = meta("device_alpha2").or("")
    let merchant = this.iso8583.field."43".string().slice(-2).uppercase()
    root = this
    let outcome = if $device != "" {
        if $device.split(",").contains($merchant) { "match" } else { "mismatch" }
      } else if meta("device_mcc").or("") != "" && meta("merchant_mcc").or("") != "" {
        if meta("merchant_mcc").split(",").contains(meta("device_mcc")) { "match" } else { "mismatch" }
      } else if meta("no_country").or("") == "yes" { "no_country"
      } else if meta("op_status").or("") == "404" { "no_subscriber"
      } else if meta("op_status").or("") == "503" { "operator_down"
      } else { "no_answer" }
    root.risk.roaming = $outcome
    root.risk.roaming_code = "RS" + match $outcome {
        "match" => "00", "mismatch" => "01", "no_subscriber" => "10",
        "operator_down" => "11", "no_answer" => "12", "no_country" => "13", _ => "19",
      }
    meta msisdn = deleted()
```

The gear runs at `log_level: WARN`, because Bento logs message context at INFO on some paths and the number passes through here.

### What happens when a message will not decode

Every codec here runs `on_error: "reject"`, which is not the default. The default
is `drop`: a message that fails to decode is filtered out, and the sender waits
for a reply that will never come until it times out.

`reject` returns the message with its error instead, so the failure is attributed
to the gear that had it and is counted in `flux.gear.errors` rather than
disappearing. On a switch that matters more than on a pipeline: a message
silently dropped mid-flight is an authorization nobody answered, and the first
sign of it is a scheme's timeout graph rather than anything in your own
telemetry.

Both settings log the failure with the spec hash, which is usually the answer:
a decode that fails on every message means the spec and the wire disagree, and
the hash says which spec was loaded.

### The specs, and why there are two

Two SDL specs differing in one field: the issuer's declares `DE 48`, aliased to `risk.roaming_code`, and the scheme's does not. That single difference is why the reply needs no re-encoding, and why the outcome cannot reach the scheme even by accident: its dialect has nowhere to put it.

### The telco simulator

The operator endpoint, and the only counterparty played by fluxrig rather than by `iso8583-tool`: a `bento` gear with an `http_server` input, serving the vendored contract. It selects its behaviour from the last two digits of the number, so a test asks for a case by dialling it: a country list with several entries, an empty list, a reply with the country fields absent, one that arrives after the time budget, and a `503`.

## How a transaction flows

The authorization completes in every case. Only a match and a mismatch carry a usable signal.

| What happened | `DE 48` carries |
|:---|:---|
| Handset in the merchant's country | `RS00` |
| Handset somewhere else | `RS01` |
| The operator does not know the number | `RS10` |
| The operator answered with a failure | `RS11` |
| No answer inside the time budget | `RS12` |
| The code maps to no country | `RS13` |
| The card is not in the enrolment file | nothing: the field is absent and the message is otherwise untouched |
| None of the above | `RS19`, which should never be seen. The mapping has a catch-all so an outcome added in the pipeline and forgotten here reaches the authorizer as an unknown code rather than as a mapping failure that drops the payment |

**A code, not a word.** A private ISO 8583 field carries compact data, so `match` and `no_subscriber` are what the pipeline calls things internally and what the metrics are labelled with; what goes on the wire is four characters.

`RS` tags the value, so a second enrichment can be appended later without moving this one. The two digits are grouped: **`0x` means the network answered, `1x` means it did not.** An authorizer that only wants to know whether the countries disagree tests one character; one that wants the reason reads both. That is deliberate, because the first kind is a change most authorizers can absorb and the second is a project.

Every row records its outcome and how long the operator took, except the unenrolled card, which records neither because no call is made, and that absence is the measure of coverage: not every card has a usable number on file, and the enrichment is expected to skip those.

**Four ways to have no signal, and they are not the same thing.** A slow operator, an operator returning failures, a subscriber the network does not recognise and a code that maps to nowhere are four different problems, and collapsing them into one value would make them indistinguishable exactly when someone needs to tell them apart.

They are distinguished at the source: the operator call treats `404` and `503` as answers rather than failures (`successful_on: [200, 404, 503]`), so the status is readable instead of being swallowed as a caught error. A request that produces no status at all did not get an answer inside the time budget.

`no_subscriber` is worth separating for a second reason: it is not only an operational fact. A card enrolled against a number the network does not recognise is itself mildly interesting, and an authorizer may want to treat it differently from an operator that happens to be down.

### What the reply does

Nothing to the message. The authorizer's response is relayed to the scheme exactly as it arrived, with no re-encode: nothing on the way back modifies it, so packing it again would spend a full round of work per transaction to produce identical bytes.

It is decoded, but only so the Coat Check can read the fields it correlates on, and the decode leaves the arriving bytes untouched. The private field never comes back: the authorizer consumed it and does not echo it, which is what keeps the outcome away from the scheme, whose dialect does not declare that field at all.

## What you get to watch

Every Rack embeds OpenTelemetry, and the Mixer's sink writes what arrives into DuckDB and flushes it to Parquet. There is no collector to run and nothing to install beside it.

### What is already emitted

These exist for every gear in the scenario, with no configuration:

| Metric | What it tells you here |
|:---|:---|
| `flux.gear.messages_in` / `_out` | throughput per gear, and where a message stopped |
| `flux.gear.processing_time_ms` | **per-gear latency**, which is how you see the enrichment's cost separately from the codec's |
| `flux.gear.errors` | failures, attributed to the gear that had them |
| `flux.codec.iso8583.fields_count` | field counts per message, which catches a spec drifting from the wire |

These are infrastructure metrics: they describe the pipeline, not the business. The two sections below cover what the transactions themselves say.

Spans carry `trace_id`, `parent_span_id`, timings and status, so a single authorization can be followed across every gear it passed through.

### The business metrics

The message carries them and the codec has already decoded it, so a `metric` processor emits them as counters. Labels come from fields:

```yaml
- metric:
    type: counter
    name: authorization_result
    labels:
      merchant_category: '${! json("iso8583.field.18").catch("") }'   # DE 18, not a mobile country code
      entry_mode: '${! json("iso8583.field.22").catch("") }'          # chip, contactless, keyed
      acceptor_country: '${! json("iso8583.field.43").string().slice(-2).uppercase().catch("") }'
```

Every label is wrapped in `.catch("")`, because a field that is optional in some dialect would otherwise fail the interpolation and take the message with it: a metric must never be able to drop a payment.

**The response code is not there, and cannot be.** `DE 39` exists on the reply and nowhere else, while this counter runs on the way out. The decline rate is counted on the reply path instead, by the same gear that measures the round trip, which already has the message parsed:

```yaml
- metric:
    type: counter
    name: authorization_outcome
    labels:
      response_code: '${! json("iso8583.field.39").catch("") }'
```

Between the two, an issuer gets much of what it asks of its own traffic: volume by category, whether contactless declines more than chip, which acquirer country a problem belongs to. All of it live, on the wire, rather than aggregated overnight by the core.

**Response times, which is usually the first thing asked for.** Two sources, and they answer different questions.

Per-gear latency comes from spans, one row per span with its own start and end, so percentiles are exact rather than bucketed:

```sql
SELECT name,
       count(*)                                             AS calls,
       avg(epoch_ms(end_time) - epoch_ms(start_time))       AS avg_ms,
       quantile_cont(epoch_ms(end_time) - epoch_ms(start_time), 0.50) AS p50,
       quantile_cont(epoch_ms(end_time) - epoch_ms(start_time), 0.95) AS p95,
       quantile_cont(epoch_ms(end_time) - epoch_ms(start_time), 0.99) AS p99
FROM telemetry_spans
GROUP BY name ORDER BY p99 DESC;
```

That is where a slow leg shows itself: the operator call, the codecs and the enrichment each appear separately, and the tail is what matters, because a mean stays comfortable long after the p99 has stopped fitting the time budget.

The round trip to the authorizer is the other number, and it is a `timing`, which reaches the store as a sum and a count per collection interval:

```sql
SELECT date_trunc('minute', timestamp) AS minute,
       sum(CASE WHEN name LIKE '%.sum'   THEN value END)
     / sum(CASE WHEN name LIKE '%.count' THEN value END) / 1e6 AS avg_ms
FROM telemetry_metrics
WHERE name LIKE 'bento.authorizer_round_trip_ns.%'
GROUP BY 1 ORDER BY 1;
```

**That gives an average, not percentiles.** A histogram's bucket counts are not exported today, only its sum and count, so a p99 of the authorizer round trip cannot be computed from it. Per-gear percentiles from spans cover most of what that question is really after, since no single gear sees both halves of a transaction. Exporting the buckets is [Roadmap].

**Do not label by acceptor.** `DE 42` and `DE 43` identify individual merchants, and a counter labelled by them grows a time series per merchant, which is how a metrics store falls over. Merchant-level analysis belongs in the Parquet, where a `GROUP BY` costs nothing and cardinality is not a problem. The rule generalises: bounded fields become labels, unbounded ones stay columns.

### Emitting the outcome

The decision itself is not a built-in metric, and it does not need Go. Bento has a `metric` processor, so the enrichment pipeline counts its own outcomes as it produces them:

```yaml
- metric:
    type: counter
    name: risk_roaming_outcome
    labels:
      outcome: '${! json("risk.roaming") }'
```

A `timing` named `operator_call_ns` measures the call itself, stamped either side of it, which is what makes the time budget measurable rather than aspirational.

**About that 250ms.** It is a starting point, not a recommendation. Published latency figures for network APIs generally come from vendor material rather than from operator service levels, and they vary by operator, by aggregator and by country. Treat the number in this scenario as a placeholder to be replaced by a measurement against the aggregator you actually use. The timing above is how you get that measurement; until you have it, the time budget is a guess with a unit attached.

### Measuring the authorizer, which needs correlation

Per-gear timings do not give the round trip: the request and the reply are different messages travelling different gears, and no gear sees both. The **Coat Check** pair parks the arrival stamp under a transaction key on the way out and hands it back on the matching reply. One small gear on the reply path subtracts the two and emits `authorizer_round_trip_ns`, guarded so an unmatched reply skips the measurement rather than failing the message.

**The pair needs a third gear.** A correlation bucket is not created on demand, because expiring entries needs one owner, so a `coatcheck` in `daemon` mode declares the bucket and sweeps it. Without it the first store fails with `bucket not found`, which is at least loud, unlike the two settings below.

The key is `DE 41` and `DE 11`, terminal and trace number, which is the pairing widely used as a default across ISO 8583 switch implementations. `DE 11` is six digits and wraps, so it cannot stand alone; a terminal scopes it better than a timestamp, since two terminals can produce the same trace number in the same second while one will not reuse it for a long while.

Two details of how the key is built fail quietly rather than loudly.

**Both sides must render the value the same way.** They are decoded against different specs, so the same trace number can arrive zero-padded on one side and trimmed on the other. The key is a join, so `000123` and `123` are different keys and the measurement simply stops happening. `key_normalize` settles it: `trim` (the default) removes surrounding whitespace; this scenario uses `numeric`, which also drops leading zeros, because both fields are fixed-width and the two specs need not declare the same width.

**The message class belongs in the key.** The full MTI cannot, since a request and its reply carry different ones. The leading digits can, and the codec exposes them as `iso8583.mti_class`. Without it, an authorization and a reversal reusing a trace number are indistinguishable in the store.

**And the restore must overwrite.** What is parked is not only the timestamp: it is also the connection the request arrived on. The reply is a different message, returning over the link to the authorizer, and it already carries a connection identifier of its own. Restoring without `merge_strategy: overwrite` preserves that one, so the switch has an answer and writes it into the wrong socket. The symptom is a connection that does not exist, several gears away from the setting that caused it.

The store runs with `await_store: false`, so the authorization does not wait for it. The entry exists only to measure the authorizer, and blocking a payment on a write to the control plane, or failing it when that write fails, trades a payment for a metric. Losing an entry loses a measurement, which is the right thing to lose. The default is the opposite, because the gear's original use reattaches a stripped field to the reply, and there forwarding before the entry exists produces a reply that can never be made whole.

An unmatched reply is forwarded rather than dropped (`on_missing: forward`), and the count of them is worth watching: a rising unmatched rate usually means the key is wrong rather than that replies are missing.

**But forwarding does not rescue an expired entry, and the TTL is therefore a ceiling on the authorizer's latency.** The connection the request arrived on is parked in that entry along with the timestamp. Once the sweeper has been through, the reply still crosses the pipeline and is then dropped at the last gear, because nothing knows where to send it. `on_missing: forward` forwards it into a dead end.

So the TTL is not only a memory bound. Set it below the authorizer's worst case and those transactions are lost, and lost late: the switch logs a connection it cannot find, while the scheme sees a timeout. Give it comfortable headroom over the slowest reply you are willing to wait for, and alert on that log line, which is the only local evidence that it happened.

The scenario sets `default_ttl: "30s"` on the `coatcheck` in `daemon` mode, which owns the bucket:

```yaml
- name: "in-flight-daemon"
  type: "coatcheck"
  config:
    mode: "daemon"
    bucket: "auth_in_flight"
    storage: "memory"
    replicas: 1
    default_ttl: "30s"
```

Thirty seconds is generous against an authorizer that answers in milliseconds, and it is meant to be: the cost of being generous is memory, and the cost of being tight is lost authorizations.

### Where it lands, and how to look at it

Three tables on the Mixer, and the same three flushed to Parquet.

| Table | One row per | Columns |
|:---|:---|:---|
| `telemetry_metrics` | metric observation | `timestamp, entity_id, entity_name, name, description, type, value, unit, attributes` |
| `telemetry_spans` | span | `start_time, end_time, entity_id, entity_name, trace_id, span_id, parent_span_id, name, kind, status_code, status_message, attributes` |
| `telemetry_logs` | log record | timestamp, severity, body and attributes |

`entity_name` is the Rack, and `attributes` is JSON, so a label is `json_extract_string(attributes, '$.outcome')`.

**What this scenario puts there.** Metrics declared in the pipeline arrive prefixed with `bento.`, and a `timing` arrives as two rows per interval rather than one, because a histogram is exported as its sum and its count:

| `name` | From | `attributes` |
|:---|:---|:---|
| `bento.risk_roaming_outcome` | the outcome counter | `{"outcome": "match"}` |
| `bento.operator_call_ns.sum` / `.count` | the operator timing | |
| `bento.authorizer_round_trip_ns.sum` / `.count` | the coat check pair | |
| `flux.gear.processing_time_ms` | every gear, with no configuration | `{"flux.name": "enrich-roaming"}` |
| `flux.gear.messages_in` / `_out` | every gear | `{"flux.name": "scheme-in"}` |

Outcomes over an hour:

```sql
SELECT json_extract_string(attributes, '$.outcome') AS outcome, sum(value) AS n
FROM telemetry_metrics
WHERE name = 'bento.risk_roaming_outcome'
  AND timestamp > now() - INTERVAL 1 HOUR
GROUP BY 1 ORDER BY n DESC;
```

Which gear is spending the time, from spans, since those carry their own start and end:

```sql
SELECT name, count(*) AS calls,
       quantile_cont(epoch_ms(end_time) - epoch_ms(start_time), 0.99) AS p99_ms
FROM telemetry_spans GROUP BY name ORDER BY p99_ms DESC;
```

**On disk**, under the Mixer's data directory:

```
telemetry/metrics/2026/08/25/14/metrics_1756134000000000000.parquet
telemetry/spans/2026/08/25/14/spans_1756134000000000000.parquet
telemetry/logs/2026/08/25/14/logs_1756134000000000000.parquet
```

One directory per table, partitioned by year, month, day and hour, zstd compressed, same columns as the tables. That layout is the point: a glob is a partition prune, so a query over one afternoon reads one afternoon.

```sql
SELECT * FROM 'telemetry/metrics/2026/08/25/**/*.parquet' WHERE name LIKE 'bento.%';
```

Which means the visualisation options are the ones already available. Point a notebook at the directory. Connect Metabase or Superset through a DuckDB driver. Load it into whatever warehouse the institution runs, since Parquet is what warehouses ingest. Nothing here is a fluxrig-shaped format needing a fluxrig-shaped viewer.

**What is not there today:** fluxrig does not push to an external OTLP collector, so pointing a Rack at an existing Grafana or Tempo is not a matter of configuration. The telemetry reaches you through DuckDB and Parquet, and the bridge to a collector is [Roadmap].

### The alerts worth having

Three, and none of them is the mismatch rate:

- **Answer rate below a floor.** A signal that stops arriving degrades silently: every authorization still completes, carrying one of the no-signal values, and the control quietly stops existing. Which value it is names the cause, so the alert points somewhere.
- **Latency tail past the time budget.** The mean will look fine long after the p99 has stopped fitting.
- **Enrolment snapshot older than expected.** If the file stops arriving nothing breaks: the snapshot goes stale, lookups miss, and transactions pass as though those cards had never been enrolled. Watching the volume of enriched transactions will not catch it, because a stalled feed and a quiet night produce the same graph. The age of the snapshot separates them, so alarm on that.

The mismatch rate is the interesting number, but it is a business signal, not an operational one. Alert on the three above; watch the mismatch rate.

### Shadow mode first

Nothing above requires acting on the signal. Run this scenario with the authorizer ignoring the private field and an institution gets a view of a fraud signal, and of its own authorization traffic, before anyone has to decide to change an outcome. It is the easiest version of this to get approved, and the data it produces is the argument for the next step.

## Validation

The result below is the last run, published from the Robot output rather than transcribed. Failing runs publish the same way as passing ones: a report that only appeared when it was green would say nothing about whether the suite is green.

<RobotReport suite="roaming" title="make test-robot-roaming" />

The suite asserts degradation, not success:

| Case | Expected |
|:---|:---|
| Handset in the merchant's country | `match`, authorizer receives the field |
| Handset elsewhere | `mismatch` |
| Operator slower than the time budget | `no_answer`, and the authorization costs about the time budget more than a fast one rather than the operator's whole delay |
| Operator returns an error | `operator_down` |
| Subscriber unknown to the operator | `no_subscriber` |
| Card not in the nightly file | message passes through unchanged |
| Every case | MSISDN absent from the outgoing message and from all logs |
| Operator returns several countries for one code | `match` when the merchant's country is any of them |
| Operator returns an empty country list | `no_country`, not `mismatch` |
| Every response the telco simulator serves | conforms to the vendored `r2.2` definition |
| Every reply at all | the correlation matched. A broken key restores no `conn.id`, so the reply is written to no connection and every case times out: the suite cannot pass with the correlation broken |
| A reversal reusing an authorization's trace number | each connection gets its own message class back. Both are sent before either reply is read, so the two contexts are parked at once by construction rather than by timing |
| A reply arriving after its entry expired | the transaction is lost, and the switch says so. The connection is parked in that entry, so `on_missing: forward` has nowhere to forward to; what is asserted is that the loss reaches a log rather than only the scheme's timeout graph |

### Under load

The cases above run one transaction at a time. Correlation, though, is a
concurrency property: the enrichment parks each message's context under a key and
restores it on the reply, and a key that collides returns the wrong reply to the
wrong terminal. A single-file suite cannot see that. This one drives the same
scenario with a fleet of terminals at once.

<RobotReport suite="roaming-stress" title="make test-robot-roaming-stress" />

Four cases, all against the shipped scenario with `await_store: false`, the
detached-write path that spawns one goroutine per message:

| Case | Shape | What it proves |
|:---|:---|:---|
| Sustained load, match outcome | 4 connections, 200 TPS for 15s | The match path stays correlated at a steady rate |
| Concurrent multi-outcome | 7 terminals, 4 connections each, 60 TPS each | Every telco outcome runs at once and each reply still finds its terminal |
| High concurrency | 7 terminals, 6 connections each, 120 TPS each | Peak goroutine pressure on the detached write, correlation intact |
| Mixed message classes | Authorizations and reversals over one trace range | Both classes flow together without failures |

**The shape is rate, not sockets.** An issuer talks to a scheme over a handful of
long-lived links, not the hundreds of connections a switch sees from a terminal
estate, so the pressure worth applying in this position is transactions per
second. A few connections carrying a high rate reproduces what the correlation
store actually has to survive here; many idle ones would not.

Every case asserts `cross_wired == 0`: not one reply reached a terminal that did
not send it. The fleet fails if any terminal does not finish, so a stalled
terminal cannot pass as zero work.

**What it happened to sustain, and why that is not a benchmark.** The suite
asserts correctness, not capacity, but it does record what the run did:

| Case | Transactions | Achieved TPS |
|:---|---:|---:|
| Concurrent multi-outcome | 7,000 | ~780 |
| High concurrency | 12,600 | 784 |

Latency is reported per path rather than as one figure, because the mix is
deliberately bimodal and a single number across it describes no transaction that
actually happened. Six of the seven cards get an answer; the seventh waits out
the budget on every request:

| Path | p50 | p99 |
|:---|---:|---:|
| Operator answers | 31ms | ~100ms |
| Operator does not answer in time | 261ms | 443ms |

An average across those two would land somewhere near 60ms and mean nothing. The
first row is what the pipeline costs: decode, a cache read, an HTTP call, the
policy, re-encode. The second is the 250ms budget plus the same work, which is
the design refusing to wait rather than a slow path.

That is one Linux developer desktop, an i7-6800K with 12 threads and 46 GB,
running **everything at once**: the Mixer, both Racks, the simulated authorizer
and all forty-two terminal processes, over loopback with no TLS and no network
between them. It was also running a full GNOME session throughout, which is to
say a browser, an editor and a compositor competing for the same cores. A
deployment separates all of that onto hardware that does one job each, so the
number is not transferable in either direction, though the direction it errs in
here is downward.

The aggregate rate carries the same caveat. Only one of the seven cards is slow,
and it is slow on purpose: the other no-signal outcomes answer immediately, with a
`404`, a `503` or an empty country list. That one terminal runs at a sixth of the
others' rate and drags the total down, so the figure understates what the pipeline
does with an operator that answers.

And nothing here looked for a limit: each case sets a target rate and the
generator met it. Treat the figure as a floor observed while proving something
else, and measure your own before quoting one.

**What load cannot prove.** Two properties of the correlation key are
deterministic, and a load suite is the wrong instrument for both. A reversal
reusing an authorization's trace number only collides while both are parked at
once, and two fleets walking the same trace sequence at the same rate simply stay
offset from each other, so they never are. The same goes for an entry expiring
before its reply arrives: it needs a reply slower than the TTL, not more traffic.

Both are in the functional suite instead, which constructs the situation rather
than waiting for it: one keyword sends every message before reading any reply, so
the contexts are parked simultaneously by construction. Removing the message class
from the key fails that case and no other, which is the only evidence that the
class is doing work.

## Current status

| Piece | Status |
|:---|:---|
| `io_iso8583`, `codec_iso8583` | Available |
| `bento` gear (lookup, HTTP call, policy) | Available (requires the full Rack build) |
| Memory and multilevel caches for the enrolment snapshot | Available |
| Private-field outcome on the issuer leg | Available (SDL) |
| This tutorial's scenario and suite | Available, and green |
| This scenario under load | Measured. The [stress suite](#under-load) drives the enrichment with a fleet of terminals at up to 60-way concurrency each, with the store detached, and every reply returns to the terminal that sent it: zero cross-wiring. The switch underneath is measured separately by the [payment switch suite](payment_switch_conductor.md#non-functional-load-and-chaos) |
| Per-destination circuit breaking on the operator call | Not built here. It is a policy in front of an HTTP call, which is where the rest of this scenario already lives, so it is configuration rather than a new component |

## Related

- [Mobile network signals in payments](../use_cases/mobile_network_signals.md): why this pattern exists, and what GSMA Open Gateway is.
- [Building a multi-region payment switch](payment_switch_conductor.md): the routing layer this composes with.
