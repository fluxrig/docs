---
slug: /use_cases/mobile-network-signals
title: Mobile network signals in payments
---

# Mobile network signals in payments

## What this is about

A mobile operator knows things about a phone number that no payment system can see: whether the SIM was replaced last night, whether the handset is switched on, and which country it is in right now. Payments and telecommunications have been adjacent for years without those facts crossing between them, because there was no common way to ask.

[GSMA Open Gateway](https://www.gsma.com/solutions-and-impact/gsma-open-gateway/) is the industry initiative that changed that, and [CAMARA](https://camaraproject.org/) is the open-source project, run under the Linux Foundation with the GSMA and the TM Forum, that defines the APIs themselves. The point of both is that one integration works across operators instead of one per operator, per country.

For a bank the appeal is direct. Most account takeover starts with control of a phone: a SIM is swapped, the one-time code arrives at the attacker's handset, and the transaction that follows looks perfectly ordinary from inside the payment network. The operator can see the swap. The payment system cannot.

The adoption is real and is measurable. One published deployment among several: Itaú Unibanco, in Brazil, had processed the SIM Swap API **more than 36.5 million times by April 2025, averaging five million checks a month**, according to its [GSMA case study](https://www.gsma.com/solutions-and-impact/gsma-open-gateway/wp-content/uploads/2025/09/Itau-Unibanco-Case-Study.pdf). GSMA's [case study library](https://www.gsma.com/solutions-and-impact/gsma-open-gateway/gsma-open-gateway-case-studies/) carries others across banks and digital lenders, several anonymised at the institution's request, which suggests the published record understates how many exist.

One figure from that library is worth carrying into the design rather than leaving as background. A UK bank running SIM Swap checks at scale reported that in March 2024, of 3.1 million calls, 87% found no swap, 1% were flagged, and [**around 12% came back `unknown`**](https://www.gsma.com/solutions-and-impact/gsma-open-gateway/wp-content/uploads/2025/08/UK-Bank-Case-Study.pdf), typically because a number was inaccessible or invalid. Roughly one call in eight produced no answer at all. Any design that treats "no signal" as an edge case is wrong about its own traffic.

Latin America has been among the faster regions to deploy these APIs.

CAMARA is careful about what it is offering: these are **technical network signals, not risk scores**, and the specification says explicitly that the consumer applies its own decision outside the API contract. That gap is where this use case lives.

---

## The signals a network can answer

The APIs relevant to a financial institution are a small, stable set, and they differ mainly in the timescale of the question they answer.

| API | What it answers | Timescale | Released | Maturity |
|:---|:---|:---|:---|:---|
| [SIM Swap](https://github.com/camaraproject/SimSwap) | did the SIM behind this number change? | hours to days | **2.1.0** | Incubating |
| [Device Swap](https://github.com/camaraproject/DeviceSwap) | did the handset change? | hours to days | **1.0.0** | Incubating |
| [Device Roaming Status](https://github.com/camaraproject/DeviceRoamingStatus) | is the handset roaming, and in which countries? | right now | 1.2.0-rc.3 | Incubating |
| [Tenure](https://github.com/camaraproject/Tenure) | has the line been continuously active since a date? | months to years | 0.3.0-rc.1 | Sandbox |
| [Number Recycling](https://github.com/camaraproject/NumberRecycling) | has this number changed owner since a date? | months to years | 0.2.0 | Sandbox |

Versions and maturity are as published by **August 2026**, and both move: the table is a reading of the repositories on a date, not a standing fact.

The answers are shaped alike: a timestamp, or a boolean over a window you specify. Roaming is the exception, returning a flag, a mobile country code and a *list* of countries.

**Two of the five were on a stable version, and that is the column to read first.** SIM Swap and Device Swap have released `2.1.0` and `1.0.0`; the other three are release candidates or pre-1.0, which is a different proposition regardless of what the maturity label says. Roaming is the case worked through in the tutorial, and its contract has been stable in the part that matters even while the version number moves.

The two labels mean different things and both matter. CAMARA marks a repository **Sandbox** while an API is still being shaped and **Incubating** once it is stabilising. A Sandbox API on a pre-1.0 version is early twice over.

Each link goes to the API's own repository, which holds the OpenAPI definition, its release tags and its issue history. That is the source to read before designing against any of them, and to read at a **release tag**: CAMARA develops in the open and default branches carry work in progress.

Both columns move. Roaming has already changed address once, from the combined `DeviceStatus` repository to its own, which is the sort of thing that makes a link go quiet rather than break.

None of them returns a verdict, and that boundary is deliberate. "The SIM changed eleven hours ago" is a fact. Whether that fact should decline a transaction, trigger a step-up or do nothing at all depends on the amount, the merchant, the customer and the institution's own appetite. CAMARA draws the line there on purpose, and everything on the far side of it belongs to whoever runs the payment system.

In practice they get combined, because one signal alone rarely says enough:

- **Onboarding:** Tenure, then SIM or Device Swap, then Number Recycling. Is this line old enough to mean something, has it moved recently, and did it belong to someone else last year?
- **A payment in flight:** Device Swap, then SIM Swap. Anything that changed in the last hours is more interesting than anything that changed last year, because the time allowed only stretches to a short question.
- **A password reset:** SIM Swap, then Number Recycling. The two ways a recovery code reaches the wrong handset.

## How a bank reaches an operator

The published material can look like it says two different things here, and which one applies decides where the work goes.

CAMARA secures every API with OpenID Connect and mandates **three-legged authorization for any API processing personal data**, which SIM Swap, Device Swap, Number Verification and KYC Match all are. The flow is CIBA: post to `/bc-authorize` with a hint identifying the subscriber and a purpose-scoped scope, receive a request id, then poll `/token` on roughly a five second interval while an out-of-band step may involve the subscriber acting on their own phone.

Nothing about that fits inside an authorization. Yet every published deployment describes these checks as real time, per transaction.

Both are true, and they describe different paths.

| Path | What the caller presents | Viable inline? |
|:---|:---|:---|
| Direct to each operator, plain CAMARA | three-legged CIBA, then polling | no |
| Through an aggregator | an API key or token with scopes, one fast call | yes, and this is the path the published deployments describe |

Itaú's five million checks a month go through an aggregator that federated the Brazilian operators behind a single contract. The aggregator does not remove the three-legged flow. It moves it to the other side of a boundary, where it happens **once per subscriber rather than once per transaction**.

For an institution the consequences are concrete. The counterparty is the aggregator, not the operator. The credential is an ordinary API key. Token and consent lifecycle against each network is somebody else's problem, on the path that matters. And the integration that has to be built is an HTTP call with a deadline, which is a much smaller thing than the specification first suggests.

It is worth being clear about what that leaves. The call itself is an ordinary HTTP request, and the aggregator has already done the work of making it one. What remains is what happens in the milliseconds around it: the deadline, the fallback, and the record of what was decided.

## Why the authorizer is the constraint

A bank's authorizer is typically the oldest system it runs: a mainframe or a licensed package, changed through a release train measured in quarters, and a time budget for each authorization that nobody is permitted to spend. Adding an outbound call to a third party inside that path raises a change request, a security review and a vendor conversation before it raises a line of code.

So the work moves in front of it. fluxrig sits between the scheme and the authorizer, holds the network integration, and hands the authorizer a value in a private ISO 8583 field. Reading one more field is a change an authorizer can absorb, because carrying proprietary data in those fields is what these messages have done for decades.

## What every one of these integrations has to solve

Three problems, and none of them is the API call.

**The identifier does not match.** Every CAMARA API is keyed on a phone number. No payment message carries one: an authorization identifies a card, and the link between that card and a subscriber lives in a customer database that has no business being reachable from the authorization path. Bridging the two, without opening that database and without the number travelling onward, is where most of the design effort goes. It is worth settling early, because a proposal can look complete without having addressed it.

**The time budget is fixed, and not by you.** A card scheme allows an issuer a fixed number of milliseconds to answer an authorization, and exceeding it is a timeout at the scheme, which is worse for the cardholder than any answer the enrichment could have produced.

The network call spends part of that allowance, and the allowance was not sized with it in mind. That makes the timeout a design parameter rather than a configuration afterthought, and it makes retries a decision rather than a default: a retry inside a time budget is simply a second way to run out of it.

**The answer will sometimes not arrive, and not rarely.** Operators have outages, aggregators rate-limit, and not every customer has a usable number on file. The 12% figure above is a useful one to size against: at that rate the no-answer path carries a substantial share of the traffic rather than a handful of edge cases. An enrichment that fails has to leave the payment exactly as it found it and say so in a way that is visible afterwards, so "no signal" is best treated as a declared outcome rather than as an error or as silence.

Two properties hold for the whole family:

**It is configuration, not development.** The lookup, the outbound call, the timeout, the fallback and the comparison are expressed in a scenario and a few pipelines, using gears that ship with fluxrig. No component is written for it, which means the thing to review is a file rather than a codebase.

**The dialect is not the design, but it is not free either.** A reader looking at ISO 8583 in 2026 reasonably asks whether this is already legacy. The shape of the pattern does not depend on the answer: read a value the message already carries, obtain another under a deadline, write a verdict somewhere the decision-maker reads. That transfers to any message format.

What does not transfer for free is the wiring. The comparison reads a specific field by a specific path, and the outcome is written to a private field the issuer's spec declares. Moving to a different dialect means new specs and new paths, which is configuration rather than code, but it is not nothing. What survives untouched is the part that took the work: the time budget, the four named reasons for having no signal, the correlation, and the degradation.

**The telco part is the replaceable half.** Underneath, the pattern is *enrich a transaction with an external answer, inside a fixed time budget, and decide what happens when it does not arrive*. The same wiring covers a sanctions screen, an internal fraud score, a device fingerprint or a limit held in another system. Swap the endpoint and the comparison; the time budget, the fallback and the audit stay where they are.

> **A worked example.** [Enriching an authorization with a network signal](../tutorials/roaming_enrichment.md) builds one of these end to end, comparing the country of the handset against the country of the merchant, with the scenario, the configurations, the key handling and the tests.

## Measurement comes before decision

Sitting in the authorization path makes fluxrig a measurement point, and that is true before a single network API is called. From fields the messages already carry:

- decline rate by reason (`DE 39`, which travels on the reply), and how it moves during the day
- where declines concentrate, by merchant category (`DE 18`) and acceptor (`DE 42`, `DE 43`)
- entry mode (`DE 22`): chip against contactless against manual
- **response time per leg**, measured rather than declared

Decline rates, concentration and entry mode an issuer gets eventually anyway: its own core produces them the next morning, in aggregate. Response time per leg has nowhere else to come from, because latency exists only while the transaction is on the wire and nothing downstream records it.

That makes for a first deployment with a small ask attached: put fluxrig in the path, change no outcome, and get an analytics view of your own traffic. Turning on a network signal in shadow mode extends that view with a fraud dimension while still deciding nothing. Only then is there a conversation about acting on it, and by then the institution is arguing from its own data.

## What this pattern is not

- **It is not an acquiring-side feature.** An acquiring switch cannot answer whose card this is, so it cannot ask about the cardholder's phone.
- **It is not a fraud score.** CAMARA is explicit that these are technical network signals, not verdicts. The decision, and the responsibility for it, stay with the institution.
- **It is not, as drawn, a direct integration with an operator.** Nothing stops it from being one: an operator's endpoint is another HTTP call, and the same configuration reaches it. What does not fit inline is CAMARA's three-legged authorization, which cannot complete while a transaction waits. A direct integration therefore means running that half yourself, ahead of time and off the transaction path, and holding the result until it is needed. That is ordinary work, and it is the work an aggregator is selling.
- **It is not a replacement for step-up authentication.** It is an input that can make step-up unnecessary, or make it obviously necessary.

## Related

- [Enriching an authorization with a network signal](../tutorials/roaming_enrichment.md): one of these built end to end.
- [Building a multi-region payment switch](../tutorials/payment_switch_conductor.md): the routing layer this composes with.
- [Payment use cases](payments.md): the broader set of operational patterns.
- [GSMA Open Gateway](https://www.gsma.com/solutions-and-impact/gsma-open-gateway/) and [CAMARA](https://camaraproject.org/): the initiative and the API project.
