# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**转接神经元 (Relay Neuron)** is a research archive of exercise science and fitness literature (2024-2026). It is not a software project — there is no source code, build system, or tests. The repository contains only markdown research files.

## Repository Structure

```
relay-neuron/
├── README.md                    # Project description
├── RESEARCH_MASTER_LIST.md      # Task checklist (8 sections, 40+ research topics)
├── CLAUDE.md                    # This file
└── research/
    ├── YYYY-MM-DD-research-round-N.md   # Periodic research summaries
    └── YYYY-MM-DD-task-X-X-*.md         # Individual research topic files
```

## Research Workflow

When asked to conduct a research iteration:

1. **Search**: Use `mcp__MiniMax__web_search` for fitness/exercise science topics (2024-2026)
2. **Write**: Save findings as `research/YYYY-MM-DD-research-round-N.md`
3. **Commit**: `git add research/ && git commit -m "Research round N"`

## Search Strategy

- **Keywords**: "resistance training", "muscle hypertrophy", "exercise science", "protein synthesis", "fitness research"
- **Time filter**: 2024-01-01 to 2026-12-31
- **Sources**: PubMed, Nature, ScienceDirect, Springer, MDPI, arXiv

## File Naming

- **Research rounds**: `YYYY-MM-DD-research-round-N.md`
- **Individual topics**: `YYYY-MM-DD-task-X-X-TOPIC.md`

## Citation Standards

- Every conclusion must cite the source DOI or stable URL
- Format: `([Journal/Author, Year](URL))`
- Source priority: PubMed > Nature/Science > preprints

## Archive Standards

- Create independent MD file per research
- Include: abstract, key findings, data sources, conclusions
- Save to `research/` directory
