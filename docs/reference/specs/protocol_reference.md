---
slug: /reference/protocol/iso8583/reference
title: Protocol reference
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Protocol reference

fluxrig routes and transforms from configuration: a scenario says how signals are
wired, a spec says what one of them carries. Configuration a machine reads can
also describe itself, and a description derived on each render cannot fall behind
what runs.

`fluxrig spec doc` is that for protocol specs. Today it renders the ISO 8583
dialects the [SDL](iso8583_sdl.md) describes, which is where fluxrig's protocol
description goes deepest. The document is derived from the spec the engine loads,
so there is no second artefact to keep in step.

This is the document a certification team asks for and an integration partner is
sent: what the interface accepts, message by message and element by element.
Normally it is a PDF or a Word file somebody maintains by hand beside the
implementation, and it is wrong in the way that only becomes visible during
testing. Here it is output, not an obligation.

<SpecReference
  title="ISO 8583:1987 (ASCII)"
  doc="/reference/iso8583-v87-ascii.html" />

It carries the spec it was made from, numbered and foldable, in a pane of its
own. In a public render the blocks of fields marked `scope: private` are cut out
of it and marked, and the lines that remain keep the numbers the file gives them,
so a line shown here is the line to open. That spec ships with fluxrig at
`examples/specs/iso8583-v87-ascii.yaml`, and this is what produced the document
above:

```bash
fluxrig spec doc examples/specs/iso8583-v87-ascii.yaml \
  --format html --scope public --out iso8583-v87-ascii.html
```

## Why it is rendered rather than written

A spec stores its rules on the fields, because a field's rules are what change
together: a scheme bulletin names one data element and states what it does across
several messages. A reader arrives with the opposite question, "what does an 0200
carry", and needs the same matrix read down the other axis.

The per-message tables in the document are that inversion, computed at render
time. A reference maintained by hand answers the reader's question by restating
the spec, and the two disagree the first time either is edited.

## Scopes

A field may declare `scope: private`. It stays in the spec and the engine keeps
enforcing it; what changes is who gets to read about it.

| `--scope` | Contains | Use |
|:---|:---|:---|
| `public` (default) | Everything except fields marked `scope: private`, plus a count of what was withheld | The document you publish or send to a counterparty |
| `complete` | Every field | Internal reference |

Only the public variant is eligible for publication. The render above is a public
one and says so in its own header: seven private elements are omitted from it,
among them the encrypted PIN block and the message authentication code.

## Formats

| `--format` | For |
|:---|:---|
| `markdown` (default) | A repository or a docs site |
| `html` | A page that is read, printed, or saved as a PDF |

The HTML is self-contained. No scripts, stylesheets or fonts are fetched, so it
opens with no network and survives being mailed as an attachment. Everything it
needs travels inside the file.

The spec is loaded before it is rendered, so a spec that does not resolve fails
here instead of producing a reference that describes nothing real.

## What the document contains

| Section | Built from |
|:---|:---|
| Introduction | `spec.overview` |
| How to read this | The notation the document itself uses: layouts, value kinds, how a condition is written |
| Messages | One entry per MTI, each with the elements it carries, their usage, and the condition that decides a conditional one |
| References | `spec.references`, with publisher and link |
| Data elements | One entry per element: what it means, its alias, its format and classification, the messages it appears in, and its closed value sets |

## What a spec has to say to be worth reading

The renderer supplies structure. The prose is the spec's, and an element with
nothing written about it renders as a name and a layout.

| Key | Becomes |
|:---|:---|
| `spec.name` | The document's title |
| `spec.overview` | The introduction, which is where the protocol's own conventions belong: what an amount's minor unit is, which element carries a currency, what the message type indicator's digits mean |
| `spec.references[]` | The References section. `title`, `publisher`, `note` and `url` |
| `fields.<n>.meaning` | What the element is, in the reader's terms |
| `fields.<n>.note` | The caveat a reader needs and the standard does not state |
| `fields.<n>.messages[]` | The per-message rows, including the `when` condition shown verbatim |
| `values` headings | Each value table, marked closed when the spec says the list is complete |

`meaning` is the semantic layer's. The wire layer's `description` is moov's label
for the element and is left alone: a spec that says nothing about an element
still renders under the name the wire layer gives it. See
[ISO8583 SDL](iso8583_sdl.md) for how the two layers divide.

## Publishing one

The public render is a single file with no external dependencies, which is what
makes it publishable by copying. Put it wherever static files are served; the
document on this page is a file in the site's static directory.

Regenerate it on release rather than on edit. A reference carries its spec's
version in its header, and a reader who saves the file needs to know which
contract they are holding.

## Proving it describes the traffic

A generated document is only worth more than a written one if the claim can be
checked, and it can. Every message the codec handles carries three values:

| Metadata | Answers |
| :--- | :--- |
| `codec.spec_id` | Which spec |
| `codec.spec_version` | Which contract, and it is the version printed in the document's header |
| `codec.spec_hash` | Which bytes, which changes when any character does |

So the chain closes: the document names a version, the traffic names a version,
and the hash says whether the bytes behind both are the same. A reviewer asking
"does this describe what the switch actually did" compares three strings rather
than reading two documents. See [Spec manager](../spec_manager.md) for where the
versions come from.
