---
type: overview
title: Explications — tout le projet en une note
status: draft
created: 2026-11-03
tags: [agent-smith, explications]
---
# Explications — tout le projet en une note

À lire après [[10 - School/10 - Agent-Smith/00 - Accueil|00 - Accueil]] : la vue « j'ai tout compris » avant d'ouvrir les MOD.

## 1. L'idée en 30 secondes
Un LLM ne peut rien *faire* : il écrit du texte. Notre programme prend ce texte, en extrait du Python, l'exécute **dans une cage** (la sandbox), et renvoie au LLM ce qui s'est passé. Les « outils » (lire un fichier, lancer des tests) sont des **fonctions Python** dans la cage, qui en réalité appellent un **serveur MCP** dehors. Quand le LLM pense avoir fini, son code appelle `final_answer(...)`.

## 2. Les 4 morceaux et qui les fait
| Morceau | Question à laquelle il répond | Qui |
|---|---|---|
| **LLM + boucle** | *Quoi* exécuter ensuite ? | 🔵 Thomas |
| **Sandbox** | *Comment* l'exécuter sans danger ? | 🟢 Sylvain |
| **Serveurs MCP** | *Avec quels* outils agir sur le monde ? | 🟢 Sylvain |
| **Agents MBPP / SWE** | *Sur quelle* tâche, avec quelles limites ? | 🔵 Thomas |

## 3. Les 5 choses à vraiment comprendre
1. **Pourquoi un process enfant pour la sandbox.** Seul moyen propre de *tuer* un `while True` (timeout) et de limiter la mémoire sans tuer l'agent. Le prix : il faut un **Pipe** pour parler, et l'enfant perd ses variables quand on le tue → on le dit au LLM. → [[MOD - Sandbox exécuteur]]
2. **Pourquoi le client MCP reste dans le parent.** L'enfant n'a ainsi besoin ni de réseau ni de subprocess : on peut **tout** lui interdire. Ses outils sont des « télécommandes » qui passent par le Pipe. → [[FLOW - Appel outil MCP]]
3. **Les budgets sont cumulés et l'historique est renvoyé à chaque tour.** MBPP (6 000 entrée) se gagne en 1–2 tours : prompt minuscule, tester **et** soumettre dans le même bloc. SWE-bench se perd sur les 10 000 tokens de **sortie** : réponses courtes, modèle non-reasoning. → [[MOD - Boucle agent]]
4. **Le LLM ne doit jamais deviner.** Pas de code, code bricolé, timeout, troncature, erreur de syntaxe après un edit : chaque cas a un message explicite. C'est exigé, et c'est ce qui évite les itérations gâchées.
5. **Les serveurs MCP marchent en local ET dans Docker** grâce à un seul helper `_sh` : la moulinette les teste seuls avec `TESTBED_PATH`, notre agent les branche sur le conteneur avec `SWE_CONTAINER`. → [[MOD - MCP SWE-bench]]

## 4. Les pièges qui font échouer (check-list)
- [ ] Une clé API dans le code → échec sécurité.
- [ ] Un seul test sandbox raté → échec (il faut **tous**).
- [ ] Un crash de l'agent → tâche ratée : toujours écrire le JSON.
- [ ] Dépasser une limite **même si la solution est juste** → tâche ratée.
- [ ] Conteneur Docker non supprimé.
- [ ] Le LLM qui va chercher la solution (PR, issue, internet) → **0**. Le prompt l'interdit, la sandbox n'a pas de réseau.
- [ ] Ne pas savoir où changer `max_iterations` en soutenance.
- [ ] Utiliser smolagents / langgraph / RestrictedPython.

## 5. Ce qui est volontairement laissé de côté (strict minimum)
Fallback automatique entre providers · multi-agents · outils en plus des obligatoires · `find_references` intelligent (jedi) · automatisation complète des métriques du benchmark · optimisation fine des tokens au-delà des limites.

## 6. Ordre de travail en une ligne
Fondations ensemble → MBPP de bout en bout → sécurité + MBPP fiable → SWE-bench → benchmark + README. Détail et répartition : [[10 - School/10 - Agent-Smith/Tickets|Tickets]].

## 7. Points encore ouverts ⚠️ (à trancher ensemble)
| Question | Où |
|---|---|
| Comment la moulinette fournit l'eval script à `run_tests()` en test isolé | [[MOD - MCP SWE-bench]] |
| `request_time_ms` : avec ou sans les retries | [[MOD - Client LLM]] |
| `test_imports` MBPP : dans le code ou dans les tests | [[MOD - MCP MBPP]] |
| Les 5 modèles du benchmark | [[GUIDE - Benchmark report]] |
| L'image Docker est-elle pré-téléchargée par la moulinette | [[MOD - Agent SWE-bench]] |
| Login 42 de Thomas pour la 1re ligne du README | [[GUIDE - README et rendu]] |
