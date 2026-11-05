---
type: module
title: MOD - Sandbox sécurité
language: Python 3.10 (builtins, ast, sys.addaudithook, resource)
path: ⚠️ proposé smith_core/sandbox/security.py
status: draft
related: ["[[MOD - Sandbox exécuteur]]", "[[GUIDE - Tests et sécurité]]"]
tags: [agent-smith, smith_core, sandbox, module, sylvain]
---
# MOD - Sandbox sécurité

## Rôle
Les 6 barrières testées par `exam_sandbox.sh` (il faut **toutes** les passer), avec **la bibliothèque standard seulement** (RestrictedPython interdit).

| Test de la moulinette | Notre barrière | Où |
|---|---|---|
| import block | `__import__` remplacé, allowlist (`x.*` = sous-modules) | builtins de l'enfant |
| builtin block | `eval`, `exec`, `compile`, `open` brut, `input`, `breakpoint`, `globals`… retirés ; `getattr` sur `_…` refusé ; AST sans dunders | builtins + `check_ast` |
| network block | `socket` non importable + hook d'audit `socket.*` | allowlist + audit |
| path restrict | `open` remplacé (realpath dans `allowed_directories`) + hook d'audit `open` | builtins + audit |
| timeout | kill par le parent | [[MOD - Sandbox exécuteur]] |
| memory limit | `resource.setrlimit(RLIMIT_AS)` dans l'enfant | `apply_limits` |
| MCP protocol | outils découverts et appelables | [[MOD - Client MCP]] |

## Fonctions à créer
| Symbole | Rôle | Appelé par | Appelle |
|---|---|---|---|
| `is_import_allowed(name, allowlist) -> bool` | `collections.abc` OK si `collections.*` | `guarded_import` | — |
| `guarded_import(name, globals=None, locals=None, fromlist=(), level=0)` | remplace `__import__` | `import` du code LLM | `is_import_allowed`, vrai `__import__` |
| `safe_open(file, mode="r", ...)` | refuse hors allowlist | code LLM | `is_path_allowed` |
| `is_path_allowed(path, dirs) -> bool` | `os.path.realpath` + préfixe **avec** séparateur | `safe_open`, `audit_hook` | — |
| `safe_builtins(config) -> dict` | copie filtrée de `builtins` | [[MOD - Sandbox exécuteur]] `_worker` | — |
| `check_ast(code)` | refuse `__xxx__` en attribut et en nom | `_worker` | `ast.walk` |
| `apply_limits(config)` | `RLIMIT_AS` | `_worker` | `resource.setrlimit` |
| `install_guards(config)` | `sys.addaudithook(audit_hook)` | `_worker` | — |
| `preimport(allowlist)` | importe les modules permis avant le hook | `_worker` | `importlib.import_module` |

## Bouts de code
```python
def is_import_allowed(name: str, allowlist: list[str]) -> bool:
    for entry in allowlist:
        if entry.endswith(".*"):
            base = entry[:-2]
            if name == base or name.startswith(base + "."):
                return True
        elif name == entry:
            return True
    return False
```
Exemples : `("collections.abc", ["collections.*"])` → True ; `("os", défauts)` → False ; `("math", ["math"])` → True ; `("mathx", ["math.*"])` → **False** (d'où le `+ "."`).

```python
def is_path_allowed(path, dirs):
    real = os.path.realpath(path)            # résout ../ et liens symboliques
    return any(real == d or real.startswith(d.rstrip("/") + "/") for d in map(os.path.realpath, dirs))
```
Exemple : `/testbed/../etc/passwd` → realpath `/etc/passwd` → refusé. `/testbedX/a` → refusé grâce au `/`.

```python
BLOCKED_EVENTS = ("socket.", "subprocess.", "os.system", "os.exec", "os.posix_spawn", "os.fork")

def audit_hook(event, args):
    if event == "open":
        ...      # TODO: args[0] est le chemin (peut être un int = fd => laisser passer)
    elif event.startswith(BLOCKED_EVENTS):
        raise PermissionError(f"Sandbox: '{event}' is not allowed")
```
Pourquoi un **hook d'audit** en plus ? Il attrape tout ce qui passerait à côté des builtins (un module autorisé qui ouvre un fichier, `ctypes`…). Il **ne peut pas être retiré** une fois installé. ⚠️ Lever `PermissionError` (sous-classe d'`OSError`) : `linecache`/`traceback` l'attrapent proprement quand ils essaient de lire des sources hors allowlist.

## L'évasion classique à bloquer
```python
().__class__.__base__.__subclasses__()   # => accès à toutes les classes, dont des Popen...
```
`check_ast` refuse tout `ast.Attribute` dont `attr` commence par `__`, et tout `ast.Name` comme `__builtins__`, `__import__`. Et `getattr(x, "__class__")` → le `getattr` sûr refuse les noms qui commencent par `_`.

## Mémoire : attention macOS
`RLIMIT_AS` fonctionne sous **Linux** ; sous **macOS** il est mal respecté ⚠️ à vérifier. La moulinette tourne probablement sous Linux : testez dans un conteneur `python:3.10`. Un `MemoryError` dans l'enfant est une `Exception` → l'attraper et renvoyer `[MEMORY]`.

## Cas limites
- `from . import x` (`level > 0`) → refuser.
- `__import__` appelé par un module **autorisé** pour ses propres dépendances : il utilise les vrais builtins de son module, donc pas bloqué — c'est voulu.
- `open(3)` (descripteur entier) → refuser dans `safe_open`.

## Notes liées
[[GUIDE - Tests et sécurité]] · [[10 - School/10 - Agent-Smith/Index des fonctions|Index des fonctions]]
