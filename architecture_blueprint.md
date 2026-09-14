# Vision-Based Autonomous Desktop Agent — Architecture Blueprint

**Stack:** OmniParser v2 + pywinauto (perception) · Gemini 3 Flash/Flash-Lite + Groq + local Qwen2.5-Coder-3B (reasoning) · LangGraph + SqliteSaver (state) · PyQt6 (UI) · PyAutoGUI (execution)

---

## 1. Directory Tree

```
vision_agent/
├── main.py
├── requirements.txt
├── README.md
│
├── config/
│   ├── settings.yaml
│   ├── app_profiles.json
│   └── config_loader.py
│
├── core/
│   ├── orchestrator/
│   │   ├── graph_builder.py
│   │   ├── state_schema.py
│   │   ├── checkpointer.py
│   │   └── nodes/
│   │       ├── perception_node.py
│   │       ├── planner_node.py
│   │       ├── coder_node.py
│   │       ├── executor_node.py
│   │       └── validator_node.py
│   │
│   ├── perception/
│   │   ├── base_perceiver.py
│   │   ├── omniparser_engine.py
│   │   ├── uia_engine.py
│   │   ├── perception_router.py
│   │   └── models.py
│   │
│   ├── reasoning/
│   │   ├── base_llm_client.py
│   │   ├── gemini_client.py
│   │   ├── groq_client.py
│   │   ├── local_llm_client.py
│   │   ├── key_rotation_pool.py
│   │   └── prompt_templates/
│   │       ├── planner_prompt.txt
│   │       └── coder_prompt.txt
│   │
│   ├── execution/
│   │   ├── action_schema.py
│   │   ├── coordinate_mapper.py
│   │   ├── rescan_guard.py
│   │   ├── pyautogui_executor.py
│   │   └── safety_policy.py
│   │
│   └── memory/
│       ├── project_state_store.py
│       └── history_logger.py
│
├── ui/
│   ├── overlay_window.py
│   ├── crop_selector.py
│   ├── compression_slider.py
│   ├── hotkey_listener.py
│   └── async_bridge.py
│
├── services/
│   ├── screenshot_service.py
│   ├── image_preprocessor.py
│   └── telemetry_service.py
│
├── data/
│   ├── project_state.db
│   └── logs/agent.log
│
└── tests/
    ├── test_perception/
    ├── test_execution/
    └── test_orchestrator/
```

---

## 2. File-by-File Contract

### `main.py`
- **Responsibility:** Composition root. Wires DI container, starts PyQt6 `QApplication`, mounts `async_bridge`, registers global hotkey, boots LangGraph runtime.
- **Core objects:** `bootstrap_app()`, `AppContainer` (holds singletons: state store, LLM clients, perception router).
- **I/O:** In: CLI args/env. Out: none (process lifetime owner).

### `config/settings.yaml`
- **Responsibility:** Non-secret runtime config — model names, thresholds, crop-compression defaults, re-scan interval (100ms), timeout/retry counts.
- **Contract:** Loaded once at boot, immutable at runtime (reload requires restart).

### `config/app_profiles.json`
- **Responsibility:** Per-application knowledge — for Blender/Premiere/Fusion360: known keyboard shortcuts, which perception engine to prefer, canvas region heuristics.
- **Consumed by:** `perception_router.py`, `planner_node.py` (so the planner knows "Blender has Ctrl+R for rotate" without re-deriving it).

### `config/config_loader.py`
- **Classes/Functions:** `Settings` (Pydantic `BaseSettings`), `load_settings() -> Settings`, `load_app_profiles() -> dict[str, AppProfile]`.
- **I/O:** In: file paths. Out: validated, typed config objects consumed everywhere else — this is the **only** file allowed to touch raw YAML/JSON for config.

---

### `core/orchestrator/state_schema.py`
- **Responsibility:** Single source of truth for the shape of shared state passed between LangGraph nodes and persisted to `project_state.db`.
- **Core objects:** `AgentState(TypedDict)` — fields: `task_goal`, `active_app`, `crop_region`, `perception_result`, `plan_steps`, `current_step_index`, `last_action`, `action_history`, `retry_count`, `model_in_use`.
- **Contract:** Every node reads a subset of this dict and returns a **partial update** (LangGraph merge semantics) — nodes never mutate fields outside their declared scope.

