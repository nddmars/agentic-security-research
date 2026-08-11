# Agentic Security Research

> Research, experiments, and tooling exploring the security of AI agents — and
> the use of AI agents for security research.

## Overview

This repository is a working space for **agentic security research**. It brings
together notes, prototypes, and reproducible experiments across two connected
themes:

- **Security *of* agentic systems** — threat modeling, prompt-injection and
  tool-abuse analysis, sandboxing and permission boundaries, and evaluation
  of autonomous or semi-autonomous agents.
- **Security research *with* agents** — using LLM-driven agents to assist with
  vulnerability triage, dependency and supply-chain review, and automated
  analysis of open-source projects.

The goal is to build a practical, well-documented body of work that others can
read, reproduce, and extend.

## Repository layout

The repository is organized around self-contained pieces of work. As material
lands, expect a structure along these lines:

```
.
├── docs/          # Write-ups, threat models, and background notes
├── experiments/   # Reproducible experiments and prototypes
├── tools/         # Small utilities and helper scripts
└── data/          # Sample inputs and fixtures (no sensitive data)
```

## Getting started

1. Clone the repository:
   ```bash
   git clone https://github.com/nddmars/agentic-security-research.git
   cd agentic-security-research
   ```
2. Browse `docs/` for background and `experiments/` for runnable material.
   Each experiment includes its own README with setup and run instructions.

## Responsible use

This project is intended for **defensive and educational** security research.
All work should be conducted ethically and legally — only against systems you
own or are explicitly authorized to test, and in line with responsible
disclosure practices. Nothing here is intended to enable harm, unauthorized
access, or abuse.

## Contributing

Contributions, corrections, and discussion are welcome. Please open an issue to
propose changes before submitting substantial work, and keep experiments
reproducible and clearly documented.

## License

Released under the [MIT License](LICENSE).

Copyright (c) 2026 Surya Nalluri.
