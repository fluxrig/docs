---
title: Side-Chain Inference Pattern
slug: /architecture/ai-sidechain
---

# Side-chain inference pattern

The **Side-Chain Inference** pattern `[Roadmap]` is the intended architectural standard for integrating Artificial Intelligence (ML/LLM) into the **fluxrig** data plane. It is in the architectural research phase (see the [AI strategy](ai_strategy.md)). The description below is the target design, not a shipped capability. It ensures that probabilistic models can provide high-value insights (scoring, categorization, anomaly detection) without compromising the **Determinism** or **Performance** of the primary execution path.

## The theory: dry vs. wet signals

*   **The Dry Signal (Deterministic)**: The core transaction logic (for example, "Is this ISO8583 message valid?") which must be fully reproducible and low-latency.
*   **The Wet Signal (Probabilistic)**: The AI-augmented insight (for example, "What is the probability this is a fraudulent transaction?") which is non-blocking and descriptive.

## Implementation blueprint

Integrating an AI model (for example, through the **Sovereign AI Bridge**) follows a three-stage lifecycle:

1. **Tap**.
2. **Infer**.
3. **Feed**.

### The signal tap (aux send)
A dedicated Gear (often the **[Bento Gear](../reference/gears/bento.md)**) acts as an "Aux Send". It receives a clone of the `fluxMsg` from the main logic flow.

```yaml
wires:
  - from: ingress.out
    to:   ai-bridge.in      # The Signal Tap
  - from: ingress.out
    to:   business-logic.in # The Hot Path
```

### Isolated inference (sovereign bridge)
The Bridge Gear sends the signal to a local inference engine (e.g., Ollama, TensorRT, or a specialized Sovereign AI node).

*   **Out-of-Process**: Inference happens outside the Rack core execution loop to prevent CPU/Memory starvation.
*   **Non-Blocking**: The primary business logic continues to execute while the AI model is "thinking."

### Metadata feedback loop
Once the AI model completes its analysis, the Bridge Gear emits a **Feedback Signal**. The original message already moved forward, so the AI result typically attaches to the **Signal Metadata** of the *next* related signal. It may also persist in shared state (for example, **[Coat Check](../reference/gears/coatcheck.md)**).

```go
// Example Metadata Feedback Structure
msg.Metadata["flux.ai.fraud_score"] = "0.92"
msg.Metadata["flux.ai.rationale"] = "unusual_temporal_cluster"
```

## Security sovereignty
In the Side-Chain pattern, mask the raw sensitive payload (for example, a credit card number) before sending it to the AI Bridge. The AI engine then receives only the "Contextual Vectors" required for analysis. PII never leaves the deterministic path. Native deterministic masking is `[Roadmap]`. Today a logic gear placed ahead of the bridge performs this masking.

---

## Technical reference
*   **Data Model**: See **[fluxMsg Fields](data.md)**.
*   **State Management**: See **[Coat Check Gear](../reference/gears/coatcheck.md)**.