### `core/orchestrator/graph_builder.py`
- **Responsibility:** Declares the LangGraph `StateGraph`: node registration, conditional edges (e.g. `perception → planner → coder → executor → validator`, with `validator` looping back to `planner` on failure).
- **Core objects:** `build_agent_graph(container: AppContainer) -> CompiledGraph`.
- **I/O:** In: `AppContainer` (DI). Out: compiled, invokable graph object used by `main.py`.

### `core/orchestrator/checkpointer.py`
- **Responsibility:** Wraps LangGraph's `SqliteSaver` pointed at `data/project_state.db`. Owns crash-recovery (resume last incomplete task) and thread-safe writes — **this is the only file that talks to the state DB directly**, replacing the earlier flat-JSON idea.
- **Core objects:** `get_checkpointer(db_path: str) -> SqliteSaver`.

### `core/orchestrator/nodes/perception_node.py`
- **Responsibility:** Calls `perception_router` to get structured screen data, writes it into `AgentState.perception_result`.
- **In:** `AgentState` (needs `crop_region`, `active_app`). **Out:** partial `AgentState` update with `perception_result: PerceptionResult`.

### `core/orchestrator/nodes/planner_node.py`
- **Responsibility:** Sends `perception_result` + `task_goal` + relevant `app_profiles` entry to the reasoning layer (Gemini first, via `key_rotation_pool`), gets back a high-level ordered plan (not code yet).
- **In:** `AgentState`. **Out:** `plan_steps: list[PlanStep]`.

### `core/orchestrator/nodes/coder_node.py`
- **Responsibility:** Converts the **current** plan step into a concrete `Action` (click/drag/scroll/hotkey/type) using coordinates from `perception_result`. Falls back Gemini → Groq → local Qwen2.5-Coder-3B per `key_rotation_pool` failover order.
- **In:** `AgentState.plan_steps[current_step_index]`. **Out:** `last_action: Action` (conforms to `action_schema.py`).

### `core/orchestrator/nodes/executor_node.py`
- **Responsibility:** Thin bridge — hands `last_action` to `core/execution/pyautogui_executor.py`, waits for result.
- **In:** `Action`. **Out:** `execution_result: bool/error`, appended to `action_history`.

### `core/orchestrator/nodes/validator_node.py`
- **Responsibility:** Triggers a fresh micro-perception pass (via `rescan_guard`) to confirm the action had the intended effect; decides success vs retry vs abort-to-user.
- **In:** post-action screen state. **Out:** `retry_count` update or graph termination signal.

---

### `core/perception/base_perceiver.py`
- **Responsibility:** Abstract base class — the contract every perception engine must satisfy, so `perception_router` can swap engines without caring which one runs.
- **Core objects:** `class BasePerceiver(ABC)` with abstract `capture(region: CropRegion) -> RawFrame` and `parse(frame: RawFrame) -> PerceptionResult`.

### `core/perception/omniparser_engine.py`
- **Responsibility:** Runs OmniParser v2 (icon detection + OCR) on canvas/3D-viewport regions (Blender, Fusion360, Premiere timeline).
- **Implements:** `BasePerceiver`. **In:** cropped screenshot. **Out:** `PerceptionResult` (list of `BoundingBox` + labels).

### `core/perception/uia_engine.py`
- **Responsibility:** Uses `pywinauto`/UIAutomation to read the native element tree for standard Windows dialogs/menus — no vision call, no latency.
- **Implements:** `BasePerceiver`. **In:** window handle. **Out:** `PerceptionResult` built from UIA element properties (name, control type, bounding rect).

### `core/perception/perception_router.py`
- **Responsibility:** Decides, per `active_app` (from `app_profiles.json`) and per current window under cursor, whether to dispatch to `omniparser_engine` or `uia_engine`. This is the **strategy pattern** hinge of the whole perception layer.
- **Core objects:** `class PerceptionRouter: def route(app_name, region) -> PerceptionResult`.

### `core/perception/models.py`
- **Responsibility:** Shared dataclasses: `BoundingBox(x1,y1,x2,y2,label,confidence)`, `PerceptionResult(elements: list[BoundingBox], source_engine: str, crop_offset: tuple)`.

