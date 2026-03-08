# Kith Guard 🛡️

**LLM-as-a-Judge sycophancy detection and response quality gate for OpenClaw agents.**

Kith Guard uses [Prometheus 2](https://github.com/prometheus-eval/prometheus-eval) (7B, run locally via Ollama) to score agent responses for sycophancy, capitulation, and misattribution — and rewrites them before they reach the user.

## Why

LLMs fold under social pressure. They abandon correct positions when questioned, agree just to agree, and blame users for their own decisions. Written rules don't fix this — mechanical enforcement does.

Kith Guard is that enforcement.

## How It Works

```
User challenges a decision
        ↓
Primary model generates response
        ↓
Trigger heuristic checks for capitulation patterns
        ↓ (flagged)
Prometheus 2 scores response against sycophancy rubric (1-5)
        ↓
Score ≤ 2: Send as-is
Score > 2: Rewrite with position-first framing, then send
```

## Architecture

- **Judge model:** Prometheus 2 7B (GGUF, runs locally via Ollama — zero API cost)
- **Rubrics:** Custom evaluation criteria (sycophancy is first; hallucination, verbosity, etc. planned)
- **Trigger heuristic:** Pattern matcher to avoid judging every response (only challenge-response patterns)
- **Integration:** OpenClaw skill (future: native pre-send hook when OpenClaw supports middleware)

## Requirements

- [Ollama](https://ollama.ai) installed
- ~4GB disk for the quantized Prometheus 2 model
- macOS (Apple Silicon) or Linux with 8GB+ RAM

## Status

🚧 **Pre-alpha** — Rubric design + prototype phase

## Roadmap

- [ ] Sycophancy rubric v1 with reference answers
- [ ] Trigger heuristic (pattern matcher)
- [ ] Prometheus 2 GGUF integration via Ollama
- [ ] Judge wrapper script
- [ ] OpenClaw skill packaging
- [ ] Test against real failure cases (Mar 8 ticket consolidation)
- [ ] Additional rubrics (hallucination, verbosity, over-promising)
- [ ] OpenClaw pre-send hook contribution (upstream)

## Part of White Russian 🥛

Kith Guard is a component of the [White Russian](https://thedudeabides.ai) product — "AI that actually pushes back."

## License

Apache 2.0

---

*The Dude Abides AI* 🎳
