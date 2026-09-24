---
slug: /overview/roadmap
title: Product roadmap
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Product roadmap

This roadmap outlines the strategic direction for **fluxrig**. It defines the journey from an orchestration foundation to a high-fidelity, AI-orchestrated patchbay for mission-critical infrastructure.

## Phase 0: proof of concept

| Version | Status | Goal | Description |
| :--- | :--- | :--- | :--- |
| POC | Delivered | Architecture Validation | Validated gRPC vs NATS JetStream, Wasm loading, and protocol parsing. |

## Phase 1: architecture & foundation

| Version | Status | Goal | Description |
| :--- | :--- | :--- | :--- |
| v0.1.x | Delivered | Core Data Physics | Defined the fluxMsg model, time-ordered identity (now UUID v7, RFC 9562), and Snake Protocol. |

## Phase 2: core runtime

| Version | Status | Goal | Description |
| :--- | :--- | :--- | :--- |
| v0.2.x | Delivered | Autonomous Execution | Single-binary Rack engine, self-healing connectivity, and DuckDB storage. |

## Phase 3: open & flexible logic

| Version | Status | Goal | Description |
| :--- | :--- | :--- | :--- |
| v0.3.x | Delivered | Universal I/O | Bento wrapped as a gear: Bloblang mapping, with the pure and local I/O connector sets. |
| v0.4.x | Delivered | Public Release & Native Gears | Open-source launch; ISO 8583 I/O and codec gears, Coat Check correlation, distributed state (NATS KV), and secure enrollment. |
| v0.5.x | Delivered | Sovereign Identity | UUID v7 (RFC 9562) identity plane and telemetry hardening. |
| v0.6.x | Delivered | Polyglot Logic | Wasm runtime (wazero) with signed, supply-chain-secure modules. |

## Phase 4: scale & hardening

| Version | Status | Goal | Description |
| :--- | :--- | :--- | :--- |
| v0.7.x | Delivered | Payment Switching | Conductor gear: connection routing + reply correlation over the valet ticket store, gear manifests, and native ISO 8583 TLS/mTLS. |
| v0.8.x | Delivered | EMV Chip Data | BER-TLV composite parsing, with tags the spec does not declare carried through untouched. |
| v0.9.x | Delivered | Enrichment & Correlation | Enrichment from outside the message under a hard deadline, and correlation keys that survive a bus hop. |
| v0.10.x | Active | The Semantic Layer | A protocol spec states its rules and the codec enforces them: per-message field usage, conditions, value domains and cross-field checks, plus a protocol reference generated from the spec. |
| Future | Planned | Differential Analysis | Correlator gear: parallel shadow mastering and reconciliation against the immutable archives. |
| Future | Planned | Guaranteed Local Lane | Rack operation without the Mixer, continued: a NATS leaf node in each Rack keeps guaranteed delivery on disk, with retention limits and durable delivery between Racks. |
| Future | Planned | Hardened Integrity | Forensic analytics (DuckLake), at-rest encryption for persistent stores, durable ticket store (`local_durable`), per-destination circuit breaker, and peer heartbeats. |

## Phase 5: enterprise control

| Version | Status | Goal | Description |
| :--- | :--- | :--- | :--- |
| Future | Planned | Global Management | fluxrig Studio (Visual Builder), Remote Fleet Management (OTA), and HSM/KMS security integration. |
| Future | Planned | Durable Workflows | Integration with Temporal for long-running business processes. |

## Phase 6: autonomous engineering

| Version | Status | Goal | Description |
| :--- | :--- | :--- | :--- |
| Future | Planned | AI Orchestration | AI-driven configuration and deterministic anomaly detection. |