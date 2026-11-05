---
type: module
title: MOD - CLI sandbox
language: Python 3.10 (argparse)
path: smith_core/sandbox/cli.py (existe déjà)
status: draft
related: ["[[MOD - Sandbox exécuteur]]", "[[MOD - Client MCP]]", "[[GUIDE - Tests et sécurité]]"]
tags: [agent-smith, smith_core, sandbox, module, sylvain]
---
# MOD - CLI sandbox

## Rôle
La commande `uv run sandbox` : un **REPL** qui lit du code tapé, l'exécute dans la **même** sandbox que l'agent (mêmes restrictions, outils MCP, `final_answer`), affiche le résultat ou l'erreur, et recommence. C'est très probablement par là que `exam_sandbox.sh` teste la sécurité ⚠️ à vérifier.

## Les 5 lancements exigés
```bash
uv run sandbox
uv run sandbox sandbox_template.json
uv run sandbox --mcp-stdio "python mcp_tools_mbpp.py" sandbox_template.json
uv run sandbox --mcp-server http://localhost:8000/mcp
uv run sandbox --mcp-stdio "python mcp_tools_swebench.py" sandbox_template.json
```
Dans `pyproject.toml` :
```toml
[project.scripts]
sandbox = "smith_core.sandbox.cli:main"
```

## Fonctions à créer
| Symbole | Rôle | Appelé par | Appelle |
|---|---|---|---|
| `parse_args(argv)` | `config` positionnel optionnel, `--mcp-stdio`, `--mcp-server` (exclusifs) | `main` | `argparse` |
| `main()` | construit la sandbox, lance le REPL, ferme tout | `uv run sandbox` | [[MOD - Modèles Pydantic]] `load_config`, [[MOD - Client MCP]], [[MOD - Sandbox exécuteur]] |
| `read_entry() -> str \| None` | une entrée ; multi-lignes si la ligne finit par `:` (jusqu'à une ligne vide) ; `None` sur EOF/`exit` | `repl` | `input(">>> ")`, `input("... ")` |
| `repl(sandbox)` | boucle lire → exécuter → afficher | `main` | `Sandbox.execute` |

```python
def repl(sandbox):
    print(sandbox.manual())                       # utile : on voit les outils découverts
    while True:
        code = read_entry()
        if code is None:                          # "exit" ou Ctrl+D
            break
        result = sandbox.execute(code)
        ...                                       # TODO: afficher output, notices ; si final_answer : "FINAL ANSWER: ..."
```

## Cas limites
- `Ctrl+D` → `EOFError` → sortie **propre** (code 0), sandbox et serveur MCP fermés.
- `Ctrl+C` pendant une exécution → doit remonter (exigé : `KeyboardInterrupt` non avalé) ; dans le REPL on peut l'attraper **au niveau du REPL** pour revenir au prompt.
- Entrée lue depuis un pipe (`echo "import os" | uv run sandbox`) : pas de TTY, `input()` marche quand même ; ne pas afficher de couleurs.
- Fichier de config invalide → message clair, pas de traceback Pydantic brut.

## Notes liées
[[GUIDE - Tests et sécurité]] · [[10 - School/10 - Agent-Smith/Index des fonctions|Index des fonctions]]
