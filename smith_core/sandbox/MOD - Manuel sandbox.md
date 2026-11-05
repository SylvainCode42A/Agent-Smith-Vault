---
type: module
title: MOD - Manuel sandbox
language: Python 3.10
path: ⚠️ proposé smith_core/sandbox/manual.py
status: draft
related: ["[[MOD - Client MCP]]", "[[MOD - Prompt système]]"]
tags: [agent-smith, smith_core, sandbox, module, sylvain]
---
# MOD - Manuel sandbox

## Rôle
Générer **automatiquement** le texte qui explique au LLM les fonctions disponibles, à partir des schémas du serveur MCP connecté. Change un serveur → le manuel change tout seul (exigé).

## Ce qu'il doit contenir
1. Une ligne par outil : **signature Python** + description.
2. `final_answer(answer: str)` — toujours présent, ajouté par la sandbox.
3. Les resources / prompts (ou comment y accéder).
4. Les limites utiles au LLM : imports autorisés, dossiers autorisés, timeout.

## De ce schéma…
```json
{"name": "read_file", "description": "Read a file with line numbers.",
 "inputSchema": {"type": "object",
   "properties": {"filepath": {"type": "string"}, "start_line": {"type": "integer"}, "end_line": {"type": "integer"}},
   "required": ["filepath"]}}
```
## …à cette ligne
```text
- read_file(filepath: str, start_line: int = None, end_line: int = None) -> str
    Read a file with line numbers.
```

## Fonctions à créer
| Symbole | Rôle | Appelé par | Appelle |
|---|---|---|---|
| `build_manual(tools, resources, prompts, config) -> str` | le texte complet | [[MOD - Sandbox exécuteur]] `manual` | `tool_signature` |
| `tool_signature(tool) -> str` | requis d'abord, optionnels `= None` | `build_manual` | `JSON_TO_PY` |
| `JSON_TO_PY` | `{"string": "str", "integer": "int", "number": "float", "boolean": "bool", "array": "list", "object": "dict"}` | `tool_signature` | — |

```python
def tool_signature(tool) -> str:
    props = tool.inputSchema.get("properties", {})
    required = tool.inputSchema.get("required", [])
    parts = [f"{n}: {JSON_TO_PY.get(p.get('type'), 'Any')}" for n, p in props.items() if n in required]
    ...  # TODO: ajouter les optionnels avec " = None"
    return f"{tool.name}({', '.join(parts)}) -> str"
```

## Budget MBPP
Le manuel MBPP doit tenir en **~150 tokens** : `run_tests` + `final_answer` + 1 ligne d'imports. Paramètre `compact=True` qui coupe les descriptions longues à la première phrase.

## Notes liées
[[MOD - Prompt système]] · [[10 - School/10 - Agent-Smith/Index des fonctions|Index des fonctions]]
