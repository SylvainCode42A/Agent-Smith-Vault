---
type: module
title: MOD - Modèles Pydantic
language: Python 3.10 / pydantic v2
path: ⚠️ proposé smith_core/models.py
status: draft
related: ["[[OVR - smith_core]]", "[[MOD - Boucle agent]]", "[[MOD - Sandbox exécuteur]]"]
tags: [agent-smith, smith_core, module, sylvain, thomas]
---
# MOD - Modèles Pydantic

## Rôle
Les 5 modèles imposés par le sujet, **copiés tels quels** : `StepMetrics`, `SolutionOutput`, `SandboxConfig`, `MBPPTaskInput`, `SWEBenchTaskInput`. Un seul fichier, importé partout. À écrire **ensemble le jour 1** : c'est le contrat de données.

## Ce qui est exigé
- Les noms de champs **exacts** (la moulinette valide le JSON). Ne renommez rien, n'enlevez rien.
- `SandboxConfig` chargeable depuis un JSON (`uv run sandbox sandbox_template.json`).
- Un fichier `sandbox_template.json` dans le repo (le sujet demande « configuration files for sandbox and models »).

## Fonctions à créer
| Symbole | Rôle | Appelé par | Appelle |
|---|---|---|---|
| `StepMetrics`, `SolutionOutput` | sortie (copie du sujet) | [[MOD - Boucle agent]] | — |
| `SandboxConfig` | limites sandbox | [[MOD - CLI sandbox]], [[MOD - Sandbox exécuteur]] | — |
| `MBPPTaskInput` / `SWEBenchTaskInput` | entrée | [[MOD - Agent MBPP]] / [[MOD - Agent SWE-bench]] | — |
| `load_config(path) -> SandboxConfig` | JSON → modèle, défauts si absent | [[MOD - CLI sandbox]], agents | `SandboxConfig.model_validate_json` |
| `write_solution(sol, path)` | écrit le JSON (crée le dossier) | [[MOD - Boucle agent]] | `model_dump_json(indent=2)` |

## Bout de code
```python
def load_config(path: str | None) -> SandboxConfig:
    if path is None:
        return SandboxConfig()                     # défauts du sujet
    with open(path) as f:
        return SandboxConfig.model_validate_json(f.read())
```
Exemple `sandbox_template.json` (pour SWE-bench, `os`/`subprocess` ne sont **pas** nécessaires : on passe par les outils MCP) :
```json
{ "authorized_imports": ["math", "re", "json", "collections", "collections.*", "itertools"],
  "allowed_directories": ["/testbed", "/tmp/agent"],
  "max_execution_time_seconds": 30, "max_memory_mb": 512 }
```

## Pièges
- `task_id` est un `int` en entrée MBPP mais une `str` dans `SolutionOutput` → `str(task.task_id)`.
- `total_requests` = nombre de requêtes HTTP **retries compris** = `sum(1 + step.retries)`.
- `iterations` = `len(steps)`.

## Notes liées
[[10 - School/10 - Agent-Smith/Index des fonctions|Index des fonctions]]
