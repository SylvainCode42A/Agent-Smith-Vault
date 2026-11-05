---
type: guide
title: GUIDE - Tests et sécurité
path: tests/
status: draft
tags: [agent-smith, tests, guide, sylvain, thomas]
---
# GUIDE - Tests et sécurité

## Pourquoi
`exam_sandbox.sh` exige **100 %** des tests de sécurité, et les outils SWE-bench sont testés **un par un**. On écrit nos propres tests qui imitent ceux de la moulinette ⚠️ (on ne connaît pas leur code exact : dès que vous avez le dossier `moulinette/`, lisez ses tests et alignez-vous).

## Tests sandbox à écrire (`tests/test_sandbox.py`, pytest)
| Test | Code envoyé à `sandbox.execute` | Attendu |
|---|---|---|
| import bloqué | `import os` | erreur `ImportError` explicite, pas de crash |
| sous-module permis | `import collections.abc` | OK |
| builtin bloqué | `eval("1+1")`, `open` hors allowlist, `__import__("os")` | erreur |
| évasion dunder | `().__class__.__base__.__subclasses__()` | refusé |
| réseau | `import socket` ; `import urllib.request` | refusé |
| chemin | `open("/etc/passwd")`, `open("/tmp/agent/../../etc/passwd")` | refusé ; `open("/tmp/agent/x.txt","w")` OK |
| timeout | `while True: pass` | `timed_out=True` en ~30 s, puis la sandbox **remarche** |
| sortie partielle | `print("a"); while True: pass` | sortie contient `a` + notice partielle |
| mémoire | `x = "a" * (2 * 1024**3)` | notice mémoire, pas de crash de l'agent |
| persistance | `x = 5` puis `print(x)` | `5` |
| final_answer | `final_answer("ok")` | `final_answer == "ok"` |
| SystemExit | `raise SystemExit` | remonte côté parent |
| MCP | connecter `mcp_tools_mbpp.py`, appeler `run_tests` | JSON `success` |

## Tests outils SWE-bench (`tests/test_tools_swebench.py`)
- Un mini-repo dans `tests/fixtures/` (2-3 fichiers `.py` + `git init`), `TESTBED_PATH` pointé dessus, **sans** `SWE_CONTAINER` → mode local.
- Appeler chaque fonction **directement** (FastMCP garde les fonctions Python appelables) et vérifier le **format** : `n: ligne`, `/abs/path:42 ligne`.
- `edit_file` : 0 occurrence, 2 occurrences, erreur de syntaxe introduite → message.
- `get_patch` après un edit → diff non vide.

## Commandes
```bash
uv run pytest tests/ -q
printf 'import os\nexit\n' | uv run sandbox               # test manuel rapide
uv run sandbox --mcp-stdio "python mcp_tools_mbpp.py" sandbox_template.json
```

## Pièges dans l'ordre où on les rencontre
1. Tests de mémoire sur **macOS** : peuvent passer à tort ou planter → les lancer dans un conteneur Linux `python:3.10`.
2. Le test timeout prend 30 s → config de test avec `max_execution_time_seconds: 2`.
3. Fixture git : configurer `user.email`/`user.name` dans le repo de test sinon `git commit` échoue.

## Notes liées
[[MOD - Sandbox sécurité]] · [[MOD - MCP SWE-bench]] · [[MOD - CLI sandbox]]
