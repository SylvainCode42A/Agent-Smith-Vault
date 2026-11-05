---
type: guide
title: GUIDE - README et rendu
path: README.md (anglais)
status: draft
tags: [agent-smith, guide, sylvain, thomas]
---
# GUIDE - README et rendu

## README.md — sections obligatoires (en anglais)
1. **1re ligne en italique, exactement** : `*This project has been created as part of the 42 curriculum by slidriss, <login_thomas>.*` ⚠️ login de Thomas à compléter.
2. **Description** — but et vue d'ensemble.
3. **Instructions** — `uv sync`, `.env` (noms des variables), les commandes `sandbox`, `agent_mbpp`, `agent_swebench`.
4. **Resources** — docs (MCP, SWE-bench, MBPP, smolagents *comme lecture* seulement), **et comment l'IA a été utilisée** (pour quoi, quelles parties).
5. Imposées en plus : **System architecture**, **Agent loop explanation**, **Sandbox design**, **Tool implementation details**, **Benchmark results and analysis** (résumé + lien vers `BENCHMARK_REPORT.md`).

Astuce : les sections 5 sont des résumés des notes du vault ([[OVR - smith_core]], [[MOD - Boucle agent]], [[MOD - Sandbox sécurité]], [[MOD - MCP SWE-bench]], [[GUIDE - Benchmark report]]) traduits.

## Ce que le repo doit contenir au rendu
- [ ] `mcp_tools_mbpp.py`, `mcp_tools_swebench.py` **à la racine**
- [ ] `sandbox_template.json` (+ configs MBPP/SWE) et config des modèles
- [ ] `README.md`, `BENCHMARK_REPORT.md`, `benchmark/**/solution.json`
- [ ] `pyproject.toml` (script `sandbox`), `uv.lock`, Python 3.10
- [ ] **Aucune clé API** (`git grep -i "api_key\s*=\s*['\"]"`), `.env` dans `.gitignore`
- [ ] Pas de `__pycache__`, pas d'images Docker, pas de `Archive ZIP.zip` (⚠️ présent dans le tree actuel : à retirer), pas de sorties générées hors `benchmark/`

## Préparer la soutenance
- Chacun doit savoir expliquer **toute** l'architecture (pas seulement sa partie) : relire [[Explications - Lire en dernier]] ensemble.
- S'entraîner à la **modif live** (voir le tableau dans [[MOD - Agent MBPP]]), puis `git checkout .`.
- Avoir un `.env` d'exemple (`.env.example` sans valeurs).
