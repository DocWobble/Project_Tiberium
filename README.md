PROJECT TIBERIUM

Arcology Ops: Game‑shaped Admin Interface (Design Target)

0) Purpose (what this UI must make effortless)
	•	Trial models fast (A/B/C on a test set; pick winners).
	•	Wire workflows across runtimes (text ⇄ audio ⇄ video) with dependencies.
	•	Schedule tasks without collisions (respect GPU exclusive vs shared).
	•	Run/observe agents (long‑lived “copilot” plus burst jobs).
	•	Prove changes (regression/golden sets, replay, artifacts with lineage).

The implementation is free to choose methods; the end state below is non‑negotiable.

⸻

1) Core Concepts (objects & behaviors)

These define what exists and how it behaves, not how to code it.

1.1 Runtime
	•	Examples: textgen (oobabooga), video (ComfyUI), tts (TTS‑WebUI).
	•	State: online/offline, queue_len, health, endpoints (for the adapter).
	•	Exposes capabilities (e.g., text.summarize, audio.tts, video.gen).

1.2 Model
	•	Lives on a runtime.
	•	Attrs: name, hash, context_tokens, vram_cost_gb, throughput_metric (e.g., tok/s), compatibility (list of capabilities).
	•	State: loaded/unloaded, last_used_at.
	•	Actions: load, unload, promote_to_copilot.

1.3 Capability
	•	Verb tags used by tasks/workflows (ex: text.embed, ocr.pdf, text.summarize, audio.tts, video.gen).
	•	Any runtime+model can fulfill a capability if marked compatible.

1.4 Task (job card)
	•	Attrs: id, capabilities_required (one or chain), inputs (file paths or text), policy_snapshot, priority, dependencies (task ids), artifacts[].
	•	State: queued/running/done/error, timestamps, runtime_assigned, model_used.
	•	Actions: queue, cancel, retry, reroute, replay.

1.5 Policy (global rules)
	•	Examples:
	•	copilot_persistent=true|false
	•	gpu_shared_slots=2
	•	gpu_exclusive_for=["video.gen"]
	•	unload_idle_after_minutes=10
	•	Switching policies affects scheduling immediately.

1.6 Resource
	•	GPU: mode=shared|exclusive, util%, vram_used_gb.
	•	CPU/RAM: util%.
	•	Must be visible in real time and queryable.

1.7 Artifact
	•	Attrs: type (pdf, csv, wav, mp4, json), path, checksum, created_at, task_id, model_hash, seed/prompt_snapshot.

1.8 Experiment
	•	A grouped run across multiple models/blueprints on a test set.
	•	Outputs: timing, simple quality metrics, winner flag.

⸻

2) Views (screens that map to actual ops)

Each view describes UI affordances and quick actions; tech is flexible.

2.1 Control Room (Empire Map)
	•	Tiles: one per runtime (shows queue, health, loaded models as chips).
	•	Top HUD: GPU/CPU/RAM bars + lock status + active policy chips.
	•	Side Queue: global task list with filter (by project/runtime/state).
	•	Quick actions: drag task onto a tile; pause all; drain queues; toggle policies; promote/unload model from chip.

2.2 Model Testbench (Proving Ground)
	•	Prompt/Test Set panel: paste text or load a CSV/JSON of test cases.
	•	Results grid: columns = models; rows = tests; cells show latency, length, simple score (BLEU/ROUGE/regex hits/custom rubric).
	•	Actions: run A/B/C instantly; canary (subset first); promote winner; save preset.

2.3 Workflow Forge (DAG‑lite)
	•	Nodes are capabilities, not hard‑coded tools (e.g., ocr.pdf → text.summarize → audio.tts).
	•	Each node can be set to a specific model or auto.
	•	Edges: sequence, fan‑out, fan‑in merge.
	•	Actions: simulate resource fit; run as smoke test; save as blueprint; apply to project; view dependency arrows in Control Room queue.

2.4 Scheduler & Locks (Logistics Strip)
	•	A simple visual: Shared slots (N) and Exclusive lock (1) timeline.
	•	Show when an exclusive task will preempt; countdown to start.
	•	Actions: bump priority; defer until time; preempt after batch; inspect why queued.

2.5 Agent Roster (Long‑lived Workers)
	•	Table of agents (incl. Copilot): assigned blueprint, latency, last 10 tasks, memory footprint.
	•	Actions: play/pause; reset; swap model; clear memory.

2.6 Evaluation Lab (Regression)
	•	Load Golden Set; select models/blueprints; run; get a scorecard.
	•	Diff viewer to compare outputs; mark hallucinations; Promote winner to baseline.

2.7 Incidents & Replay
	•	Stream of errors/warnings; click to open Replay with pinned model/prompt/seed.
	•	Bug bundle export (inputs, logs, metadata, artifact links).

2.8 Artifact & Dataset Vault
	•	Search by project|task|model|tag.
	•	Preview; “open containing folder”; “send downstream node”.

⸻

3) Scheduling Behavior (the rules of the world)

High‑level logistics that must hold true regardless of implementation.

	1.	GPU semaphore
	•	SHARED: up to gpu_shared_slots tasks (e.g., LLM + TTS).
	•	EXCLUSIVE: single lock (e.g., video.gen) drains shared slots first.
	2.	Policy precedence
	•	copilot_persistent=true prevents unloading the copilot model and blocks new EXCLUSIVE tasks unless force=true.
	3.	Idle unload
	•	Non‑copilot models may auto‑unload after unload_idle_after_minutes.
	4.	Dependencies
	•	A task with dependencies remains blocked until all upstream done.
	5.	Batching
	•	For identical capability tasks, scheduler may batch under one model load.
	6.	Fit checks before run
	•	Validate VRAM, context window, disk space; otherwise task → error(fit_failed).

