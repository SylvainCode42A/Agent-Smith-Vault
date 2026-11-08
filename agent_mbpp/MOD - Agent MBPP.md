---
type: module
title: MOD - Agent MBPP
language: Python 3.10
path: agent_mbpp/__main__.py (existe déjà)
status: draft
related: ["[[MOD - Boucle agent]]", "[[MOD - MCP MBPP]]", "[[MOD - Prompt système]]", "[[FLOW - Tâche MBPP]]"]
tags: [agent-smith, agent_mbpp, module, thomas]
---
# MOD - Agent MBPP

## Rôle
Le programme lancé par la moulinette : `uv run python -m agent_mbpp --task-file … --output … --model-name … --provider-url …`. Il colle les morceaux : tâche → sandbox + serveur MBPP → prompt → boucle → `solution.json`.

## Ce qui est exigé
- Ces 4 options exactes. `max_iterations` configurable (option `--max-iterations`, défaut 10).
- Limites : **10 itérations, 6 000 tokens entrée, 1 500 sortie, 120 s**. Dépasser = tâche ratée.
- **4 tâches sur 5** réussies à l'exam, sans relance.
- `solution` = le **code de la fonction**.

## Marges de sécurité proposées ⚠️
| Limite | Sujet | On vise |
|---|---|---|
| Entrée | 6 000 | stop si `cumul + estimation_prochain_tour > 5 500` |
| Sortie | 1 500 | `max_tokens = min(500, reste - 50)` par appel |
| Temps | 120 s | arrêter la boucle à 100 s (démarrage `uv` + serveur MCP compris) |

## Fonctions à créer
| Symbole                            | Rôle                                                                | Appelé par             | Appelle                    |
| ---------------------------------- | ------------------------------------------------------------------- | ---------------------- | -------------------------- |
| `main()`                           | parse, exécute, écrit, **toujours** un JSON                         | `python -m agent_mbpp` | tout ce qui suit           |
| `parse_args()`                     | `--task-file --output --model-name --provider-url --max-iterations` | `main`                 | `argparse`                 |
| `load_task(path) -> MBPPTaskInput` | lit le JSON                                                         | `main`                 | [[MOD - Modèles Pydantic]] |
| `mbpp_fallback(steps, func_name)`  | dernier code extrait qui contient `def func_name`                   | [[MOD - Boucle agent]] | —                          |

## Squelette (à trous)
```python
def main():
    args = parse_args()
    load_dotenv(override=False)
    task = load_task(args.task_file)
    server = f"{sys.executable} {ROOT / 'mcp_tools_mbpp.py'}"     # chemin absolu : la moulinette lance depuis student/
    sandbox = Sandbox(load_config(ROOT / "sandbox_mbpp.json"), McpClient.from_stdio(server))
    try:
        sandbox.start()
        sandbox.execute(f"TESTS = {task.test_list!r}")            # la variable existe pour le LLM, 0 token
        solution = run_agent(str(task.task_id), "mbpp",
                             build_system_prompt(MBPP_TEMPLATE, sandbox.manual()),
                             build_user_prompt_mbpp(task), sandbox, LLMClient(...), Limits(...), ...)
    except Exception as e:
        solution = SolutionOutput(..., success=False, error=str(e))   # TODO: remplir tous les champs requis
    finally:
        sandbox.close()
    write_solution(solution, args.output)
```
⚠️ L'appel `execute("TESTS = …")` ne doit **pas** compter comme une itération ni apparaître dans `steps`.

## Ce que voit le LLM (prompt utilisateur, ~150 tokens)
```text
Task: Write a function to find the shared elements from the given two lists.
Signature: def similar_elements(test_tup1, test_tup2):
Tests (also available as variable TESTS):
assert set(similar_elements((3, 4, 5, 6),(5, 7, 4, 10))) == set((4, 5))
```
Tour 1 attendu : la fonction + `run_tests(code=solution, test_list=TESTS)` + `final_answer` si succès. **1 itération, ~1 300 tokens entrée.**

## Évaluation : la modif live (2–5 min)
On vous demandera de changer un truc et de relancer sur une tâche MBPP. Exemples probables, sachez **où** c'est :
| Demande | Où |
|---|---|
| « Passe max_iterations à 3 » | `--max-iterations` / `Limits` |
| « Change le modèle » | `--model-name` |
| « Ajoute une phrase au prompt » | `MBPP_TEMPLATE` dans [[MOD - Prompt système]] |
| « Interdis `random` » | `sandbox_mbpp.json` |
| « Tronque les observations à 500 caractères » | `truncate` dans [[MOD - Sandbox exécuteur]] |

## Notes liées
[[FLOW - Tâche MBPP]] · [[10 - School/10 - Agent-Smith/Index des fonctions|Index des fonctions]]
