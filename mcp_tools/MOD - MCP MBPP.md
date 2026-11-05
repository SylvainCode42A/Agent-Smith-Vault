---
type: module
title: MOD - MCP MBPP
language: Python 3.10 (mcp FastMCP, subprocess)
path: mcp_tools_mbpp.py (racine du repo, imposé)
status: draft
related: ["[[MOD - Client MCP]]", "[[MOD - Agent MBPP]]", "[[FLOW - Tâche MBPP]]"]
tags: [agent-smith, mcp, module, sylvain]
---
# MOD - MCP MBPP

## Rôle
Serveur MCP (process séparé) qui expose **un seul outil obligatoire** : `run_tests(code, test_list)` → JSON `{"success": bool, "output": str}`. C'est le plus petit fichier du projet : le faire **en premier** pour tester tout le reste de la chaîne.

## Fonctions à créer
| Symbole | Rôle | Appelé par | Appelle |
|---|---|---|---|
| `run_tests(code: str, test_list: list[str]) -> str` | exécute code + asserts dans un **sous-process** avec timeout | [[MOD - Client MCP]] (depuis la sandbox) | `subprocess.run` |
| `main()` | `--http` → streamable HTTP, sinon stdio | `python mcp_tools_mbpp.py` | `mcp.run` |

## Squelette (à trous)
```python
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("mbpp-tools")

@mcp.tool()
def run_tests(code: str, test_list: list[str]) -> str:
    """Run a candidate solution against assert statements. Returns JSON {success, output}."""
    script = code + "\n" + RUNNER.format(tests=repr(test_list))   # TODO: écrire RUNNER
    try:
        proc = subprocess.run([sys.executable, "-c", script], capture_output=True, text=True, timeout=10)
    except subprocess.TimeoutExpired:
        return json.dumps({"success": False, "output": "Timeout after 10s"})
    ...  # TODO: success = (returncode == 0) ; output = stdout + stderr tronqués

if __name__ == "__main__":
    mcp.run()          # stdio ; avec --http : mcp.run(transport="streamable-http")
```
Idée pour `RUNNER` : exécuter **chaque** assert dans un `try` et imprimer `PASS`/`FAIL: <assert>` → le LLM voit **lequel** échoue, ce qui économise une itération.

Exemple : `code = "def add(a,b): return a-b"`, `test_list = ["assert add(1,2)==3", "assert add(0,0)==0"]` →
`{"success": false, "output": "FAIL: assert add(1,2)==3\nPASS: assert add(0,0)==0"}`

## Pièges
- `test_imports` de la tâche (ex. `import math`) ne sont pas dans la signature → l'agent les **met en tête du code** ou dans les tests. ⚠️ décision.
- **Rien sur stdout** en dehors du protocole (pas de `print` de debug dans le serveur stdio).
- Le code s'exécute **hors sandbox** (c'est un outil MCP) : d'où le sous-process + timeout.

## Notes liées
[[FLOW - Tâche MBPP]] · [[10 - School/10 - Agent-Smith/Index des fonctions|Index des fonctions]]
