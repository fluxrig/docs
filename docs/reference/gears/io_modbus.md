---
slug: /reference/gears/io_modbus
title: Modbus I/O gear [Roadmap]
---

<!-- Copyright (c) 2026 JAAB Tech SAS, Uruguay All Rights Reserved -->
<!-- See https://jaab.tech -->

# Modbus I/O gear [Roadmap]

> [!WARNING]
> **[Roadmap]**: This gear is currently under development and not yet available.

The `io_modbus` Gear will provide a native Go implementation for polling industrial registers from PLCs and SCADA systems.

## Planned capabilities

- **Protocols**: Modbus/TCP and Modbus/RTU (Serial).
- **Register types**: Coils, Discrete Inputs, Holding Registers, and Input Registers.
- **Polling engine**: Deterministic, high-frequency poller with sub-millisecond precision.
- **Metadata**: Automatic injection of Slave ID, Register Address, and Unit context into `fluxMsg`.
