---
type: tickets
title: TICK - Tickets Agent Smith
status: draft
created: 2026-11-03
tags: [agent-smith, tickets]
related: ["[[10 - School/10 - Agent-Smith/00 - Accueil|00 - Accueil]]", "[[OVR - smith_core]]"]
---
# Tickets Agent Smith — strict minimum

## Légende
🟢 **Sylvain** — *exécution & sécurité* : sandbox, client MCP, CLI, serveurs MCP
🔵 **Thomas** — *intelligence* : client LLM, boucle, extraction, prompts, agents MBPP / SWE-bench
🟣 **Ensemble**

## Pourquoi cette répartition
- **Une seule frontière entre vous** : l'interface `Sandbox.execute(code) -> ExecResult` ([[OVR - smith_core]]). Sylvain fournit ce qui *exécute*, Thomas ce qui *décide quoi exécuter*. Vous ne touchez presque jamais aux mêmes fichiers → pas de conflits git.
- **Thomas n'attend pas Sylvain** : il code la boucle contre un `FakeSandbox` (simple `exec`) dès le jour 1.
- **Sylvain n'attend pas Thomas** : il teste sandbox + MCP à la main via `uv run sandbox`, sans LLM.
- **Charge équilibrée** : Sylvain a la partie la plus *technique* (sécurité, async, Docker exec) ; Thomas la partie la plus *itérative* (faire réussir le LLM sous budget, ce qui demande beaucoup d'essais).
- **Les livrables qui notent tout le monde** (benchmark, README, soutenance) sont **à deux** : chacun doit pouvoir expliquer tout le projet.

## Jalons (on se synchronise à chacun)
| Jalon | Preuve | Débloque |
|---|---|---|
| **M0** — contrat figé | `models.py` + `ExecResult`/`Sandbox` mergés dans `main` | tout |
| **M1** — MBPP de bout en bout | task 2 résolue avec la **vraie** sandbox, `validate mbpp` OK | phase 2 |
| **M2** — MBPP + sandbox « exam-ready » | ≥ 8/10 tâches MBPP en local, tous les tests sécurité verts | phase 3 |
| **M3** — SWE-bench de bout en bout | `sympy__sympy-14711` résolue, conteneur supprimé | benchmark |
| **M4** — rendu | rapport, README, repo propre | soutenance |

---

## Phase 0 — Fondations 🟣 (½ journée, ensemble)
- [x] 🟣 `pyproject.toml` : Python **3.10**, deps `pydantic`, `mcp`, `httpx`, `python-dotenv`, `pytest` ; script `sandbox = "smith_core.sandbox.cli:main"` → *preuve : `uv run sandbox --help`*
- [x] 🟣 `smith_core/models.py` : les 5 modèles copiés du sujet → [[MOD - Modèles Pydantic]]
- [x] 🟣 Contrat `ExecResult` + signatures `Sandbox` + `FakeSandbox` → [[OVR - smith_core]]
- [x] 🟣 `.gitignore` (`.env`, `__pycache__`, `cache/`), `.env.example`, retirer `Archive ZIP.zip`
- [x] 🟣 Branches `dev-sylvain` / `dev-thomas`, GitHub Project avec ces tickets
- [x] 🟣 Récupérer le dossier `moulinette/` et lancer `moulinette_eval dump mbpp` une fois

## Phase 1 — MBPP qui marche (jusqu'à M1)
### 🟢 Sylvain
- [ ] `mcp_tools_mbpp.py` : `run_tests` (sous-process + timeout, PASS/FAIL par assert), stdio → [[MOD - MCP MBPP]]
- [ ] `McpClient` stdio : connect, `list_tools`, `call_tool` synchrone (thread asyncio) → [[MOD - Client MCP]]
- [ ] `Sandbox` exécuteur : enfant persistant, Pipe, stdout au fil de l'eau, wrappers d'outils, `final_answer` → [[MOD - Sandbox exécuteur]]
- [ ] `cli.py` : REPL (`exit`, Ctrl+D), options `--mcp-stdio`, config JSON → [[MOD - CLI sandbox]]
- *preuve : dans `uv run sandbox --mcp-stdio "python mcp_tools_mbpp.py"`, taper `print(run_tests(code="def f(): return 1", test_list=["assert f()==1"]))` → `success: true`*
### 🔵 Thomas
- [x] `LLMClient.chat` compatible OpenAI + `KeyRing` (plusieurs clés, rotation sur 429) + retries comptés + usage tokens → [[MOD - Client LLM]]
- [x] `extract_code` format (a) + bloc non fermé + messages « pas de code » → [[MOD - Extraction de code]]
- [ ] `run_agent` + `StepMetrics` complètes + `SolutionOutput` → [[MOD - Boucle agent]]
- [ ] Prompt MBPP court (≤ 700 tokens) → [[MOD - Prompt système]]
- [ ] `agent_mbpp/__main__.py` (4 options + `--max-iterations`, JSON toujours écrit) → [[MOD - Agent MBPP]]
- *preuve : avec `FakeSandbox`, une tâche MBPP produit un `solution.json` valide*
### 🟣 M1 : brancher la vraie sandbox, résoudre la task 2, `validate mbpp`

## Phase 2 — Exam MBPP + exam sandbox (jusqu'à M2)
### 🟢 Sylvain
- [ ] Imports : `guarded_import` + allowlist avec `.*` → [[MOD - Sandbox sécurité]]
- [ ] Builtins restreints + `getattr` sûr + `check_ast` (dunders)
- [ ] `safe_open` + hook d'audit (fichiers, réseau, subprocess)
- [ ] Timeout (kill + relance + sortie partielle) et mémoire (`RLIMIT_AS`) + notices
- [ ] Troncature des sorties + notice ; `SystemExit`/`KeyboardInterrupt` propagés
- [ ] Transport **HTTP streamable** (client + `--http` côté serveurs) ; resources & prompts exposés
- [ ] `build_manual` dynamique → [[MOD - Manuel sandbox]]
- [ ] `tests/test_sandbox.py` (tableau de [[GUIDE - Tests et sécurité]]) ; tests mémoire sous Linux
### 🔵 Thomas
- [ ] Gardes de budget (entrée, sortie, temps) avec marges + fallback MBPP
- [ ] Extraction formats (b) XML, (c) Hermes, (d) ReAct + notice « interprété comme… »
- [ ] Observation : notices en tête, `(no output…)`
- [ ] Lancer 10–20 tâches MBPP, ajuster le prompt et le choix du modèle → ≥ 80 %
- [ ] `tests/test_extract.py`
### 🟣 M2 : simuler `exam_mbpp.sh` (5 tâches au hasard) et `exam_sandbox.sh`

## Phase 3 — SWE-bench (jusqu'à M3)
### 🟢 Sylvain
- [ ] `mcp_tools_swebench.py` : helper `_sh/_read/_write` local **et** `docker exec` → [[MOD - MCP SWE-bench]]
- [ ] Les 9 outils aux formats imposés ; `edit_file` avec vérif syntaxe ; troncatures signalées
- [ ] Fixture git + `tests/test_tools_swebench.py` (mode local, `TESTBED_PATH`)
- [ ] Vérifier comment la moulinette fournit l'eval script à `run_tests()` en test isolé ⚠️
### 🔵 Thomas
- [ ] Cycle Docker : `start_container`, `copy_eval_script`, `remove_container` idempotent → [[MOD - Agent SWE-bench]]
- [ ] Handler SIGTERM + `finally` (nettoyage + JSON partiel)
- [ ] Prompt SWE (méthode localiser → lire → éditer → tester → soumettre)
- [ ] Fallback `get_patch()` ; troncature des observations ; budgets 30 / 290k / 9,5k / 850 s
### 🟣 M3 : `sympy__sympy-14711`, puis `sympy__sympy-13480` et `pydata__xarray-4629`

## Phase 4 — Benchmark, README, rendu (jusqu'à M4) 🟣
- [ ] 🟣 Choisir 5 modèles / 3 providers, script `benchmark/run.sh` → [[GUIDE - Benchmark report]]
- [ ] 🟢 Runs modèles 1–3 (+ validation) · 🔵 Runs modèles 4–5 + **ablation**
- [ ] 🔵 `BENCHMARK_REPORT.md` tableaux · 🟢 sections fiabilité & métriques intermédiaires · 🟣 conclusions
- [ ] 🟢 README : Sandbox design, Tool implementation · 🔵 README : Agent loop, Benchmark · 🟣 Description, Instructions, Resources/IA, Architecture → [[GUIDE - README et rendu]]
- [ ] 🟣 Check-list du rendu (clés, fichiers à la racine, propreté)
- [ ] 🟣 Répétition soutenance : chacun explique la partie de l'autre + 3 modifs live chronométrées

## Workflow Git
1. `main` toujours stable. Branches `dev-sylvain`, `dev-thomas` (ou une branche par ticket).
2. Une PR par ticket ; l'autre relit (c'est aussi ce que le sujet appelle « collective intelligence »).
3. Merge à chaque jalon ; `git pull origin main` des deux côtés.
4. Point de 10 min à chaque jalon : ce qui marche, ce qui bloque, prochain jalon.
