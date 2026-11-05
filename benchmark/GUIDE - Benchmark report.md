---
type: guide
title: GUIDE - Benchmark report
path: BENCHMARK_REPORT.md (racine, en anglais) + benchmark/<modèle>/<tâche>/solution.json
status: draft
tags: [agent-smith, benchmark, guide, sylvain, thomas]
---
# GUIDE - Benchmark report

## Le minimum exigé
- **≥ 5 modèles**, sur **≥ 2 providers**, sur **les mêmes ≥ 3 tâches SWE-bench** → 15 runs minimum.
- **+ 1 ablation** (même modèle, mêmes tâches, une seule chose changée) → +3 runs.
- Les `solution.json` **dans le repo** (dossier `benchmark/`). Mesures manuelles autorisées.
- Free tier uniquement → répartir les runs sur **3 providers** (conseil du sujet) pour ne pas buter sur un quota.

## Étapes dans l'ordre
1. Choisir les 3 tâches : celles conseillées (`sympy__sympy-14711`, `sympy__sympy-13480`, `pydata__xarray-4629`) — raison à écrire : « simples, rapides à évaluer, images plus légères ».
2. Choisir 5 modèles **non-reasoning** de préférence ⚠️ à décider selon les free tiers du moment (ex. 2 Gemini via AI Studio, 2 via Groq, 1 via OpenRouter).
3. Lancer avec un petit script `benchmark/run.sh` (boucle modèle × tâche) qui range `benchmark/<model>/<task>/solution.json`. Lancer la nuit, quotas obligent.
4. Valider chaque solution avec `moulinette_eval validate swebench` → Pass/Fail.
5. Remplir le rapport (un petit script Python qui lit les JSON et sort le tableau Markdown fait gagner du temps — optionnel).

## Squelette du rapport (en anglais)
```markdown
# Benchmark Report
## 1. Setup            models, providers, tasks, why
## 2. Results          | Model | Task | Pass | Iter | In tok | Out tok | Time (s) |
## 3. Provider reliability   | Model/Provider | Avg request ms | Retries | Availability |
## 4. Intermediary metrics   exploration efficiency ; submission discipline
## 5. Ablation         before/after, same model, same tasks
## 6. Conclusions      chosen model(s), discarded ones, based on data
```

## D'où vient chaque colonne
| Colonne | Champ du `solution.json` |
|---|---|
| Iter | `iterations` |
| In / Out tok | `total_input_tokens` / `total_output_tokens` |
| Time | `total_time_seconds` |
| Avg request ms | moyenne de `steps[].request_time_ms` |
| Retries | somme de `steps[].retries` |
| Availability | runs sans erreur API / runs lancés |
| Exploration efficiency | 1er `step` dont `sandbox_input` lit/édite le fichier du patch |
| Submission discipline | `step(final_answer)` − `step(1er run_tests OK)` |

## Ablation facile (choisir une)
- Avec / sans troncature des observations.
- Prompt avec / sans exemple de boucle.
- Tous les outils / seulement 5 outils (le sujet pose la question : « que se passe-t-il quand on retire les outils inutilisés ? »).

## Notes liées
[[MOD - Client LLM]] · [[FLOW - Tâche SWE-bench]]