⸻

4) Minimal Data Contracts (so pieces can talk)

These are interfaces, not internal schemas. Keep flexible; extend with metadata.

4.1 Task (submit)

{
  "capability": "text.summarize",
  "inputs": { "text": "..." , "files": ["/abs/path/a.pdf"] },
  "dependencies": ["task-id-1"],
  "priority": 0,
  "policy_overrides": { "gpu_exclusive": false },
  "target": { "runtime": "textgen", "model": "auto" },
  "tags": ["project:Acme", "sprint:BoardPacket"]
}

4.2 Task (status)

{
  "id": "task-id-2",
  "state": "queued|blocked|running|done|error",
  "runtime_assigned": "textgen",
  "model_used": "qwen2.5-7b-q4",
  "eta_seconds": 42,
  "artifacts": [{"type":"md","path":"/runs/2025-08-12/.../brief.md"}],
  "logs_tail": ["..."],
  "error": null
}

4.3 Model (describe)

{
  "runtime": "textgen",
  "name": "qwen2.5-7b-q4",
  "hash": "sha256:...",
  "context_tokens": 32000,
  "vram_cost_gb": 7.8,
  "throughput": {"tokens_per_s": 75},
  "capabilities": ["text.summarize","text.generate"],
  "state": "loaded|unloaded",
  "last_used_at": "..."
}

4.4 Resource summary

{
  "gpu": {"mode": "shared|exclusive", "util_pct": 63, "vram_used_gb": 9.2},
  "cpu": {"util_pct": 28},
  "ram_gb": {"used": 38.0, "total": 96.0},
  "locks": [{"type":"exclusive","owner_task_id":"task-id-9"}]
}

These contracts let Codex wire any backend while keeping the UI consistent.

⸻

5) Golden Flows (must be smooth end‑to‑end)

5.1 Model trial → promotion
	•	Discover models on a runtime → pick 2–3 → run on Golden Set → compare metrics → Promote winner as Copilot → old copilot becomes rollback for 1 hour.

5.2 Cross‑engine workflow
	•	Build ocr.pdf → text.summarize → audio.tts blueprint → simulate fit → run → see dependency arrows in queues → artifacts appear in Vault.

5.3 GPU‑exclusive interruption
	•	Queue video.gen while copilot is active → scheduler shows pending exclusive lock and drains shared slots → video runs → lock clears → shared tasks resume.

5.4 Incident → replay
	•	Task errors → click incident → open replay with pinned model/prompt/seed → rerun or reroute → attach bug bundle to task.

⸻

6) Telemetry & Signals (prevent surprises)
	•	Fit warnings before run (context/VRAM/disk).
	•	ETA bands (optimistic/likely/pessimistic) based on recent samples.
	•	Bottleneck hints (“Waiting for exclusive lock; ~00:45”).
	•	Idle cost badge showing VRAM held by unused models.
	•	Throughput samples per model (rolling avg for tok/s or frames/min).

⸻

7) MVP Scope (smallest version that’s still great)
	•	Views: Control Room, Model Testbench, Workflow Forge, Evaluation Lab.
	•	Adapters: minimal bridges to current tools (oobabooga, ComfyUI, TTS‑WebUI).
Adapters can be stubs; behavior must match contracts.
	•	Scheduler: shared‑vs‑exclusive GPU rules + policy toggles.
	•	Persistence: tasks, artifacts, experiments, incidents (any store, even files + JSON).

Non‑goals for MVP: auth, multi‑user RBAC, long‑term subscriptions, cloud scaling.

⸻

8) Acceptance Criteria (definition of done)
	•	I can queue a text summarize task, see it run, and get an artifact with lineage.
	•	I can A/B two models on a 5‑item test set and promote the winner.
	•	I can build a 3‑node workflow (capabilities, not tool names), run it, and see dependencies honored.
	•	A video.gen task blocks shared tasks via exclusive lock, then releases correctly.
	•	I can replay a failed task with pinned model/prompt/seed and export a bug bundle.
	•	The HUD shows live GPU/CPU/RAM and clearly indicates lock state.

⸻

9) UX Notes (tone & interactions)
	•	Treat runtimes like cities/factories (tiles). Treat models as units (chips). Treat tasks as production cards (queue items).
	•	Drag‑and‑drop to schedule; right‑click or kebab menus for power actions.
	•	One‑click policy toggles with clear, reversible impact (“Copilot persistent: ON”).
	•	Always offer simulate fit before a large run.

⸻

10) Extensibility Hooks (optional, not prescriptive)
	•	Blueprints export/import as JSON (capability graph + prefs).
	•	Metrics plugin slot in Testbench to compute custom scores.
	•	Adapter registry so new runtimes can add capabilities at runtime.

⸻

11) Seed Content for the Repo (so Codex has context)
	•	/docs/
	•	design_target.md (this file)
	•	golden_set_example.json (5 prompts + expected traits)
	•	blueprint_example.json (3‑node capability DAG)
	•	/stubs/
	•	adapter_textgen_stub.md (how to call TextGen UI API shape)
	•	adapter_comfyui_stub.md
	•	adapter_tts_stub.md
	•	/samples/
	•	inputs/board_packet_sample.pdf
	•	tasks/submit_examples.json

⸻

