---
type: module
title: MOD - Client LLM
language: Python 3.10 (httpx ou openai SDK)
path: ⚠️ proposé smith_core/llm/client.py + keys.py
status: draft
related: ["[[MOD - Boucle agent]]", "[[GUIDE - Benchmark report]]"]
tags: [agent-smith, smith_core, module, thomas]
---
# MOD - Client LLM

## Rôle
Un seul point d'entrée `chat(messages, max_tokens, stop) -> LLMResponse` qui marche pour **n'importe quel provider**, gère **plusieurs clés** par provider avec **rotation**, **retry** sur erreurs, et mesure **tokens / latence / retries**.

## Ce qui est exigé
- Plusieurs providers et modèles (choisis par `--provider-url` et `--model-name`).
- **Plusieurs clés par provider + rotation** sur rate limit / quota (obligatoire).
- Clés **uniquement** depuis l'environnement / `.env` (une clé dans le code = échec).
- Suivi : tokens entrée/sortie, retries, latence, nombre de requêtes.
- Uniquement des **free tiers**.
- Bonus, on ne fait pas : fallback automatique vers un autre provider.

## L'astuce qui simplifie tout
Presque tous les providers gratuits exposent une API **compatible OpenAI** (`POST {base_url}/chat/completions`). Donc **un seul client**, seule l'URL change :
| Provider | `--provider-url` | Variable d'env ⚠️ convention à fixer |
|---|---|---|
| Google AI Studio (Gemini) | `https://generativelanguage.googleapis.com/v1beta/openai` | `GEMINI_API_KEY`, `GEMINI_API_KEY_2`… |
| OpenRouter | `https://openrouter.ai/api/v1` | `OPENROUTER_API_KEY`, `OPENROUTER_API_KEY_2`… |
| Groq | `https://api.groq.com/openai/v1` | `GROQ_API_KEY`, `GROQ_API_KEY_2`… |
⚠️ à vérifier : URLs et support du paramètre `stop` par chacun (déjà testé avec `scripts/try_gemini.py` ?).

## Fonctions à créer
| Symbole | Rôle | Appelé par | Appelle |
|---|---|---|---|
| `LLMResponse` (dataclass) | `text, input_tokens, output_tokens, request_time_ms, retries, api_url, model_name` | [[MOD - Boucle agent]] | — |
| `KeyRing(env_prefix)` | charge `X_API_KEY`, `X_API_KEY_2`… ; `current()`, `rotate()` | `LLMClient` | `os.environ`, `dotenv` |
| `provider_prefix(url) -> str` | `"openrouter.ai"` → `"OPENROUTER"` | `LLMClient.__init__` | — |
| `LLMClient(base_url, model)` | le client | agents (`__main__`) | `KeyRing` |
| `LLMClient.chat(messages, max_tokens, stop) -> LLMResponse` | 1 appel logique, N requêtes HTTP | [[MOD - Boucle agent]] | `httpx.post`, `KeyRing.rotate` |

## Squelette (à trous)
```python
RETRYABLE = {429, 500, 502, 503, 504}

def chat(self, messages, max_tokens, stop):
    retries = 0
    t0 = time.monotonic()
    while True:
        try:
            r = httpx.post(f"{self.base_url}/chat/completions",
                           headers={"Authorization": f"Bearer {self.keys.current()}"},
                           json={"model": self.model, "messages": messages,
                                 "max_tokens": max_tokens, "stop": stop, "temperature": 0},
                           timeout=60)
        except httpx.TimeoutException:
            r = None
        if r is not None and r.status_code == 200:
            break
        if retries >= MAX_RETRIES or (r is not None and r.status_code not in RETRYABLE):
            raise LLMError(...)          # TODO: message lisible (code + début du body)
        retries += 1
        self.keys.rotate()               # 429 => clé suivante
        time.sleep(...)                  # TODO: backoff 2, 4, 8 s
    data = r.json()
    usage = data.get("usage", {})
    ...  # TODO: LLMResponse(text=data["choices"][0]["message"]["content"] or "", ...)
```
Exemple : clé 1 → 429, clé 2 → 200. `retries = 1`, donc `total_requests` de ce step = 2.

## Pièges
- `content` peut être `None` (modèle qui ne répond que des tool calls) → `or ""`.
- Pas de `usage` dans la réponse → estimer `len(texte) // 4`, et le dire dans le README.
- Modèles « reasoning » : leurs tokens de réflexion comptent dans la limite de sortie → **préférer des modèles non-reasoning** (ou désactiver le thinking : pour Gemini 2.5 Flash ⚠️ à vérifier comment via l'API compatible OpenAI).
- `request_time_ms` : choisir *total retries compris* ou *dernière requête* et l'écrire dans le README ⚠️ décision à prendre.
- Le `.env` est passé par la moulinette (`--envfile`) : charger avec `dotenv` **sans écraser** les variables déjà présentes.

## Notes liées
[[GUIDE - Benchmark report]] · [[10 - School/10 - Agent-Smith/Index des fonctions|Index des fonctions]]
