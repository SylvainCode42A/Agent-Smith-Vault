---
type: module
title: MOD - MCP SWE-bench
language: Python 3.10 (mcp FastMCP, subprocess, docker CLI)
path: mcp_tools_swebench.py (racine du repo, imposé)
status: draft
related: ["[[MOD - Client MCP]]", "[[MOD - Agent SWE-bench]]", "[[FLOW - Tâche SWE-bench]]", "[[GUIDE - Tests et sécurité]]"]
tags: [agent-smith, mcp, module, sylvain]
---
# MOD - MCP SWE-bench

## Rôle
Serveur MCP des **9 outils obligatoires**. Chaque outil est **testé seul** par la moulinette, qui met la variable **`TESTBED_PATH`** (nom exact) avant de lancer le serveur. Formats de sortie **imposés** (voir tableau).

## Décision proposée ⚠️ : un seul helper `_sh()` qui marche en local ET dans Docker
- Moulinette (outils seuls) : le repo est sur le disque local à `TESTBED_PATH` → on exécute en **local**.
- Notre agent : le repo est **dans le conteneur** → l'agent lance le serveur avec `SWE_CONTAINER=<nom>` et `TESTBED_PATH=/testbed` → on exécute avec `docker exec`.
Tous les outils passent par `_sh` / `_read` / `_write` : un seul endroit à changer.
```python
ROOT = os.environ.get("TESTBED_PATH", "/testbed")
CONTAINER = os.environ.get("SWE_CONTAINER")          # notre convention, absente pendant les tests isolés

def _sh(cmd: str, cwd: str = ROOT, timeout: int = 120, stdin: str | None = None):
    if CONTAINER:
        argv = ["docker", "exec", "-i", "-w", cwd, CONTAINER, "bash", "-lc", cmd]
    else:
        argv = ["bash", "-lc", cmd]
    ...  # TODO: subprocess.run(argv, cwd=None if CONTAINER else cwd, input=stdin, capture_output=True, text=True, timeout=timeout)
```
C'est l'approche **(b)** du sujet (sandbox sur l'hôte, outils qui pontent vers Docker).

## Les 9 outils
| Outil | Implémentation minimale | Format de sortie |
|---|---|---|
| `read_file(filepath, start_line, end_line)` | `_read` + découpe | `12: def foo():` (cat -n, `n: ligne`) |
| `edit_file(filepath, old_str, new_str)` | `old_str` doit apparaître **exactement 1 fois** ; remplacer ; `_write` ; si `.py` → `python -m py_compile` | `"OK: 1 replacement"` + **avertissement si erreur de syntaxe** (exigé) |
| `list_files(directory, pattern)` | `find dir -name 'pattern'` (limiter à 200) | un chemin par ligne |
| `search_code(pattern, file_pattern)` | `grep -rnE --include='pattern' ...` | `/abs/path.py:42 ligne` (**espace** après le numéro, pas `:`) |
| `search_function_or_class_definition_in_code(name)` | `grep -rnE "^\s*(def\|class) name\b" --include='*.py'` (le `\|` est l'échappement du tableau, dans le code c'est une barre verticale simple) | idem search_code |
| `find_references(name, filepath, line)` | `grep -rnw name --include='*.py'` (filepath/line ignorés au minimum) | idem |
| `run_tests()` | exécuter l'`eval_script` | sortie (fin) + code retour |
| `get_patch()` | `git -c core.fileMode=false diff` dans ROOT | diff unifié |
| `run_command(command, workdir)` | `_sh(command, cwd=workdir)` | `exit_code`, `stdout`, `stderr` |

Conversion de format pour la recherche :
```python
# grep :  /testbed/sympy/core/add.py:42:    def flatten(cls, seq):
# exigé : /testbed/sympy/core/add.py:42     def flatten(cls, seq):
path, lineno, content = raw.split(":", 2)
...  # TODO: f"{path}:{lineno} {content}"
```

## `edit_file` : le retour exigé sur la syntaxe
```python
if count != 1:
    return f"ERROR: old_str found {count} times in {filepath}; it must match exactly once. Add surrounding lines."
...
res = _sh(f"python -m py_compile {shlex.quote(path)}")
if res.returncode != 0:
    return f"EDITED, BUT SYNTAX ERROR:\n{res.stderr[-800:]}\nFix it with another edit_file."
```

## `run_tests()` : d'où vient le script ? ⚠️ à vérifier
La signature n'a **aucun argument**. Proposition : l'agent copie `eval_script` dans le conteneur à `/tmp/agent/eval.sh` et passe `EVAL_SCRIPT=/tmp/agent/eval.sh` au serveur. Pour les tests isolés de la moulinette, regarder comment elle fournit le script (variable d'env ? fichier ?) dans le code de la moulinette dès que vous l'avez.
⚠️ L'eval script **applique le patch de test puis le retire** (`git checkout` des fichiers de test) : vérifier qu'après `run_tests()`, `get_patch()` ne contient **que** vos modifications.

## Pièges
- Chemins relatifs → les joindre à `ROOT`. Chemin hors `ROOT` → refuser.
- Sorties énormes (`grep` sur sympy) → tronquer (ex. 100 lignes) **et le dire** : `"[truncated: 1 532 matches, showing 100; refine the pattern]"`.
- Python dans les images SWE-bench : env conda `testbed` ⚠️ (`source /opt/miniconda3/bin/activate testbed` avant `py_compile` si `python` n'est pas le bon).
- Quoting : toujours `shlex.quote` sur les chemins et motifs.

## Notes liées
[[FLOW - Tâche SWE-bench]] · [[10 - School/10 - Agent-Smith/Index des fonctions|Index des fonctions]]
