# fluxrig documentation

The published documentation for [fluxrig](https://fluxrig.org), an open-source
protocol orchestration engine. This repository holds **content only**: no build
tooling, no scripts, no maintenance code.

## Structure

```
docs/tutorials/     end-to-end walkthroughs, each one runnable
docs/use_cases/     the generic problem and the standards involved
docs/reference/     gears, configuration, the API
docs/overview/      concepts
docs/changelog.md   generated at release time, never edited by hand
```

## Writing

- **A tutorial builds one worked example, completely.** A use case describes the
  generic problem and links to the tutorial rather than summarising it.
  Duplicated prose drifts apart and then contradicts itself.
- **The reader has only ever seen the released version.** Do not write "an
  earlier version", "we changed", or "this used to": that past is not published
  and cannot be checked. Describe what the thing does and why.
- **Anything described must exist in the release.** Tag what does not with
  `[Roadmap]` or `[Planned]`, and never describe unimplemented behaviour in the
  present tense.
- **Every measured figure carries its conditions**: the hardware, what else was
  running, what was being measured. A number without them is decoration.
- **Configuration tables must match** the `koanf`/`json` tags in the source.
- Sentence case in headings. No emojis. No pricing or cost comparisons.
- **No em-dashes.** Use a colon, a full stop, or parentheses, whichever the
  sentence actually wants. A hyphen substituted for one leaves the sentence
  mispunctuated, so the sentence gets rewritten rather than patched.
- Every third-party product mentioned needs a validated link.

## Before committing

The site is built from a sibling repository, not from here, so validation and
preview run there. Ask if you do not have that workspace: a change here is not
visible until it is built somewhere else.
