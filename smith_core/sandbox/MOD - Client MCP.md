---
type: module
title: MOD - Client MCP
language: Python 3.10 (mcp SDK officiel, asyncio, threading)
path: ⚠️ proposé smith_core/sandbox/mcp_client.py
status: draft
related: ["[[MOD - Sandbox exécuteur]]", "[[MOD - Manuel sandbox]]", "[[MOD - MCP MBPP]]", "[[MOD - MCP SWE-bench]]", "[[FLOW - Appel outil MCP]]"]
tags: [agent-smith, smith_core, sandbox, module, sylvain]
---
# MOD - Client MCP

## Rôle
Se connecter à **n'importe quel** serveur MCP (le nôtre ou un inconnu fourni par la moulinette), en **stdio** ou en **HTTP streamable**, découvrir ses **tools / resources / prompts**, et appeler un outil de façon **synchrone** pour la sandbox.

## Ce qui est exigé
- Transports **stdio et streamable HTTP** (les deux).
- Découverte **dynamique** : aucun nom d'outil codé en dur dans la sandbox.
- Exposer tools, **resources et prompts**.
- Testé avec un **serveur inconnu** → ne rien supposer sur les noms et arguments.

## Le problème : MCP est `async`, notre sandbox est synchrone
Le SDK `mcp` s'utilise avec `async with`. Solution simple : **une boucle asyncio dans un thread** qui garde la session ouverte, et des appels synchrones qui lui envoient des coroutines.
```mermaid
sequenceDiagram
    participant Sync as Sandbox.execute (thread principal)
    participant Loop as Thread asyncio
    participant Srv as Serveur MCP
    Sync->>Loop: run_coroutine_threadsafe(session.call_tool(...))
    Loop->>Srv: JSON-RPC tools/call
    Srv-->>Loop: CallToolResult
    Loop-->>Sync: future.result()
```

## Fonctions à créer
| Symbole | Rôle | Appelé par | Appelle |
|---|---|---|---|
| `McpClient.from_stdio(command: str)` | `shlex.split("python mcp_tools_mbpp.py")` | [[MOD - CLI sandbox]], agents | `StdioServerParameters` |
| `McpClient.from_http(url)` | HTTP streamable | [[MOD - CLI sandbox]] | `streamablehttp_client` |
| `connect()` | démarre le thread, ouvre la session, `initialize()`, met en cache `list_tools/resources/prompts` | [[MOD - Sandbox exécuteur]] `start` | `_run_forever` |
| `tools -> list[Tool]` | schémas (nom, description, `inputSchema`) | [[MOD - Manuel sandbox]], exécuteur | — |
| `call_tool(name, kwargs) -> str` | appel synchrone, joint les `TextContent` | exécuteur | `session.call_tool` |
| `read_resource(uri)`, `get_prompt(name, args)` | resources / prompts | wrappers dans la sandbox | session |
| `close()` | signale l'arrêt, joint le thread | `Sandbox.close` | — |

## Squelette (à trous)
```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from mcp.client.streamable_http import streamablehttp_client

async def _run_forever(self):
    async with self._transport() as streams:            # stdio_client(params) ou streamablehttp_client(url)
        read, write = streams[0], streams[1]           # HTTP renvoie 3 valeurs, stdio 2
        async with ClientSession(read, write) as session:
            await session.initialize()
            self.session = session
            self.tools = (await session.list_tools()).tools
            ...                                         # TODO: resources, prompts (try/except : serveur sans resources)
            self._ready.set()
            await self._stop.wait()                     # garde la session ouverte

def call_tool(self, name, kwargs) -> str:
    fut = asyncio.run_coroutine_threadsafe(self.session.call_tool(name, kwargs), self._loop)
    result = fut.result(timeout=...)                    # TODO: timeout large (run_tests SWE ~ minutes)
    text = "\n".join(c.text for c in result.content if getattr(c, "type", "") == "text")
    ...                                                 # TODO: si result.isError -> lever avec le texte
```
⚠️ à vérifier : noms exacts des imports avec la version du SDK installée (`uv add mcp`).

## Pièges
- `list_resources()` / `list_prompts()` lèvent une erreur si le serveur ne les supporte pas → `try/except` et liste vide.
- Le serveur stdio écrit ses logs sur **stdout** → ça casse le protocole JSON-RPC. Côté serveurs : logs sur **stderr** uniquement.
- Le process serveur doit hériter de l'environnement (`TESTBED_PATH`, nom du conteneur) : passer `env=dict(os.environ)` à `StdioServerParameters`.

## Notes liées
[[FLOW - Appel outil MCP]] · [[10 - School/10 - Agent-Smith/Index des fonctions|Index des fonctions]]
