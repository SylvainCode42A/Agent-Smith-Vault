---
type: module
title: MOD - Extraction de code
language: Python 3.10 (re, json)
path: ⚠️ proposé smith_core/agent/extract.py
status: draft
related: ["[[MOD - Boucle agent]]", "[[MOD - Prompt système]]"]
tags: [agent-smith, smith_core, module, thomas]
---
# MOD - Extraction de code

## Rôle
Transformer le **texte brut** du LLM en **code Python** pour la sandbox, plus une *notice* qui explique ce qui s'est passé. La sandbox ne voit que du Python : tout autre format est **converti** en appel de fonction Python.

## Ce qui est exigé
| Format | Exemple reçu | Python produit |
|---|---|---|
| (a) bloc Python — **principal** | ` ```python\nx = 1\n``` <end_code>` | `x = 1` |
| (b) XML Anthropic | `<invoke name="read_file"><parameter name="filepath">/testbed/a.py</parameter></invoke>` | `result = read_file(filepath='/testbed/a.py')\nprint(result)` |
| (c) JSON Hermes | `<tool_call>{"name": "read_file", "arguments": {"filepath": "a.py"}}</tool_call>` | idem |
| (d) ReAct | `Action: read_file\nAction Input: {"filepath": "a.py"}` | idem |

Et les **messages explicites** au LLM (exigé) :
- aucun bloc trouvé → `"No valid code block found. Answer with Thought: then a ```python block ending with <end_code>."`
- bloc mal formé mais interprété → dire *comment* : `"Note: your code block was not closed; I executed everything after ```python."`

Minimum acceptable : (a) solide + (b)(c)(d) en version simple (une regex chacun). On n'utilise vraiment que (a) avec Gemini/Llama ; les autres servent si un modèle du benchmark les produit.

## Fonctions à créer
| Symbole | Rôle | Appelé par | Appelle |
|---|---|---|---|
| `extract_code(text) -> tuple[str \| None, str]` | essaie les formats dans l'ordre | [[MOD - Boucle agent]] | les 4 ci-dessous |
| `_from_python_block(text)` | regex ```` ```(?:python|py)?\n(.*?)``` ```` (DOTALL) ; plusieurs blocs → les concaténer | `extract_code` | — |
| `_from_xml_invoke(text)` | `<invoke name="…">` + `<parameter name="…">` | `extract_code` | `_call_to_python` |
| `_from_hermes(text)` | `<tool_call>{json}</tool_call>` | `extract_code` | `_call_to_python` |
| `_from_react(text)` | `Action:` / `Action Input:` | `extract_code` | `_call_to_python` |
| `_call_to_python(name, args) -> str` | `result = name(k=repr(v), …)\nprint(result)` | les 3 du dessus | `repr` |

## Bout de code
```python
PY_BLOCK = re.compile(r"```(?:python|py)?\s*\n(.*?)```", re.DOTALL)

def _call_to_python(name: str, args: dict) -> str:
    kwargs = ", ".join(f"{k}={v!r}" for k, v in args.items())   # repr() => chaînes bien échappées
    return f"result = {name}({kwargs})\nprint(result)"
```
Exemple : `_call_to_python("read_file", {"filepath": "/testbed/a.py", "start_line": 1})` →
`result = read_file(filepath='/testbed/a.py', start_line=1)` puis `print(result)`.

## Le piège du `stop`
Sans `stop=["<end_code>"]` dans l'appel API, le modèle continue à écrire et **invente** une fausse `Observation:`. Avec le stop, le texte reçu **ne contient pas** `<end_code>` et souvent pas le dernier ```` ``` ```` → gérer un bloc **non fermé** (`"```python\n…"` jusqu'à la fin) et le signaler dans la notice.

## Cas limites
- Le modèle écrit `Observation:` lui-même → couper le texte à `Observation:` avant extraction.
- Bloc ```` ```json ```` ou ```` ```bash ```` → ce n'est pas du Python : l'ignorer et le dire.
- Arguments JSON invalides dans (c)/(d) → notice, pas de crash.

## Notes liées
[[MOD - Prompt système]] · [[10 - School/10 - Agent-Smith/Index des fonctions|Index des fonctions]]