---

### `core/reasoning/base_llm_client.py`
- **Responsibility:** ABC defining `generate(prompt, context) -> LLMResponse` — every reasoning backend (Gemini/Groq/local) implements this identically so nodes never know which model answered.

### `core/reasoning/gemini_client.py`
- **Responsibility:** Wraps Gemini 3 Flash/Flash-Lite calls; delegates actual key selection to `key_rotation_pool`.
- **Implements:** `base_llm_client`.

### `core/reasoning/groq_client.py`
- **Responsibility:** Wraps Groq-hosted Llama 3.3 / Qwen 2.5 Coder — used as secondary logic tier when Gemini quota is exhausted.
- **Implements:** `base_llm_client`.

### `core/reasoning/local_llm_client.py`
- **Responsibility:** Talks to local Ollama instance running Qwen2.5-Coder-3B-Q4_K_M — used for trivial routing/coordinate-offset micro-tasks that don't need cloud intelligence.
- **Implements:** `base_llm_client`.

### `core/reasoning/key_rotation_pool.py`
- **Responsibility:** Owns the failover chain: rotate across Gemini project keys → fall to Groq → fall to local model. Tracks per-key quota/error state, applies exponential backoff, never lets a dead key block the pipeline.
- **Core objects:** `class KeyRotationPool: def get_client(task_type) -> BaseLLMClient`.

### `core/reasoning/prompt_templates/*.txt`
- **Responsibility:** Externalized prompt text (planner vs coder prompts) — kept out of Python so prompts can be iterated without touching logic code.

---

### `core/execution/action_schema.py`
- **Responsibility:** Pydantic models for every executable action type: `ClickAction`, `DragAction`, `ScrollAction`, `TypeAction`, `HotkeyAction` — this is the **data contract** between `coder_node` and `executor_node`.

### `core/execution/coordinate_mapper.py`
- **Responsibility:** Pure function module implementing `Global X = Crop Offset X + AI X` (and Y). No side effects — easily unit-testable.
- **Core function:** `map_to_global(local_coord, crop_offset) -> GlobalCoord`.

### `core/execution/rescan_guard.py`
- **Responsibility:** Implements the 100ms pre-click re-scan safety check — calls perception layer again right before firing the action to catch UI drift (e.g. a panel that moved).
- **Core function:** `verify_before_click(action: Action) -> Action` (returns corrected coordinates or raises `DriftDetectedError`).

### `core/execution/pyautogui_executor.py`
- **Responsibility:** The **only** file that imports `pyautogui` — actually moves the mouse/keyboard. Keeping this isolated means execution can be mocked entirely in tests.
- **Core function:** `execute(action: Action) -> ExecutionResult`.

### `core/execution/safety_policy.py`
- **Responsibility:** Enforces hotkey-first preference (checks `app_profiles` for a keyboard-shortcut equivalent before allowing a mouse action), and gates any destructive-looking action (e.g. delete, overwrite-save) behind an extra confirmation callback to the UI.

---

### `core/memory/project_state_store.py`
- **Responsibility:** High-level read/write API over the same SQLite file the checkpointer uses, for **non-graph** data (user preferences, session history) — kept separate from `checkpointer.py` which is LangGraph-internal only.

### `core/memory/history_logger.py`
- **Responsibility:** Appends structured audit-log entries (action + timestamp + success/fail) — used for debugging and for the validator's retry logic.

---

### `ui/overlay_window.py`
- **Responsibility:** The PyQt6 always-on-top, transparent `QWidget` shown on `F8`. Hosts the crop selector and compression slider as child widgets.
- **Core objects:** `class OverlayWindow(QWidget)`.

### `ui/crop_selector.py`
- **Responsibility:** Mouse-drag rectangle selection tool; emits the chosen `CropRegion` (x, y, w, h) back to the orchestrator entry point.

### `ui/compression_slider.py`
- **Responsibility:** Lets the user pick downscale factor / grayscale toggle before the crop is sent for perception — directly reduces token cost per earlier design discussion.

### `ui/hotkey_listener.py`
- **Responsibility:** Registers the global `F8` hook (e.g. via the `keyboard` library) on a background thread so it works even when the app isn't focused; emits a Qt signal into the main thread on trigger.

