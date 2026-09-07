---
slug: /reference
title: Technical reference
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Technical reference

Five sections, each with its own page listing what is in it. The split follows
the question being asked, not the order things happen.

| Section | Answers |
| :--- | :--- |
| **[Protocol & data](/docs/reference/protocol)** | What a message is on the wire, and what it means |
| **[Platform & configuration](/docs/reference/platform)** | How a fleet is configured, and who is allowed into it |
| **[Gears catalog](/docs/reference/gears)** | What each gear does and what it accepts |
| **[Operations & analytics](/docs/reference/operations)** | Running the two processes, and reading what they report |
| **[Development & QA](/docs/reference/development)** | Extending the platform, and proving it behaves |

[Technical stack](./tech_stack.md) sits outside them: it is what fluxrig is built
on, and the licence of each dependency.

## Where to start

**Evaluating.** [Data model](./data_model.md) for what travels, then
[Orchestration scenarios](./scenario.md) for how a node is composed. The
[Gears catalog](/docs/reference/gears) says what can be composed.

**Integrating a protocol.** [ISO8583 SDL](./specs/iso8583_sdl.md) describes a
dialect, and [Protocol reference](./specs/protocol_reference.md) is the document
rendered from one, which is what a counterparty reads.

**Running it.** [Operating fluxrig](./operations.md) covers both processes;
[Platform configuration](./configuration.md) is every field and its default.

> [!TIP]
> For the "Studio & Stage" thinking behind the naming, see
> [Design philosophy](../overview/philosophy.md).
