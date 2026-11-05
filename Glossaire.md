---
type: overview
title: Glossaire — Agent Smith
status: draft
tags: [agent-smith, glossaire]
---
# Glossaire

| Terme | Sens dans ce projet |
|---|---|
| **Code agent** | Agent dont l'« action » est du code Python exécuté, pas un JSON d'appel d'outil |
| **Thought / Code / Observation** | Les 3 slots d'un tour : raisonnement, code, résultat renvoyé au LLM |
| **Itération / step** | Un appel LLM + une exécution sandbox = une `StepMetrics` |
| **Sandbox** | Process enfant qui exécute le code LLM avec restrictions ([[MOD - Sandbox exécuteur]]) |
| **Namespace** | Le dict de variables du code LLM ; persiste entre les tours |
| **`final_answer`** | Fonction **de la sandbox** (pas MCP) qui termine la tâche |
| **MCP** | *Model Context Protocol* : protocole JSON-RPC entre un client et un serveur d'outils |
| **Tool / Resource / Prompt** | Les 3 choses qu'un serveur MCP expose : fonctions, données lisibles, gabarits de prompts |
| **stdio / streamable HTTP** | Les 2 transports MCP exigés : le serveur est un sous-process (stdin/stdout) ou une URL |
| **Manuel sandbox** | Texte généré depuis les schémas MCP, inséré dans le prompt |
| **MBPP** | *Mostly Basic Python Problems* : 1 fonction + des `assert` |
| **SWE-bench Verified** | Issues GitHub réelles ; on produit un **patch** validé par l'`eval_script` |
| **`/testbed`** | Racine du repo dans le conteneur SWE-bench (`TESTBED_PATH`) |
| **Patch** | Sortie de `git -c core.fileMode=false diff` |
| **Moulinette** | L'outil d'évaluation (`moulinette_eval dump/validate`, scripts `exam_*.sh`) |
| **Free tier** | Quota gratuit d'un provider ; seul autorisé |
| **Rotation de clés** | Passer à la clé API suivante sur rate limit (429) |
| **Stop sequence** | `<end_code>` : coupe la génération pour éviter une fausse observation |
| **Ablation** | Comparer avant/après **un seul** changement, même modèle, mêmes tâches |
| **Hook d'audit** | `sys.addaudithook` : fonction appelée par Python à chaque action sensible (open, socket…) |
