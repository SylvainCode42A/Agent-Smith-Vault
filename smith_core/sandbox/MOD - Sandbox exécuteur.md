---
type: module
title: MOD - Sandbox exécuteur
language: Python 3.10 (multiprocessing, resource, io)
path: ⚠️ proposé smith_core/sandbox/executor.py
status: draft
related: ["[[OVR - smith_core]]", "[[MOD - Sandbox sécurité]]", "[[MOD - Client MCP]]", "[[MOD - Manuel sandbox]]", "[[FLOW - Appel outil MCP]]"]
tags: [agent-smith, smith_core, sandbox, module, sylvain]
---
# MOD - Sandbox exécuteur

## Rôle
Exécuter le code du LLM **dans un process enfant** qu'on peut tuer, avec un **namespace persistant** (les variables restent d'un tour à l'autre), les **wrappers d'outils MCP** et `final_answer` injectés, et renvoyer un `ExecResult` (sortie, réponse finale, notices).

## Décision proposée ⚠️ à valider
**Un process enfant persistant** (`multiprocessing.Process` + `Pipe`) plutôt qu'un `exec` dans le process agent.
| Option | Timeout | Mémoire | Réseau/FS | Variables persistantes |
|---|---|---|---|---|
| `exec` dans le même process | ❌ impossible de tuer un `while True` proprement | ❌ `setrlimit` tuerait l'agent | 😐 | ✅ |
| nouveau process **à chaque** exécution | ✅ | ✅ | ✅ | ❌ |
| **process persistant** (choisi) | ✅ `kill()` puis relance | ✅ `setrlimit` dans l'enfant | ✅ hook d'audit dans l'enfant | ✅ (sauf après un kill : on le dit au LLM) |

