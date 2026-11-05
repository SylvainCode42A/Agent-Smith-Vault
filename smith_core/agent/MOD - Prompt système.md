---
type: module
title: MOD - Prompt système
language: texte + Python
path: ⚠️ proposé smith_core/agent/prompts.py
status: draft
related: ["[[MOD - Manuel sandbox]]", "[[MOD - Agent MBPP]]", "[[MOD - Agent SWE-bench]]"]
tags: [agent-smith, smith_core, module, thomas]
---
# MOD - Prompt système

## Rôle
Assembler le prompt système : **règles de format** + **manuel des outils** (généré par la sandbox) + **un exemple de boucle**. Il est copié tel quel dans `SolutionOutput.system_prompt` (vérifié par les évaluateurs).

## Ce qui est exigé
1. Documentation claire des outils → c'est [[MOD - Manuel sandbox]] qui la génère, on l'insère.
2. Exemples des *slots* : `Thought:` / `Code:` / `Observation:`.
3. Exemple d'une boucle efficace.

## Deux prompts, deux budgets
| | MBPP | SWE-bench |
|---|---|---|
| Taille cible | **≤ 700 tokens** (budget 6 000 cumulés) | 1 500–3 000 tokens OK |
| Stratégie à enseigner | écrire la fonction, `run_tests`, si OK `final_answer` — **dans le même bloc** | localiser (search) → lire → reproduire → éditer → `run_tests` → `final_answer(get_patch())` |

## Bout de prompt MBPP (squelette)
~~~text
You solve a Python task. Reply with:
Thought: <one or two sentences>
Code:
```python
<code>
```<end_code>

Tools available in the code: {manual}
Rules: define the function with the exact given signature; test it; submit.
Example:
Thought: I write the function and test it before submitting.
Code:
```python
solution = '''def add(a, b):
    return a + b'''
r = run_tests(code=solution, test_list=TESTS)
print(r)
if '"success": true' in r:
    final_answer(solution)
```<end_code>
~~~
Idée clé : **une seule itération** suffit si le test passe. `TESTS` peut être injectée dans le namespace par l'agent avant le premier tour (une variable, pas des tokens). ⚠️ à vérifier : le format exact du JSON renvoyé par votre `run_tests` (espaces) → mieux : `json.loads(r)["success"]` (avec `json` dans les imports autorisés).

## SWE-bench : ce que le prompt doit dire
- Le repo est à `/testbed`. Ne **jamais** modifier les tests ; corriger le code source.
- Méthode : `search_code` / `search_function_or_class_definition_in_code` → `read_file` par plages (≤ 80 lignes) → `edit_file` (petit `old_str` unique) → `run_tests()` → `final_answer(get_patch())`.
- **Ne pas** chercher la solution en ligne / dans l'historique git (sécurité IA, note 0).
- Réponses courtes : le budget **sortie** est 10 000 tokens pour 30 tours (~330/tour).

## Fonctions à créer
| Symbole | Rôle | Appelé par | Appelle |
|---|---|---|---|
| `build_system_prompt(template, manual) -> str` | insère le manuel | [[MOD - Agent MBPP]], [[MOD - Agent SWE-bench]] | [[MOD - Manuel sandbox]] `build_manual` |
| `MBPP_TEMPLATE`, `SWE_TEMPLATE` | textes | idem | — |
| `build_user_prompt_mbpp(task)` / `build_user_prompt_swe(task)` | énoncé de la tâche | agents | — |

## Cas limites
- `hints_text` SWE-bench : on peut l'inclure (fourni par la tâche), pas de recherche externe.
- Le prompt contient des accolades (`{}` du code) → attention à `str.format` ; préférer `template.replace("{manual}", manual)`.

## Notes liées
[[MOD - Extraction de code]] · [[10 - School/10 - Agent-Smith/Index des fonctions|Index des fonctions]]
