---
type: méthode
title: Conventions — Agent Smith
status: à valider
created: 2026-11-03
tags: [agent-smith, conventions]
---
# Conventions — Agent Smith

Paramètres propres à Agent Smith, selon [[Instructions]] § 11.

| Paramètre | Valeur |
|---|---|
| Racine | `10 - School/10 - Agent-Smith/` |
| Domaines | Calqués sur la hiérarchie du repo (choix de l'auteur) : `smith_core/` (agent, llm, sandbox), `agent_mbpp/`, `agent_swebench/`, `mcp_tools/` (= `mcp_tools_*.py` à la racine du repo), `benchmark/`, `tests/`, `flux/`. ⚠️ s'écarte de la règle « un domaine = une frontière d'exécution » : les vraies frontières sont **process agent**, **process sandbox**, **process serveur MCP**, **conteneur Docker** — elles sont montrées dans [[OVR - smith_core]]. |
| Accès | L'agent ne lit pas le repo (Filesystem limité au vault). Le contenu vient du sujet v1.1 et du `tree` collé le 2026-11-03. Tout le code des notes est **non exécuté** : ⚠️ à vérifier. |
| **Le juge** | La moulinette : `moulinette_eval validate mbpp` / `validate swebench` et les scripts `exam_mbpp.sh` (4/5), `exam_swebench.sh` (2/3), `exam_sandbox.sh` (tout). En local : `uv run pytest tests/`. |
| Workflow | Tickets dans [[10 - School/10 - Agent-Smith/Tickets\|Tickets]] (et GitHub Project). Une branche par personne, PR vers `main`. |
| Langue | Notes en français ; noms du code tels quels. README et BENCHMARK_REPORT en **anglais** (imposé). |
| Personnes | Tags `sylvain` et `thomas` dans le frontmatter des MOD. |
| Historique | 2026-11-03 : création du vault complet sur délégation explicite (« remplis mon vault »). |

## Règles propres à ce projet
- **Strict minimum** : chaque note dit ce qui est *exigé* et ce qui est *optionnel* (marqué « bonus, on ne fait pas »).
- **Code « à trous »** : les extraits montrent la structure et les pièges, pas la solution complète. Les `...` et `# TODO` sont volontaires.
- **Tableau « Fonctions à créer »** dans chaque MOD (Symbole | Rôle | Appelé par | Appelle), relié à [[Index des fonctions]].
- **Interdit par le sujet** : smolagents, langgraph, llama-index, crewai, autogen, RestrictedPython. Clés API en dur = échec.
- **Python 3.10 + uv** obligatoires.
