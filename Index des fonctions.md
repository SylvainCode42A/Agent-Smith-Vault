---
type: overview
title: Index des fonctions — Agent Smith
status: draft
tags: [agent-smith, index]
---
# Index des fonctions

Toutes les fonctions à créer, avec leur module. ⚠️ noms proposés, à aligner sur le code.

| Fonction | Module | Qui | Flux |
|---|---|---|---|
| `StepMetrics`, `SolutionOutput`, `SandboxConfig`, `MBPPTaskInput`, `SWEBenchTaskInput` | [[MOD - Modèles Pydantic]] | 🟣 | tous |
| `load_config`, `write_solution` | [[MOD - Modèles Pydantic]] | 🟣 | tous |
| `run_agent`, `Limits`, `format_observation`, `budget_left` | [[MOD - Boucle agent]] | 🔵 | [[FLOW - Tâche MBPP]], [[FLOW - Tâche SWE-bench]] |
| `extract_code`, `_from_python_block`, `_from_xml_invoke`, `_from_hermes`, `_from_react`, `_call_to_python` | [[MOD - Extraction de code]] | 🔵 | idem |
| `build_system_prompt`, `build_user_prompt_mbpp`, `build_user_prompt_swe` | [[MOD - Prompt système]] | 🔵 | idem |
| `LLMClient.chat`, `LLMResponse`, `KeyRing`, `provider_prefix` | [[MOD - Client LLM]] | 🔵 | idem |
| `Sandbox.start/execute/manual/close`, `_worker`, `_make_tool_wrapper`, `_PipeStdout`, `FinalAnswer`, `truncate` | [[MOD - Sandbox exécuteur]] | 🟢 | [[FLOW - Appel outil MCP]] |
| `is_import_allowed`, `guarded_import`, `safe_open`, `is_path_allowed`, `safe_builtins`, `check_ast`, `apply_limits`, `install_guards`, `preimport` | [[MOD - Sandbox sécurité]] | 🟢 | — |
| `McpClient.from_stdio/from_http/connect/call_tool/read_resource/get_prompt/close` | [[MOD - Client MCP]] | 🟢 | [[FLOW - Appel outil MCP]] |
| `build_manual`, `tool_signature` | [[MOD - Manuel sandbox]] | 🟢 | — |
| `main`, `parse_args`, `read_entry`, `repl` (CLI) | [[MOD - CLI sandbox]] | 🟢 | — |
| `run_tests(code, test_list)` | [[MOD - MCP MBPP]] | 🟢 | [[FLOW - Tâche MBPP]] |
| `read_file`, `edit_file`, `list_files`, `search_code`, `search_function_or_class_definition_in_code`, `find_references`, `run_tests`, `get_patch`, `run_command`, `_sh` | [[MOD - MCP SWE-bench]] | 🟢 | [[FLOW - Tâche SWE-bench]] |
| `main`, `load_task`, `mbpp_fallback` | [[MOD - Agent MBPP]] | 🔵 | [[FLOW - Tâche MBPP]] |
| `main`, `start_container`, `copy_eval_script`, `remove_container`, `on_sigterm`, `swe_fallback` | [[MOD - Agent SWE-bench]] | 🔵 | [[FLOW - Tâche SWE-bench]] |