### `ui/async_bridge.py`
- **Responsibility:** Integrates asyncio with PyQt6's event loop (via `qasync` or a `QThread`-based executor) so LangGraph's async node calls (LLM API calls) never freeze the overlay UI.

---

### `services/screenshot_service.py`
- **Responsibility:** Wraps `mss` for fast raw screen capture of the selected crop region.

### `services/image_preprocessor.py`
- **Responsibility:** Applies downscale + grayscale per the compression slider's chosen level before the frame is handed to OmniParser or an LLM vision call.

### `services/telemetry_service.py`
- **Responsibility:** Optional local metrics (latency per node, API call counts, cache-key usage) — useful once the rotation pool is live so you can see quota burn rate.

---

## 3. End-to-End Data Flow on `F8` Press

1. `hotkey_listener.py` detects `F8` → emits Qt signal → `overlay_window.py` renders on top of the screen.
2. User drags `crop_selector.py` → produces `CropRegion`; adjusts `compression_slider.py` if needed.
3. `main.py`'s bootstrap hands `CropRegion` into the compiled LangGraph (`graph_builder.py`), seeding `AgentState.crop_region`.
4. **Node: `perception_node`** → `screenshot_service.py` captures the crop → `image_preprocessor.py` compresses it → `perception_router.py` decides OmniParser vs UIA → returns `PerceptionResult`.
5. **Node: `planner_node`** → `key_rotation_pool.py` selects a live Gemini key → sends `PerceptionResult` + `task_goal` + relevant `app_profiles` entry → gets back ordered `plan_steps`.
6. **Node: `coder_node`** → for the current step, produces a concrete `Action` (using `action_schema.py`), preferring a hotkey per `safety_policy.py` if one exists.
7. **Node: `executor_node`** → `rescan_guard.py` re-captures the region (100ms), runs `coordinate_mapper.py` to confirm `Global X/Y`, then `pyautogui_executor.py` fires the actual input event.
8. **Node: `validator_node`** → triggers another light perception pass to confirm expected UI change; on success, appends to `action_history` via `history_logger.py` and advances `current_step_index`; on failure, loops back to `planner_node` with `retry_count += 1`.
9. Throughout, `checkpointer.py` persists `AgentState` after every node so a crash mid-task can resume instead of restarting.
10. Loop continues until `plan_steps` is exhausted → overlay shows completion state → `AgentState` finalized in `project_state.db`.

---

## 4. Modularity Patterns Applied

- **Abstract Base Classes (Strategy Pattern):** `base_perceiver.py` and `base_llm_client.py` mean `perception_router` and `key_rotation_pool` can add/remove engines or models without touching any node code — new backend = new file implementing the ABC, zero changes elsewhere.
- **Dependency Injection:** `AppContainer` (built once in `main.py`) holds all singletons (state store, clients, router) and is passed into `graph_builder.build_agent_graph()` — nodes receive their dependencies rather than importing globals, which makes every node independently unit-testable with mocked dependencies.
- **Data Contracts over Shared Globals:** `action_schema.py`, `models.py` (perception), and `state_schema.py` are the *only* places shapes are defined — every file that crosses a boundary (perception→planner, coder→executor) does so through one of these typed contracts, never ad-hoc dicts.
- **Isolation of Side Effects:** `pyautogui_executor.py` is the sole file touching real mouse/keyboard; `checkpointer.py`/`project_state_store.py` are the sole files touching the DB. This means tests in `tests/test_execution/` and `tests/test_orchestrator/` can fully mock these and run the entire decision pipeline without ever moving a real mouse.
- **Non-blocking asyncio + PyQt6:** `async_bridge.py` ensures the LLM/API-bound nodes (`planner_node`, `coder_node`) run as `async def` coroutines scheduled on the Qt event loop, so the floating overlay stays responsive (draggable, closeable) while a Gemini/Groq call is in flight — no UI freeze during network latency.
- **Config/Secrets Separation:** `config_loader.py` is the only file reading raw YAML/JSON; API keys themselves should live in a `.env` (not shown above, add `config/.env` + `python-dotenv`) so `key_rotation_pool.py` never hardcodes credentials.
