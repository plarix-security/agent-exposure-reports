# WYSCAN v1.6.2 | SCAN DATE: 2026-04-12/13 | AFB04 Taxonomy v1.6
# Full Index: 10 OSS Agentic Systems

---

## 1. AgentGPT
**Repo:** reworkd/AgentGPT | **Framework:** langchain | **Files scanned:** 234 | **Duration:** 2.7s

**CEE breakdown:** 29 total | 20 classified | 9 unclassified | CRITICAL: 0 | WARNING: 10 | INFO: 10 | UNCLASSIFIED: 9

**AFB04 Governance: UNGOVERNED** (100% of classified findings, no policy gate detected)

**Critical findings:** None.

**Warning findings:**
- NETWORK (2): Both trace from `call` tool (LangChain) to `aiohttp.ClientSession.post`. Finding at `sidsearch.py:28` has tool-controlled input flowing directly into an authenticated POST to `https://api.sid.ai/v1/users/me/query` (user's private knowledge base). Second at `sidsearch.py:47` reaches the Sid.ai OAuth token endpoint. Neither path has a policy gate.
- STATE_MUTATION (8): `call` tool reaches `answer_values.append`, `snippets.append`, `texts.append` in `search.py` via Serper/Google search path. Search result content flows into answer buffers without sanitization -- prompt injection via poisoned search results reaches LLM context.

**Info findings (10):** MEMORY_READ on `answerBox`, `snippet`, `snippetHighlighted`, `results` fields in `search.py` and `sidsearch.py`.

**Unclassified CEEs (9):** All LOW_RISK_UNTRACED. 2 are UI event handlers (React). 7 are `call` registrations in `code.py`, `conclude.py`, `image.py`, `reason.py`, `search.py:55`, `sidsearch.py:130`, `wikipedia_search.py` -- no resolved execution paths.

**Coverage gap:** 563 unresolved call edges (2.4 per file). Only 2 entrypoints successfully traced to sensitive ops. True exposure likely larger.

**Key points:**
1. All 20 classified findings UNGOVERNED.
2. Tool-controlled input -> `aiohttp.ClientSession.post` -> Sid.ai private knowledge API (`sidsearch.py:28`). Direct prompt injection to private knowledge store.
3. OAuth token endpoint reachable from same ungoverned `call` tool.
4. 8 WARNING state-mutation findings: Serper results into answer buffers without sanitization.
5. 2 unique traced entrypoints vs 234 files -- floor, not ceiling.
6. 9 unclassified `call` registrations in `code.py`, `image.py`, `reason.py`, `wikipedia_search.py` not resolved; may perform sensitive ops.

---

## 2. Agno (formerly Phidata)
**Repo:** agno-agi/agno | **Framework:** agno, pydantic-ai | **Files scanned:** 2,719 | **Duration:** 136.8s

**CEE breakdown:** 239 total | 152 classified | 87 unclassified | CRITICAL: 2 | WARNING: 90 | INFO: 60 | UNCLASSIFIED: 87

**AFB04 Governance: UNGOVERNED** (100% of classified findings, no policy gate detected)

**Critical findings:**
1. `execute_shell_command` -> `subprocess.check_output(command, shell=True)` at `external_tool_execution.py:29`. Tool-controlled `command` string. `shell=True` = arbitrary shell injection. File: `external_tool_execution.py:18` entry point.
2. `run_shell_command` -> `subprocess.check_output(shlex.split(command))` at `external_tool_execution_stream.py:38`. Tool-controlled input. `shlex.split` removes metacharacter injection but agent still executes arbitrary commands with no policy gate. Team-variant of the same PROCESS_EXECUTION pattern.

**Warning findings:**
- MEMORY_WRITE (2): `save_learning` (`human_in_the_loop.py:94`) and `save_validated_query` (`save_query.py:73`, tool-controlled `payload`) reach `.insert()` into persistent knowledge stores. Poisoning via compromised agent instruction.
- STATE_MUTATION (12+): `list_sources`, `get_metadata`, `list_buckets`, `list_files`, `read_file`, `search_files` (in `awareness.py`, `s3.py`), `add_to_watchlist` (`confirmation_with_session_state.py:37`, tool-controlled `symbol`), `add_item` (`retry_tool_call_from_post_hook.py:49`, tool-controlled `item`), `get_session_runs` (pydantic-ai `mcp.py`). All ungoverned appends to in-memory state.
- FILESYSTEM (1): `write_file` tool -> `s3.py:192`, tool-controlled `parent` argument. Arbitrary write to accessible paths.
- NETWORK (29): `get_top_hackernews_stories` across 14+ cookbook files (`confirmation_advanced.py`, `confirmation_required.py`, `approval_async.py`, `approval_basic.py`, `human_in_the_loop.py`, `async_tool_decorator.py`, `cache_tool_calls.py`, `tool_decorator_with_hook.py`, `tool_decorator_with_instructions.py`, `tool_decorator.py`, `pre_and_post_hooks.py`). `get_current_weather` -> geocoding + forecast APIs. `save_intent_discovery` (`save_discovery.py:61`, tool-controlled `payload`). All via `httpx.get` / `httpx.AsyncClient.get`, no policy gates.

**Info findings (60):** MEMORY_READ and IDENTITY ops across `awareness.py`, `s3.py`, `tool_choice.py`, `tool_call_limit.py`, `tool_use.py`, `mcp.py`, `async_tool_decorator.py`, `tool_decorator.py`, `pre_and_post_hooks.py`.

**Unclassified CEEs (87):**
- HIGH_RISK_UNTRACED (26): `send_email`, `deploy_to_production`, `delete_user_data`, `send_bulk_email`, `run_security_scan`, `critical_action`, `approve_deployment`, `send_money`, `delete_user_account`, `book_flight`, `process_payment`, `create_user`, `submit_order`, `plan_delivery`, `transfer_funds`. Execution paths not resolved; authorization status unknown.
- MEDIUM_RISK_UNTRACED (11): `write_file`, `read_file`, `search_files`, `create_session`, `rename_session`, `delete_session`, `delete_sessions`, `create_memory`, `update_memory`, `delete_memory`, `delete_memories` (AgnoOS MCP server).
- LOW_RISK_UNTRACED (50): UI helpers, computation tools, read-only lookups, framework internals, interop stubs.

**Coverage gap:** N/A (not reported). 87 unclassified CEEs, actual reachability likely substantially higher than 152 classified findings.

**Key points:**
1. 2 CRITICAL: both `subprocess.check_output` with tool-controlled `command`, no gate. Highest-severity AFB04 pattern -- arbitrary OS command injection.
2. 100% of 152 classified findings ungoverned. Framework design defers authorization entirely to app code; cookbook examples universally omit it.
3. 26 HIGH_RISK_UNTRACED tools including `transfer_funds`, `send_money`, `deploy_to_production`, `delete_user_data`, `send_bulk_email` -- execution paths unresolved, authorization unknown.
4. Dominant warning pattern: unguarded state mutation and in-memory appends across S3, awareness, and session tools.
5. Network ops via httpx reachable from 12+ distinct registrations, no request-level authorization.
6. 11 MEDIUM_RISK_UNTRACED session/memory mgmt tools in AgnoOS MCP: `create_memory`, `delete_session`, etc. Persistent state attack surface not fully resolved.

---

## 3. AutoGPT
**Repo:** Significant-Gravitas/AutoGPT | **Framework:** langchain, export-entry | **Files scanned:** 1,735 | **Duration:** 95.5s

**CEE breakdown:** 556 total | 367 classified | 189 unclassified | CRITICAL: 3 | WARNING: 92 | INFO: 272 | UNCLASSIFIED: 189

**AFB04 Governance: UNGOVERNED** (100% of classified findings, no policy gate detected)

**Critical findings:**
1. `_execute` -> `asyncio.create_subprocess_exec` at `agent_browser.py:87`. Entry: `agent_browser.py:449`. Framework: langchain. Tool-controlled input: No. Resource hint: `cmd`. 3 intermediate calls. Browser-layer subprocess spawning, no gate.
2. `_execute` -> `os.unlink(tmp_path)` at `agent_browser.py:833`. Entry: `agent_browser.py:784`. FILESYSTEM. Resource hint: `tmp_path`. Arbitrary file deletion from same browser tool entry point as CRITICAL #1.
3. `_execute` -> `asyncio.create_subprocess_exec` at `sandbox.py:243`. Entry: `bash_exec.py:78`. Framework: langchain. Tool-controlled input: **Yes**. Resource hint: `full_command`. **Most severe:** tool-controlled input traced directly into subprocess exec arguments, single intermediate call. Direct command injection primitive: any instruction influencing `full_command` = arbitrary code execution.

**Warning findings:**
- STATE_MUTATION (83): `_execute` reaches `.append()` and `.set()` on agent lists, node lists, link lists, output buffers, missing-field lists, credential lists, execution task holders -- across `create_agent.py`, `customize_agent.py`, `edit_agent.py`, `find_agent.py`, `find_library_agent.py`, `run_block.py`, `continue_run_block.py`, `agent_output.py`, `run_agent.py`. ~40 with instruction-influenced input, 8 with tool-controlled input. LLM output directly shapes in-memory agent state.
- FILESYSTEM (5): `_execute` -> `os.makedirs` (`sandbox.py`, `workspace_files.py`), `sandbox.files.write`, `builtins.open` (write mode). 4 of 5 carry tool-controlled or instruction-influenced input. `workspace_files.py` cluster most significant: tool-controlled input to both sandbox remote path and local validated path. Direct arbitrary file write.
- NETWORK (2): `aiohttp.ClientSession.get` -> `https://api.github.com/user` (GitHub identity lookup, from `bash_exec.py:78`). `fetch` in TypeScript direct-upload path (`direct-upload.ts:36`, export-entry). No policy gates.
- PROCESS_EXECUTION (2): `.append()` calls building subprocess command argument arrays (`cmd_args`) in `agent_browser.py:816`, `agent_browser.py:817`. Feed directly into CRITICAL #1 subprocess call. No gate.

**Info findings (272):** MEMORY_READ, IDENTITY, FILESYSTEM ops across `__init__.py`, `helpers.py`, `context.py`, `integration_creds.py`, `file_ref.py`, `agent_browser.py`, `core.py`, `pipeline.py`, `run_agent.py`, `workspace_files.py`, `simulator.py`, `file_content_parser.py`, `json.py`, `type.py`, etc.

**Unclassified CEEs (189):** All LOW_RISK_UNTRACED. 37+ backend tool registrations (`add_understanding.py`, `agent_browser.py`, `bash_exec.py`, `connect_integration.py`, `create_agent.py`, `customize_agent.py`, `edit_agent.py`, `feature_requests.py`, `find_agent.py`, `find_block.py`, `find_library_agent.py`, `fix_agent.py`, `get_agent_building_guide.py`, `get_doc_page.py`, `get_mcp_guide.py`, `manage_folders.py`, `run_agent.py`, `run_block.py`, `run_mcp_tool.py`, `search_docs.py`, `validate_agent.py`, `web_fetch.py`, `workspace_files.py`). Also frontend React/TS components, hooks, Google Analytics gtag.js registrations, `forge_agent.py:211 execute`.

**Coverage gap:** 7,432 unresolved call edges (4.3 per file). 123 unique entrypoints traced to sensitive ops. High ratio indicates dynamic dispatch, plugin-style block registration, runtime-resolved imports. 7,432 unresolved edges = likely many additional ungoverned paths beyond confirmed findings.

**Key points:**
1. Direct command injection confirmed: `bash_exec.py` -> `sandbox.py` (CRITICAL #3), tool-controlled input -> `asyncio.create_subprocess_exec`, 1 intermediate call, no gate.
2. Zero authorization coverage across all 367 classified findings. Every op -- subprocess, file write, file read, network, identity lookup -- equally accessible without access control.
3. Agent graph manipulation fully LLM-controlled: `create_agent.py`, `customize_agent.py`, `edit_agent.py` allow LLM to mutate agent graph state (nodes, links, metadata, IDs) without authorization.
4. Filesystem write paths (`workspace_files.py:163-181`): tool-controlled input -> `sandbox.files.write` and `open(validated, "wb")` with `os.makedirs`. Direct arbitrary file write.
5. GitHub user identity fields (name, email, login, ID) reachable via `aiohttp.ClientSession.get` from `bash_exec.py:78` entry point, no gate.
6. 7,432-edge gap substantially underestimates actual exposure. 189 unclassified CEEs include 37+ backend registrations -- resolved entries already produce 367 findings; unresolved likely comparable.

---

## 4. Eliza (elizaOS/eliza)
**Repo:** elizaOS/eliza | **Framework:** export-entry, action-object, runtime-composition, service-class | **Files scanned:** 513 | **Duration:** 39.4s

**CEE breakdown:** 198 total | 100 classified | 98 unclassified | CRITICAL: 2 | WARNING: 78 | INFO: 20 | UNCLASSIFIED: 98

**AFB04 Governance: UNGOVERNED** (100% of classified findings, no policy gate detected)

**Critical findings:**
1. `createRuntimes` (export-entry, `runtime-composition.ts:308`) -> `Bun.spawn` at `plugin.ts:45:17`. 5 intermediate calls, no gate. Unresolved edges: `<unknown>`, `pluginNames.add`, `pluginInput.push`. Plugin-controlled data feeds into execution chain. Social-media-ingested character definitions load plugins dynamically -> process spawning.
2. `createRuntimes` (same entry) -> `Bun.spawn` at `plugin.ts:69:18`. Second distinct call site, same 5-call chain, same unresolved edges. Two ungoverned process execution paths from single plugin-loading entry point.

**Warning findings:**
- NETWORK (5+): `loadPlugin` -> `fetch` (index.ts:130:27, remote plugin manifests). `loadWasmPlugin` -> `fetch` (wasm-loader.ts:156:27, WASM binaries). `validateWasmPlugin` -> `fetch` (wasm-loader.ts:156:27). `loadWasmPlugin` -> `fetch` (wasm-loader.ts:276:27). `processAttachments` -> `fetch` (index.ts:230:22, index.ts:345:21, arbitrary URLs from message content). Bidirectional network access surface, no auth boundary.
- MEMORY_WRITE -- `runtime.createMemory` (20): Reachable from `_shouldFollow` (`followRoom.ts:105`, `followRoom.ts:129`), `followRoomAction.handler` (`followRoom.ts:105`, `followRoom.ts:129`, `followRoom.ts:178`), `_shouldMute` (`muteRoom.ts:90`, `muteRoom.ts:113`), `muteRoomAction.handler` (`muteRoom.ts:90`, `muteRoom.ts:113`, `muteRoom.ts:161`), `unfollowRoomAction.handler` (`unfollowRoom.ts:86`, `unfollowRoom.ts:151`), `_shouldUnmute` (`unmuteRoom.ts:68`, `unmuteRoom.ts:93`), `unmuteRoomAction.handler` (`unmuteRoom.ts:68`, `unmuteRoom.ts:93`, `unmuteRoom.ts:177`), `handler (reflection)` (`reflection.ts:573`), `sendToAdminAction.handler` (`action.ts:206`), `runAutonomyPostResponse` (`execution-facade.ts:117`), `worker.execute` (`followUp.ts:398`). All ungoverned. Social media posts -> persistent memory poisoning.
- MEMORY_WRITE/STATE -- `runtime.useModel` (30+): Every action handler across advanced-capabilities, advanced-memory, basic-capabilities, autonomy packages. Includes `addContactAction.handler`, `createTaskAction.handler`, `followRoomAction.handler`, `generateImageAction.handler`, `muteRoomAction.handler`, `removeContactAction.handler`, `updateRoleAction.handler`, `scheduleFollowUpAction.handler`, `searchContactsAction.handler`, `sendMessageAction.handler`, `updateSettingsAction.handler`, `thinkAction.handler`, `unfollowRoomAction.handler`, `unmuteRoomAction.handler`, `updateContactAction.handler`, `updateEntityAction.handler`, `handler (reflection evaluator)`, `longTermExtractionEvaluator.handler`, `handler (advanced-memory reflection)`, `summarizationEvaluator.handler`, `choiceAction.handler`, `replyAction.handler`, `processAttachments`, `generateTextEmbedding`, `trimTokens`. All ungoverned: instruction injection from social media channels -> model inference across all action types, no gate.
- FILESYSTEM (15): `fs.mkdirSync` via `logger.ts:407` reachable from 8 distinct entry points (`createTaskAction.handler`, `processKnowledgeAction.handler`, `searchKnowledgeAction.handler`, `generateText`, `generateTextEmbedding`, `start (knowledge service)`, `createRuntimes`, `worker.execute`). Cache/registry deletions in `utils.ts:1218` (from `createTaskAction.handler`, `sendToAdminAction.handler`, `runAutonomyPostResponse`, `processKnowledgeAction.handler`, `loadCharacters`, `createRuntimes`, `worker.execute`). `plugin.ts:313` and `task-scheduler.ts:101` also.

**Info findings (20):** `fs.readFile`, `fs.stat`, `fs.readFileSync`, `fs.existsSync`, `fs.readdir`, `readFile` across `index.ts`, `wasm-loader.ts`, `actions.ts`, `runtime-composition.ts`, `pairing-migration.ts`.

**Unclassified CEEs (98):** All LOW_RISK_UNTRACED. Framework lifecycle hooks, event handlers (ACTION_STARTED, ACTION_COMPLETED, EVALUATOR_STARTED, RUN_STARTED, RUN_ENDED, RUN_TIMEOUT, CONTROL_MESSAGE, WORLD_JOINED, MESSAGE_SENT), service start methods (`approval.ts`, `followUp.ts`, `hook.ts`, `embedding.ts`, `pairing.ts`, `relationships.ts`, `task.ts`, `tool-policy.ts`, `triggerWorker.ts`), utility functions, infrastructure functions, plugin-loading helpers, Discord/Telegram event handler registrations.

**Coverage gap:** 10,399 unresolved call edges (largest absolute count). 140 unique entrypoints reaching sensitive ops. 1 file failed analysis. `<unknown>` edges in both CRITICAL paths.

**Key points:**
1. Both CRITICALs: same `createRuntimes` entry point -> `Bun.spawn`, plugin loading system provides ungoverned path to arbitrary process execution.
2. 100% of 100 classified findings ungoverned. Framework fully lacks policy layer for sensitive op authorization.
3. 30 `runtime.useModel` WARNINGs across full breadth of action types: role mgmt, settings mutation, contact ops, autonomous follow-up scheduling -- all reachable from social media content.
4. 20 `runtime.createMemory` WARNINGs: content-driven memory poisoning across follow/unfollow, mute/unmute, autonomy, follow-up paths. Persistent state corruption from external social input.
5. `fs.mkdirSync` via `logger.ts:407` reached from 8 entry points through cross-file chains -- filesystem write as secondary consequence of logging, not intentional design, unlikely to be audited.
6. 10,399 unresolved edges + 98 unclassified CEEs + dynamic plugin system: true scope dramatically underreported.

---

## 5. GPT Pilot
**Repo:** Pythagora-io/gpt-pilot | **Framework:** class-based-tool | **Files scanned:** 302 | **Duration:** 12.9s

**CEE breakdown:** 232 total | 209 classified | 23 unclassified | CRITICAL: 0 | WARNING: 88 | INFO: 121 | UNCLASSIFIED: 23

**AFB04 Governance: UNGOVERNED** (100% of classified findings, no policy gate detected)

**Critical findings:** None.

**Warning findings:**
- NETWORK (14): `run` tool -> `httpx.Client.get`, `client.post`, `httpx.AsyncClient.post`. Entry points: `external_docs.py:76`, `external_docs.py:143`, `frontend.py:248`, `frontend.py:470`, `wizard.py:142`, `__init__.py:387` (x9 -- all different agents, same POST destination `self.endpoint`). Instruction-influenced input present in majority. `self.endpoint` resource hint on the 9 `__init__.py:387` findings: POST destination URL itself is reachable from agent context -> SSRF if adversarial instruction supplies crafted endpoint.
- COMMUNICATION / STATE_MUTATION (74): Dominant pattern: `self.send_message` and `self.ui.send_message` from every agent's `run` entrypoint, no gate. Affected agents: `architect.py`, `bug_hunter.py`, `developer.py`, `executor.py`, `frontend.py`, `human_input.py`, `importer.py`, `orchestrator.py`, `tech_lead.py`, `tech_writer.py`, `troubleshooter.py`, `wizard.py`. Also `.append()` calls on `async_tasks`, `unique_steps`, `epic_tasks`, `tasks`, `parallel`, `input_required_files`, `file_paths_to_remove_mock`. LLM-generated strings + instruction-influenced content -> UI channel, no gate. Notable: telemetry calls (`telemetry.set`) in `architect.py` and `orchestrator.py` receive architecture specs, template names, file metrics from `run` entrypoint with no policy check.

**Info findings (121):** MEMORY_READ and FILESYSTEM across all agent files: `bug_hunter.py`, `code_monkey.py`, `developer.py`, `error_handler.py`, `executor.py`, `external_docs.py`, `frontend.py`, `human_input.py`, `orchestrator.py`, `task_completer.py`, `tech_lead.py`, `troubleshooter.py`, `wizard.py`, `helpers.py`, `api_server.py`, `text.py`. FILESYSTEM ops in `orchestrator.py` (package_json_path, node_modules_path, absolute_path, index_path, template_path), `frontend.py` (absolute_path). MEMORY_READ on `description`, `user_feedback`, `redo_human_instructions`, `source`, `status`, `command`, `chatHistory`, `scripts`, `related_api_endpoints`, `files`, etc.

**Unclassified CEEs (23):** All LOW_RISK_UNTRACED. 18 Python agent `run` registrations: `architect.py:99`, `bug_hunter.py:61`, `code_monkey.py:72`, `developer.py:95`, `error_handler.py:24`, `executor.py:72`, `external_docs.py:46`, `frontend.py:44`, `human_input.py:10`, `importer.py:21`, `legacy_handler.py:9`, `orchestrator.py:48`, `problem_solver.py:38`, `task_completer.py:15`, `tech_lead.py:59`, `tech_writer.py:16`, `troubleshooter.py:39`, `wizard.py:24`. Plus `sidebar.tsx:100` (handleKeyDown x2), `useMobile.tsx:10` (onChange x2), `api_server.py:335`.

**Coverage gap:** 379 unresolved call edges. Only 4 unique entrypoints traced to sensitive ops despite 18 Python agent classes. 23 unclassified registrations = potential additional entrypoints to already-flagged operations.

**Key points:**
1. Zero CRITICAL findings but 88 WARNINGs, all single framework pattern (`class-based-tool` / `run` method), no policy gate anywhere. 100% ungoverned.
2. Primary risk: unbounded outbound HTTP. 14 network-reaching WARNINGs with SSRF potential via `self.endpoint` reachable from 9 agent entry points.
3. Instruction-influenced data flows into UI message channels across every agent -> prompt injection surface: adversarial LLM response content propagates unfiltered to developer interface.
4. Telemetry calls (`telemetry.set`) in `architect.py` and `orchestrator.py` receive architecture specs + file metrics, no gate -> data exfiltration path if endpoint is attacker-controlled.
5. 4 traced entrypoints vs 18 agent classes + 379 unresolved edges: substantial coverage gap.
6. No process execution findings (no `subprocess`, `os.system`, `exec`) in classified set. GPT Pilot's shell execution capability operates through a different unresolved code path -> unquantified risk outside confirmed findings.

---

## 6. GPT Researcher
**Repo:** assafelovic/gpt-researcher | **Framework:** langchain | **Files scanned:** 278 | **Duration:** 3.9s

**CEE breakdown:** 40 total | 3 classified | 37 unclassified | CRITICAL: 0 | WARNING: 0 | INFO: 3 | UNCLASSIFIED: 37

**AFB04 Governance: UNGOVERNED** (100% of classified findings, no policy gate detected)

**Critical findings:** None.

**Warning findings:** None.

**Info findings (3):** `search_tool` (langchain) -> MEMORY_READ on `title` (`tools.py:216`), `content` (`tools.py:217`), `url` (`tools.py:218`). External search results read without policy gate. Primary prompt injection surface: externally sourced content flows unfiltered into agent context.

**Unclassified CEEs (37):**
- MEDIUM_RISK_UNTRACED (1): `sendChatMessage` (`scripts.js:1893`). If this transmits research output/user input to external endpoint -> ungoverned data exfiltration.
- LOW_RISK_UNTRACED (36): UI event handlers (`Home`, `checkIfMobile`, `handleScroll`, `handleKeyDown`, `handleResizeMove`, `handleResizeEnd`, `handleClickOutside`, `startLanggraphResearch`), workbox service worker internals, frontend utilities (`initHistoryPanel`, `initWebSocketPanel`, `clearConversationHistory`, `filterHistoryEntries`, `copyToClipboard`, `showImageDialog`, `initChat`, `initSpeechRecognition`, `initExpandButtons`, `initMCPSection`, `validateMCPConfig`, `formatMCPConfig`, `showMCPInfo`, `createMCPInfoModal`, `exportHistory`, `triggerImportHistory`, `handleFileImport`), and `custom_tool` (`tools.py:260`) -- user-defined tool registration point, arbitrary external ops registerable without trace.

**Coverage gap:** 2,322 unresolved call edges -- **highest density of any system evaluated** (8.4 per file). 34 unique entrypoints traced to sensitive ops (highest unique entrypoint count).

**Key points:**
1. Zero CRITICAL/WARNING findings, but this is a coverage artifact, not system safety: 2,322 unresolved edges means majority of execution paths (web scraping, content processing pipeline) were not traversable by static analysis.
2. 3 INFO findings confirm `search_tool` reads title/content/url from external search results with no gate (`tools.py:216-218`). Primary prompt injection surface confirmed.
3. 34 unique entrypoints traced to sensitive ops -- highest in evaluation set -- indicates wide, deeply connected tool surface. Warrants dynamic/manual analysis.
4. `custom_tool` at `tools.py:260` = unclassified CEE. User-defined tool registration: arbitrary external ops can be registered without Wyscan tracing their execution path.
5. `sendChatMessage` at `scripts.js:1893` = MEDIUM_RISK_UNTRACED. Potential ungoverned data exfiltration path.
6. Highest unknown attack surface of all 10 systems. Dynamic analysis strongly recommended.

---

## 7. MetaGPT
**Repo:** geekan/MetaGPT | **Framework:** class-based-tool | **Files scanned:** 660 | **Duration:** 6.8s

**CEE breakdown:** 455 total | 376 classified | 79 unclassified | CRITICAL: 10 | WARNING: 191 | INFO: 175 | UNCLASSIFIED: 79

**AFB04 Governance: UNGOVERNED** (100% of classified findings, no policy gate detected)

**Critical findings (10 total -- highest CRITICAL count of any system evaluated):**

1. `run` -> `subprocess.run(["python3", "-c", code_text], ...)` at `build_customized_agent.py:48`. Tool-controlled `code_text` passed verbatim as shell command argument. No sandboxing, no allowlist, no gate. Arbitrary code execution on host.
2. `run` -> `shutil.rmtree(path)` at `prepare_documents.py:48`. Entry: `prepare_documents.py:53`. Instruction-influenced `path`. Runs at project initialization -> crafted project path = irreversible directory tree deletion before code generation. No gate.
3. `run` -> `exec(code, namespace)` at `run_code.py:87`. Entry: `run_code.py:120`. Instruction-influenced `code`. Full Python code execution: `exec` evaluates any Python expression including imports, subprocess calls, file ops. Equivalent to arbitrary code execution.
4. `run` -> `subprocess.Popen` at `run_code.py:106`. Entry: `run_code.py:120`. Instruction-influenced `command`. Same entry point as CRITICAL #3 -- two independent process execution vectors from one ungoverned tool registration.
5. `run` -> `subprocess.run(cmd, check=check, cwd=cwd, env=env)` at `run_code.py:151`. Entry: `run_code.py:120`. Instruction-influenced `cmd`. Third process execution path from same entry. Controllable `cwd` and `env` -> attacker could modify `PYTHONPATH`/`PATH` to redirect to attacker-controlled binaries.
6. `run` -> `os.system(check_command)` at `common.py:63`. Entry: `design_api.py:64`. Instruction-influenced `check_command`, 5 intermediate calls crossing file boundaries. `os.system` = full shell, metacharacters = command injection. Shared infrastructure -> reachable from multiple agent roles.
7. `run` -> `os.system(check_command)` at `common.py:63`. Entry: `write_prd.py:86`. Same call site, second entry point. Write_prd = earliest pipeline stage (PRD writing). Crafted requirement spec triggers shell execution before code generation.
8. `run` -> `asyncio.create_subprocess_shell(commands)` at `mermaid.py:83`. Entry: `design_api.py:64`. Instruction-influenced `commands`. Shell interpreter receives commands directly. Async shell injection during architecture diagram generation.
9. `run` -> `asyncio.create_subprocess_shell(commands)` at `mermaid.py:83`. Entry: `write_prd.py:86`. Same async shell execution call, PRD writer entry point. Combined with CRITICAL #7: `write_prd` alone exposes two distinct shell execution paths.
10. `run` -> `subprocess.check_call(["npm", "install", "-g", "@mermaid-..."])` at `setup.py:16`. Entry: `setup.py:14`. Hardcoded argument, no tool-controlled input. But no policy gate -> agent can trigger global npm install without user approval. If package name were made instruction-influenced -> supply-chain compromise.

**Warning findings:**
- MEMORY_WRITE (68): `run` tool -> `self.graph_db.insert` and `self.graph_db.delete` via instruction-influenced paths in `extract_readme.py`, `rebuild_class_view.py`, `rebuild_sequence_view.py`, `graph_repository.py`. Graph DB stores architectural knowledge (class relationships, sequence diagrams, file metadata) consumed by downstream agents. Attacker-controlled insertions poison context for Engineer and QA roles -> multi-hop influence across entire pipeline. `write_code.py` -> `codes.insert`. File lines: `extract_readme.py:46/48/50/52`, `rebuild_class_view.py:103/130/133/141`, `rebuild_sequence_view.py:175/176/179/185/287/308/341/532/538/566/567/570/575`, `graph_repository.py:185/188/192/198/205/210/217/221/225/232/238/275/278/279/280`, `write_code.py:204`.
- FILESYSTEM (72): Write ops reachable from 14+ `run` entry points: `agent_creator.py:53` (tool-controlled `code_text` written verbatim to disk), `code_review.py:222/240` (tool-controlled `comments` written to JSON), `modify_code.py:109` (tool-controlled `patch_file`). Also `design_api.py:232`, `execute_nb_code.py:279`, `rebuild_class_view.py:77/83/91/97`, `invoice_ocr.py:79/148`, `write_prd.py:163/279`, `manual_record.py:51/53/54/55/154/164`, `parse_record.py:45/121/129/130/131`, `screenshot_parse.py:106`, `self_learn_and_reflect.py:72/230`, `common.py:604/608/739/741/1201/1204`, `file.py:45/48`.
- STATE_MUTATION (51): `run` and `execute` -> `.append()` calls in `execute_nb_code.py` (most critical: `execute_nb_code.py:139`, tool-controlled `code` appended as Jupyter notebook cell -- later executed via Jupyter kernel), `rebuild_sequence_view.py`, `invoice_ocr.py`, `prepare_documents.py`, `rebuild_class_view.py`, `research.py:273` (tool-controlled `summary`), `code_review.py`, `cleaner.py`, `retrieve.py`, `utils.py`, `schema.py`, `experience_operation.py`, android extension utils, Stanford Town simulation. Also `common.py:757/769`.

**Info findings (175):** MEMORY_READ (`.get()`, `.keys()`, `.read_text()`, `.readlines()`, `.readline()`) and FILESYSTEM (`.exists()`, `os.walk`, `iterdir()`) across all action files. Full list includes: `debug_error.py`, `design_api.py`, `extract_readme.py`, `import_repo.py`, `invoice_ocr.py`, `prepare_documents.py`, `project_management.py`, `rebuild_class_view.py`, `write_framework.py`, `run_code.py`, `skill_action.py`, `summarize_code.py`, `write_code_review.py`, `write_code.py`, `write_prd.py`, `manual_record.py`, `parse_record.py`, `screenshot_parse.py`, `self_learn_and_reflect.py`, `utils.py`.

**Unclassified CEEs (79):** All LOW_RISK_UNTRACED (function names all `run` or `execute`). 79 action class `run` registrations across full MetaGPT action hierarchy: `build_customized_agent.py`, `dalle_gpt4v_agent.py`, `debate.py`, `analyze_requirements.py`, `debug_error.py`, `design_api_review.py`, `design_api.py`, `ask_review.py`, `execute_nb_code.py`, `write_analysis_code.py` (x2), `write_plan.py`, `execute_task.py`, `extract_readme.py`, `generate_questions.py`, `import_repo.py`, `invoice_ocr.py` (x3), `prepare_documents.py`, `prepare_interview.py`, `project_management.py`, `rebuild_class_view.py`, `rebuild_sequence_view.py`, `write_framework.py`, `pic2txt.py`, `compress_external_interfaces.py` (x2), `detect_interaction.py` (x2), `write_trd.py`, `research.py` (x3), `run_code.py`, `search_and_summarize.py`, `search_enhanced_qa.py`, `skill_action.py` (x2), `summarize_code.py`, `talk_action.py`, `write_code_an_draft.py`, `write_code_plan_and_change_an.py`, `write_code_review.py` (x2), `write_code.py`, `write_docstring.py`, `write_prd_review.py`, `write_prd.py`, `write_review.py`, `write_teaching_plan.py`, `write_tutorial.py` (x2), `manual_record.py`, `parse_record.py`, `screenshot_parse.py`, `self_learn_and_reflect.py`, `code_review.py`, `modify_code.py`, `dummy_action.py`, `st_action.py`, `st_role.py (execute)`, `common_actions.py` (x5), `experience_operation.py` (x2), `moderator_actions.py` (x3), `role.py`, `setup.py`.

**Coverage gap:** 216 unresolved call edges (0.33 per file, moderate). Unique entrypoints: not reported. Cross-file paths in `common.py` and `graph_repository.py` = convergence points for multiple agent roles -- single authorization failure has blast radius across entire multi-agent pipeline.

**Key points:**
1. 10 CRITICAL findings -- highest of all 10 systems. All from ungoverned `run` registrations. Covers: `subprocess.run`, `subprocess.Popen`, `asyncio.create_subprocess_shell`, `os.system`, `builtins.exec`, `subprocess.check_call`, `shutil.rmtree` across 7 distinct call sites.
2. `write_prd.py` (the PRD writer, pipeline's first stage) alone exposes two shell execution paths: `os.system` via `common.py:63` and `asyncio.create_subprocess_shell` via `mermaid.py:83`. Crafted requirement spec = arbitrary shell exec before any code is generated.
3. `run_code.py:120` single entry point exposes 3 independent code execution vectors: `exec(code)`, `subprocess.Popen`, `subprocess.run`. No gate on any.
4. 68 WARNING findings reach `graph_db.insert`/`graph_db.delete` with instruction-influenced content. Graph DB poisons downstream agent context -> multi-hop propagation of malicious content across all roles.
5. Android extension (`manual_record.py`, `parse_record.py`, `screenshot_parse.py`, `self_learn_and_reflect.py`) contributes 30+ WARNING/INFO findings with tool-controlled file paths and `os.walk(unzip_path)`. Developed without same security review as core.
6. All 79 unclassified `run` registrations are functionally identical to entry points that produced CRITICAL findings. True PROCESS_EXECUTION surface substantially larger than 10 CRITICALs indicate.

---

## 8. OpenHands (formerly OpenDevin)
**Repo:** All-Hands-AI/OpenHands | **Framework:** pydantic-ai | **Files scanned:** 1,702 | **Duration:** 42.5s

**CEE breakdown:** 80 total | 15 classified | 65 unclassified | CRITICAL: 0 | WARNING: 5 | INFO: 10 | UNCLASSIFIED: 65

**AFB04 Governance: UNGOVERNED** (100% of classified findings, no policy gate detected)

**Critical findings:** None.

**Warning findings:**
- STATE_MUTATION (5): Five VCS tools -- `create_pr` (GitHub), `create_mr` (GitLab), `create_bitbucket_pr`, `create_bitbucket_data_center_pr`, `create_azure_devops_pr` -- all reach the same call site: `conversation.pr_number.append(pr_number)` at `mcp.py:84`, through 1 intermediate call, no gate. Any agent instruction can append arbitrary PR/MR numbers to conversation state without authorization -> session tracking corruption or confused-deputy attacks across VCS providers.

**Info findings (10):**
- `create_pr` -> IDENTITY: `X-OpenHands-ServerConversation-ID` (`mcp.py:115`), `ProviderType.GITHUB` (`mcp.py:122`)
- `create_mr` -> IDENTITY: `X-OpenHands-ServerConversation-ID` (`mcp.py:188`), `ProviderType.GITLAB` (`mcp.py:195`)
- `create_bitbucket_pr` -> IDENTITY: `X-OpenHands-ServerConversation-ID` (`mcp.py:255`), `ProviderType.BITBUCKET` (`mcp.py:262`)
- `create_bitbucket_data_center_pr` -> IDENTITY: `X-OpenHands-ServerConversation-ID` (`mcp.py:322`), `ProviderType.BITBUCKET_DATA_CENTER` (`mcp.py:329`)
- `create_azure_devops_pr` -> IDENTITY: `X-OpenHands-ServerConversation-ID` (`mcp.py:389`), `ProviderType.AZURE_DEVOPS` (`mcp.py:396`)

Provider credential retrieval + session ID access reachable from all five VCS tools with no gate.

**Unclassified CEEs (65):** All LOW_RISK_UNTRACED. TypeScript/JavaScript frontend: UI components (`ChatInputActions`, `ConversationPanel`, `Dialog`, `Tooltip`, `EventHandler`), event handlers (`handleKeyDown`, `handleScroll`, `handleVisibilityChange`, `handleStorage`), hooks (`useHandleWSEvents`), type guards (`isExecuteBashActionEvent`), test utilities (`createMockExecuteBashActionEvent`), VSCode extension stubs (`activate`), action generators (`generateAssistantMessageAction`), mutation wrappers (`mutateWithToast`).

**Coverage gap:** 4,849 unresolved call edges. 65 unique entrypoints traced to sensitive ops -- all from VCS MCP tools in `mcp.py`. Agent's core execution capabilities (shell commands, file edits, browser) operate through separate runtime path not fully resolved in this scan. Primary capability surface not traced.

**Key points:**
1. No CRITICAL findings in classified set. However, 4,849 unresolved edges mean OpenHands' primary capability surface (shell execution, filesystem, browser) was not fully traced. May harbor ungoverned paths not captured.
2. All 5 WARNING findings share single vulnerable code site (`mcp.py:84`): structural missing authorization check, not isolated tool-specific gap. Five VCS providers, one ungoverned append.
3. All 10 INFO findings: provider credential retrieval + session ID access (`provider_tokens.get`, `headers.get('X-OpenHands-ServerConversation-ID')`) from same five VCS tools, no gate.
4. 65 unclassified CEEs entirely frontend UI code (TypeScript event handlers, hooks, type guards) -> zero HIGH/MEDIUM risk untraced tools in classified set, but this reflects tracing scope, not agent runtime safety.
5. 100% of classified findings (15/15) ungoverned. MCP server layer has no implemented authorization controls in analyzed paths.

---

## 9. SuperAGI
**Repo:** TransformerOptimus/SuperAGI | **Framework:** langchain, registration | **Files scanned:** 362 | **Duration:** 5.4s

**CEE breakdown:** 125 total | 85 classified | 40 unclassified | CRITICAL: 2 | WARNING: 48 | INFO: 35 | UNCLASSIFIED: 40

**AFB04 Governance: UNGOVERNED** (100% of classified findings, no policy gate detected)

**Critical findings:**
1. `_execute (delete_file)` -> `os.remove(final_path)` at `delete_file.py:61`. Entry: `delete_file.py:36`. Framework: langchain. Tool-controlled `final_path`. Agent's instruction context determines which file is deleted with no intercepting gate. Self-hosted deployment with broad filesystem access -> arbitrary file deletion within accessible path space.
2. `_execute (read_file)` -> `os.remove(temporary_file_path)` at `read_file.py:99`. Entry: `read_file.py:43`. Tool-controlled `temporary_file_path`. Read-oriented tool triggers file deletion as part of temp cleanup. No path sanitization/sandboxing -> `temporary_file_path` can be influenced to target files outside intended temp dir. Unexpected secondary destructive capability.

**Warning findings:**
- NETWORK (8+): `apollo_search.py:127` (tool-controlled URL in `requests.post`), `dalle_image_gen.py:69` (tool-controlled URL, `requests.get`), `stable_diffusion_image_gen.py:83` (tool-controlled, Stability AI endpoint `requests.post`), `instagram.py:177/185/193` (3 separate `requests.post` calls: account ID retrieval, media container creation, media publishing). Marketplace UI: `DashboardService.js:56` (`handleAddTool`, api.post), `DashboardService.js:132` (`installFromMarketplace`, api.put), `DashboardService.js:176` (`handleInstallClick`, api.post). `search_scraper.py:41` (Searx, tool-controlled `/search`). All ungoverned.
- FILESYSTEM (10): `append_file.py:69/70/71` -- tool-controlled `directory`, `final_path`, `content` -> `os.makedirs` + `open('a+')` + `file.write(content)`. Agent controls both path and written content. `read_file.py:68/70/75/89` -- tool-controlled `temporary_file_path`, `contents`, `directory`. `list_files.py:68` -- instruction-influenced `file` via `os.walk`. `resource_helper.py:142` (cross-file, `delete_file` and `append_file` paths).
- STATE_MUTATION (14+): `.append()` across: `apollo_search.py:78` (`first_name`), `write_code.py:106` (`file_name`), `duck_duck_go_search.py:66/96/126` (`href`, tool-controlled `title`, `content`), `read_email.py:61` (`email_msg`), `send_email_attachment.py:123` (`draft_folder`), `send_email.py:67` (`draft_folder`), `validate_csv.py:17` (`row`), `review_pull_request.py:118/123` (`current_part`), `create_calendar_event.py:38` (`email_id`), `list_calendar_events.py:66` (`event_id`), `google_search.py:64` (`snippets`), `stable_diffusion_image_gen.py:62` (`base64`), `search_scraper.py:97/102` (`result`).
- COMMUNICATION (4): `send_email_attachment.py:137/141` and `send_email.py:81/85` -> `smtplib.SMTP` (connection) and `smtp.send_message` (transmission). Instruction-influenced input traced to `send_message`. No recipient validation, rate limiting, or human-in-the-loop gate. Outbound email fully ungoverned.
- INFRASTRUCTURE -- Google Calendar (6+): `create_calendar_event.py:63` (tool-controlled `primary`, `service.events().insert` + `execute`), `delete_calendar_event.py:33` (`service.events().delete` + execute), `event_details_calendar.py:31`, `list_calendar_events.py:53/66/73-77`. All ungoverned. Agent freely modifies user's primary calendar.

**Info findings (35):** `builtins.open` / `builtins.open.read` from `prompt_reader.py:9/10` (for `improve_code`, `write_code`, `review_pull_request`, `thinking/tools` tools). `validate_csv.py` open/read. `token_counter.py:62` dict.keys. `improve_code.py:83/88` result.get/response.get. `read_email.py:79` part.get Content-Disposition. `send_email_attachment.py:75/77/78` os.path.exists + builtins.open + read for final_path. `list_files.py:61` os.walk. `read_file.py:72` os.path.exists. Calendar event get/list ops. `instagram.py:203/204` builtins.open + read. `search_scraper.py:82/85/88/91/93` result div parsing.

**Unclassified CEEs (40):**
- HIGH_RISK_UNTRACED (4): `send_email.py:31`, `send_email_attachment.py:47`, `send_message.py:38`, `send_tweets.py:24`.
- MEDIUM_RISK_UNTRACED (8): `write_file.py:39`, `delete_file.py:36`, `delete_file.py:50` (GitHub), `read_file.py:43`, `append_file.py:38`, `list_files.py:33`, `add_file.py:52` (GitHub), `search_repo.py:42`.
- LOW_RISK_UNTRACED (28): Frontend UI handlers (AgentCreate, AgentSchedule, KnowledgeForm, KnowledgeTemplate, ToolkitWorkspace: `clearLocalStorage`, `handleClickOutside`). `base_tool.py:171` (tool base class registration). Code tools: `improve_code.py`, `write_code.py`. Search: `duck_duck_go.py`, `google_search.py`, `google_serp_search.py`, `searx.py`. GitHub: `fetch_pull_request.py`, `review_pull_request.py`. Calendar: `create_calendar_event.py`, `delete_calendar_event.py`, `event_details_calendar.py`, `list_calendar_events.py`. Image: `dalle_image_gen.py`, `stable_diffusion_image_gen.py`. Social: `instagram.py`, `tools.py:36`, `tools.py:48`. Knowledge: `knowledge_search.py`, `query_resource.py`, `apollo_search.py`.

**Coverage gap:** 966 unresolved edges. Only 9 unique entrypoints traced to sensitive ops (low relative to breadth). Marketplace installation mechanism (WARNING findings via API post) partially analyzed; downstream execution paths of installed marketplace tools fall in coverage gap.

**Key points:**
1. Both CRITICALs: `os.remove` with tool-controlled input -- one from dedicated delete tool, one as side effect of read tool's temp cleanup. Agent-controlled args determine file deletion targets, no gate.
2. 100% of 85 classified findings ungoverned. 9 traced entrypoints confirm no authorization boundary between tool invocation and sensitive op execution.
3. Email sending (`send_email.py`, `send_email_attachment.py`) doubly exposed: both appear as WARNINGs (instruction-influenced input -> `smtp.send_message`) AND as HIGH_RISK_UNTRACED unclassified CEEs. Outbound email ungoverned through both resolved and unresolved paths.
4. Marketplace installation pathway (`handleAddTool`, `installFromMarketplace`, `handleInstallClick`) contributes 3 WARNING-level network findings. Tool installation itself ungoverned -> malicious marketplace tool installs and executes in same ungoverned environment as built-ins. Supply chain risk.
5. Google Calendar mutation (create, delete, list, get) all ungoverned with tool-controlled input in insert arguments. Agent freely modifies user's primary calendar.
6. Low entrypoint trace count (9) vs 40 unclassified CEEs + 966 edges: scan substantially underestimates reachability. 8 MEDIUM_RISK_UNTRACED file op tools likely have unresolved filesystem paths.

---

## 10. Sweep (sweepai/sweep)
**Repo:** sweepai/sweep | **Framework:** sweepai | **Files scanned:** 227 | **Duration:** 12.1s

**CEE breakdown:** 18 total | 15 classified | 3 unclassified | CRITICAL: 1 | WARNING: 14 | INFO: 0 | UNCLASSIFIED: 3

**AFB04 Governance: UNGOVERNED** (100% of classified findings, no policy gate detected)

**Critical findings:**
1. `ripgrep` -> `subprocess.run(...)` at `question_answerer.py:270`. Entry: `question_answerer.py:261`. Framework: sweepai. Tool-controlled input: **Yes**. Instruction-influenced: **Yes**. Resource hint: `rg`. Both tool-controlled and instruction-influenced input flow into `subprocess.run` arguments, no gate. Adversarially crafted GitHub issue -> argument injection into `rg` subprocess -> arbitrary file reads, path traversal, or command execution. Sweep operates with codebase read access + GitHub write access: compromise could enable secret exfiltration or malicious code injection into PRs.

**Warning findings:**
- STATE_MUTATION (13): Append ops to `snippets`, `missing_files`, `bad_files`, `all_ranges`, `snippet_indexes`, `relevant_files` across four sweepai tools:
  - `semantic_search` -> `question_answerer.py:160` (`snippet.expand(expand_size).denotation`), `:166` (`f"Snippet already retrieved previously: {snippet.denotation}"`)
  - `vector_search` -> `question_answerer.py:201` (`snippet.expand(expand_size).denotation`), `:207` (`f"Snippet already retrieved previously: {snippet.denotation}"`)
  - `submit_task` -> `question_answerer.py:360` (`file_path`), `:361` (`content`)
  - `add_files_to_context` -> `question_answerer.py:433` (`file_name`), `:449` (`snippet.start`), `:450` (`i`), `:451` (`file_range`), `:458` (`file_contents`), `:475` (`i`), `:485` (`file_contents`)
  - Instruction-influenced in all but one (`missing_files.append` from `submit_task`).
- STATE_MUTATION with tool-controlled input (1, highest-severity WARNING):
  - `ask_question_about_codebase` -> `search_agent.py:204`: appends to `llm_state["questions_and_answers"]` with tool-controlled `question`. Adversarially crafted issue text -> LLM state store that drives subsequent tool calls including the subprocess-executing `ripgrep` tool.

**Info findings:** 0.

**Unclassified CEEs (3):**
- HIGH_RISK_UNTRACED (1): `submit_task` at `search_agent.py:218`. Same tool name that produced WARNINGs from `question_answerer.py` -- distinct registration in `search_agent.py`, unresolved path. May represent authorization bypass surface.
- MEDIUM_RISK_UNTRACED (2): `view_file` (`question_answerer.py:297`), `done_file_search` (`question_answerer.py:501`). Filesystem-adjacent ops, full execution paths not resolved.
- LOW_RISK_UNTRACED: 0.

**Coverage gap:** 331 unresolved call edges (1.5 per file, lowest density in evaluated set). Unique entrypoints: not reported.

**Key points:**
1. Only system with CRITICAL finding traceable through a complete chain from external input to process execution: crafted GitHub issue -> `ask_question_about_codebase` (`llm_state` mutation) -> `ripgrep` -> `subprocess.run`, no gate at any step.
2. All 15 classified findings ungoverned.
3. `ask_question_about_codebase` (`search_agent.py:204`): tool-controlled input -> `llm_state["questions_and_answers"]`. State store coordinates all subsequent tool invocations including subprocess-executing `ripgrep`. Direct chain from issue content to process exec.
4. Four sweepai tools produce 13 WARNING STATE_MUTATION findings in `question_answerer.py`: file paths, content, code snippet ranges -> collections feeding code-writing and PR-generation logic.
5. `submit_task` appears as both WARNING-producing tool (via `question_answerer.py`) and HIGH_RISK_UNTRACED (via `search_agent.py:218`): two distinct registration paths, one traced, one not. Potential authorization bypass surface.
6. Full attack chain: crafted issue text -> `subprocess.run` argument injection -> codebase modification -> PR creation. No policy gate at any point. Write access to production codebases makes this the clearest end-to-end threat chain of all 10 systems.

---

## Cross-System Summary Table

| System | Files | CEEs | CRITICAL | WARNING | Governance | Notable |
|--------|-------|------|----------|---------|------------|---------|
| AgentGPT | 234 | 29 | 0 | 10 | UNGOVERNED | Sid.ai private KB reachable via tool-controlled POST |
| Agno | 2,719 | 239 | 2 | 90 | UNGOVERNED | `subprocess.check_output(shell=True)`, 26 HIGH_RISK_UNTRACED tools |
| AutoGPT | 1,735 | 556 | 3 | 92 | UNGOVERNED | Tool-controlled input -> `asyncio.create_subprocess_exec`, 7,432-edge gap |
| Eliza | 513 | 198 | 2 | 78 | UNGOVERNED | `Bun.spawn` via plugin loader, 10,399-edge gap, social media attack surface |
| GPT Pilot | 302 | 232 | 0 | 88 | UNGOVERNED | SSRF via `self.endpoint`, shell exec path unresolved |
| GPT Researcher | 278 | 40 | 0 | 0 | UNGOVERNED | 2,322-edge gap (8.4/file), largest unknown surface |
| MetaGPT | 660 | 455 | 10 | 191 | UNGOVERNED | Most CRITICALs; `write_prd` -> shell exec in first pipeline stage |
| OpenHands | 1,702 | 80 | 0 | 5 | UNGOVERNED | Core runtime not traced; VCS tools share single ungoverned append |
| SuperAGI | 362 | 125 | 2 | 48 | UNGOVERNED | `os.remove` tool-controlled, marketplace supply chain risk |
| Sweep | 227 | 18 | 1 | 14 | UNGOVERNED | Clearest end-to-end chain: GitHub issue -> `subprocess.run`, no gate |

**Universal finding across all 10 systems: 100% of classified findings UNGOVERNED. No policy gate detected on any analyzed call path in any system.**
