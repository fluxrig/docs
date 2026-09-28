---
slug: /reference
title: Technical reference
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Technical reference

Five sections describe the system. Each has its own page that lists what it contains. The split follows
the question you ask, not the order in which things happen.

| Section | Answers |
| :--- | :--- |
| **[Protocol & data](/docs/reference/protocol)** | What a message is on the wire, and what it means |
| **[Platform & configuration](/docs/reference/platform)** | How a fleet is configured, and who is allowed into it |
| **[Gears catalog](/docs/reference/gears)** | What each gear does and what it accepts |
| **[Operations & analytics](/docs/reference/operations)** | Running the two processes, and reading what they report |
| **[Development & QA](/docs/reference/development)** | Extending the platform, and proving it behaves |

[Technical stack](./tech_stack.md) sits outside them: it is what fluxrig builds
on, and the licence of each dependency.

## Where to start

**Evaluating.** Read [Data model](./data_model.md) for what travels. Then read
[Orchestration scenarios](./scenario.md) for how you compose a node. The
[Gears catalog](/docs/reference/gears) says what you can compose.

**Integrating a protocol.** [ISO8583 SDL](./specs/iso8583_sdl.md) describes a
dialect. [Protocol reference](./specs/protocol_reference.md) is the document
that you render from one, which is what a counterparty reads.

**Running it.** [Operating fluxrig](./operations.md) covers both processes.
[Platform configuration](./configuration.md) lists every field and its default.

> [!TIP]
> For the "Studio & Stage" thinking behind the naming, see
> [Design philosophy](../overview/philosophy.md).
