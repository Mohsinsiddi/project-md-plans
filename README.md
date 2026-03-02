<h1 align="center">
  <br>
  Project Plans & Architecture Docs
  <br>
</h1>

<p align="center">
  <strong>DotMatrix</strong> — Technical architecture documents, proposals, and project plans.
</p>

<p align="center">
  <a href="https://github.com/Mohsinsiddi/project-md-plans">
    <img src="https://img.shields.io/badge/repo-project--md--plans-blue?style=flat-square&logo=github" alt="GitHub">
  </a>
</p>

---

This repository contains structured architecture documents, implementation proposals, and technical plans for active and upcoming projects. Each project lives in its own folder with a README linking to all relevant docs.

## Projects

| Project | Description | Status |
|---------|-------------|--------|
| [AQUARI Subscription & Affiliate](./aquari-subscription-affiliate/) | Recurring AQUARI token accumulation system with affiliate program on Base. Fiat + crypto payments, Uniswap V2 swaps, Privy wallet auth. | Planning |

## Structure

```
project-md-plans/
├── README.md                              ← You are here
│
├── aquari-subscription-affiliate/         ← Project folder
│   ├── README.md                          ← Project overview + doc links
│   ├── proposal-a-smart-contract.md       ← Full architecture: contract approach
│   ├── proposal-b-backend-only.md         ← Full architecture: backend-only approach
│   └── notes-and-recommendations.md       ← Risks, suggestions, timeline
│
└── <next-project>/                        ← Future projects go here
    ├── README.md
    └── ...
```

## How to Use

1. Each project gets its own folder named `<project-name>/`
2. Every folder has a `README.md` that links to all docs within it
3. Architecture proposals, flow diagrams, and technical decisions are written in Markdown
4. Docs are meant to be shared with team members and stakeholders for review and alignment

## Conventions

- **Proposal docs** — Full architecture deep-dives with diagrams, tx counts, pros/cons
- **Notes docs** — Risks, recommendations, timeline estimates, open questions
- **Diagrams** — ASCII art for portability (renders everywhere, no external tools)
- **Tables** — Used for comparisons, tx counts, and decision matrices

---

<p align="center">
  <sub>Built by <a href="https://github.com/Mohsinsiddi">@Mohsinsiddi</a> / DotMatrix</sub>
</p>
# project-md-plans
