# fluxrig documentation

The published documentation for [fluxrig.org](https://fluxrig.org).

fluxrig routes, transforms and observes data streams across heterogeneous
environments. The engine itself lives at
[jaab-tech/fluxrig](https://github.com/jaab-tech/fluxrig).

## What is here

- `docs/`: the Markdown the site is built from, one directory per section.
- `VERSION`: the release this snapshot documents.
- `CNAME`: the domain the site answers on.

The site is built with [Docusaurus](https://docusaurus.io) and served by
Cloudflare Pages. There is no `gh-pages` branch and no GitHub Pages build: the
HTML is produced by the release pipeline and deployed from there.

## How this repository is updated

This is a published mirror, not the working line. Each release replaces its
contents with the documentation for the version in `VERSION`, so a commit made
here is overwritten by the next release rather than merged into anything.

That has one consequence worth stating plainly: **a pull request against this
repository cannot be released**. It would be reverted by the next publish.

## Reporting a problem

Something wrong, unclear, or out of date is worth an
[issue](https://github.com/fluxrig/docs/issues), and the more precisely it names
the page the faster it is fixed. Corrections are made on the working line and
reach this repository with the release that carries them.

For contributions to fluxrig itself, see
[Governance & contributing](https://fluxrig.org/docs/development/contributing).

---

Made with ❤️ in Uruguay 🇺🇾 by [JAAB Tech](https://jaab.tech). © 2026 JAAB Tech SAS.
