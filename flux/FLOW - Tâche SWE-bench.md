---
type: flow
title: FLOW - Tâche SWE-bench
status: draft
related: ["[[MOD - Agent SWE-bench]]", "[[MOD - MCP SWE-bench]]", "[[MOD - Boucle agent]]"]
tags: [agent-smith, flux]
---
# FLOW - Tâche SWE-bench

## Rôle
Le parcours d'une tâche SWE-bench, et **la méthode** que le prompt doit enseigner au LLM (le sujet conseille : « avez-vous résolu la tâche vous-même à la main ? Votre prompt reflète-t-il cette méthode ? »).

## Chaîne d'appels numérotée
1. `dump swebench` → `task.json` (`instance_id`, `problem_statement`, `docker_image`, `eval_script`).
2. `start_container` → `docker run -d … sleep infinity`.
3. `copy_eval_script` → `/tmp/agent/eval.sh` dans le conteneur.
4. `McpClient.from_stdio("python mcp_tools_swebench.py", env={SWE_CONTAINER, TESTBED_PATH, EVAL_SCRIPT})` → 9 outils découverts.
5. `Sandbox.start()` ; prompt = méthode + manuel + `problem_statement`.
6. Itérations (exemple type, ~8 à 15 tours) :

| Tour | Le LLM fait | Outil |
|---|---|---|
| 1 | trouver où est le symbole cité dans l'issue | `search_function_or_class_definition_in_code("Mod")` |
| 2 | lire autour | `read_file("/testbed/sympy/core/mod.py", 20, 90)` |
| 3 | (option) mesurer la baseline des tests | `run_tests()` → noter les tests en échec |
| 4 | reproduire le bug | `run_command("python -c '…'", "/testbed")` |
| 5 | corriger | `edit_file(path, old_str, new_str)` → `OK` (ou erreur de syntaxe signalée) |
| 6 | vérifier | `run_tests()` |
| 7 | soumettre | `final_answer(get_patch())` |

7. `remove_container` dans le `finally`.
8. `validate swebench` : la moulinette applique **votre patch** dans une image propre et lance l'`eval_script`.

## Les mesures du benchmark se lisent ici
- *Exploration efficiency* : tour où le fichier du patch final est lu/édité pour la première fois (ici 2).
- *Submission discipline* : tours entre « tests passent » et `final_answer` (ici 0 → idéal).
Voir [[GUIDE - Benchmark report]].

## Notes liées
[[FLOW - Appel outil MCP]] · [[MOD - Agent SWE-bench]]