## Le protocole du Pipe (messages = dicts)
| Sens | Message | Sens du message |
|---|---|---|
| parent → enfant | `{"type": "exec", "code": "..."}` | exécute ça |
| enfant → parent | `{"type": "stdout", "data": "..."}` | un morceau de `print` (envoyé **au fil de l'eau**) |
| enfant → parent | `{"type": "tool", "name": "read_file", "kwargs": {...}}` | appelle un outil MCP pour moi |
| parent → enfant | `{"type": "tool_result", "ok": True, "value": "..."}` | réponse de l'outil |
| enfant → parent | `{"type": "done", "final": None \| "...", "error": None \| "traceback"}` | fini |
| enfant → parent | `{"type": "control", "exc": "SystemExit"}` | à re-lever côté parent (exigé) |

Pourquoi envoyer stdout **au fil de l'eau** ? Si on tue l'enfant au timeout, ce qui est déjà arrivé chez le parent constitue la **sortie partielle** exigée par le sujet.

## Fonctions à créer
| Symbole | Rôle | Appelé par | Appelle |
|---|---|---|---|
| `Sandbox.__init__(config, mcp)` / `start()` | crée Pipe + enfant, connecte le MCP | agents, [[MOD - CLI sandbox]] | `_spawn`, [[MOD - Client MCP]] `connect` |
| `Sandbox.execute(code) -> ExecResult` | envoie `exec`, sert les `tool`, gère deadline | [[MOD - Boucle agent]], [[MOD - CLI sandbox]] | [[MOD - Client MCP]] `call_tool`, `_restart` |
| `Sandbox.manual() -> str` | texte des outils | [[MOD - Prompt système]] | [[MOD - Manuel sandbox]] `build_manual` |
| `Sandbox.close()` | tue l'enfant, ferme le MCP | agents (finally) | — |
| `_worker(conn, config, tool_names)` | **corps de l'enfant** | `multiprocessing` | [[MOD - Sandbox sécurité]] `apply_limits`, `install_guards`, `safe_builtins`, `check_ast` |
| `_make_tool_wrapper(conn, name)` | fonction Python qui fait l'aller-retour Pipe | `_worker` | — |
| `_PipeStdout.write(s)` | remplace `sys.stdout` dans l'enfant | `print` | `conn.send` |
| `FinalAnswer(Exception)` | levée par `final_answer(x)` pour arrêter l'exécution | `_worker` | — |
| `truncate(text, limit) -> (text, notice)` | garde début + fin | `execute` | — |

## Côté enfant (à trous)
```python
def _worker(conn, config, tool_names):
    apply_limits(config)                    # mémoire (resource.setrlimit)
    preimport(config.authorized_imports)    # AVANT le hook d'audit
    install_guards(config)                  # hook d'audit : fichiers, réseau, subprocess
    sys.stdout = sys.stderr = _PipeStdout(conn)
    ns = {"__builtins__": safe_builtins(config), "final_answer": _final_answer}
    for name in tool_names:
        ns[name] = _make_tool_wrapper(conn, name)
    while True:
        msg = conn.recv()
        try:
            check_ast(msg["code"])          # refuse __class__, __subclasses__...
            exec(compile(msg["code"], "<agent>", "exec"), ns)
            conn.send({"type": "done", "final": None, "error": None})
        except FinalAnswer as f:
            conn.send({"type": "done", "final": str(f.value), "error": None})
        except (KeyboardInterrupt, SystemExit) as e:
            ...                             # TODO: prévenir le parent (type "control"), puis sortir
        except Exception:
            ...                             # TODO: traceback court (sans les frames internes)
```
```python
def _make_tool_wrapper(conn, name):
    def tool(*args, **kwargs):
        if args:
            ...                              # TODO: mapper les positionnels sur les noms du schéma
        conn.send({"type": "tool", "name": name, "kwargs": kwargs})
        reply = conn.recv()
        if not reply["ok"]:
            raise RuntimeError(reply["value"])
        return reply["value"]
    tool.__name__ = name
    return tool
```

## Côté parent : la deadline qui se met en pause
Le sujet : le timeout s'applique **au code sandboxé seulement**, pas aux outils MCP (un `run_tests()` SWE-bench peut durer 2 min). Donc on **arrête le chrono** pendant un appel d'outil.
```python
remaining = config.max_execution_time_seconds
while True:
    t0 = time.monotonic()
    if not self.conn.poll(remaining):           # rien reçu à temps => timeout
        return self._timeout_result(chunks)     # kill + restart + notice "partial"
    msg = self.conn.recv()
    remaining -= time.monotonic() - t0
    if msg["type"] == "tool":
        value = self.mcp.call_tool(msg["name"], msg["kwargs"])   # chrono en pause
        ...
```
Exemple : `max_execution_time_seconds = 30`. Le code fait 2 s de calcul, appelle `run_tests()` (95 s), refait 1 s → total sandbox 3 s ✅, pas de timeout.

## Les notices obligatoires (le LLM ne doit jamais deviner)
- Timeout : `"[TIMEOUT] Execution exceeded 30s and was killed. Output above is partial. Sandbox variables were reset."`
- Troncature : `"[TRUNCATED] Output was 18 342 chars; showing first 2 000 and last 2 000."`
- Mémoire : `"[MEMORY] Execution exceeded 512 MB and was stopped."`
- Rien imprimé : `"(no output — use print() to see values)"` — très utile, les LLM oublient `print`.

## Cas limites
- Enfant mort (segfault, OOM-kill) → `conn.poll` lève `EOFError` → relancer et le dire.
- `final_answer` appelé puis le code continue → l'exception l'arrête net, c'est voulu.
- Objets non picklables dans `kwargs` → les convertir en `str` ou refuser proprement.
- macOS : `multiprocessing` utilise `spawn` → `_worker` et ses arguments doivent être **picklables** (fonction de module, pas de lambda).

## Notes liées
[[MOD - Sandbox sécurité]] · [[FLOW - Appel outil MCP]] · [[10 - School/10 - Agent-Smith/Index des fonctions|Index des fonctions]]
