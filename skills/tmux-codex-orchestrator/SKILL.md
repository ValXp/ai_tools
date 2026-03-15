---
name: tmux-codex-orchestrator
description: "Orchestrate Codex workers inside tmux panes while keeping a controller session in the foreground. Use when the user asks to start a subagent, have a subagent do work, delegate work, split panes, launch worker Codex instances, capture worker outputs, coordinate event-driven worker signaling with tmux wait-for, enforce summary-to-file before worker shutdown, or ingest worker summaries into the controller context. Treat worker Codex instances like capable peers: delegate outcome-level work with context and constraints, avoid step-by-step micromanagement, and prefer short corrective nudges over rewriting the whole plan. Treat phrases like 'start a subagent to work on this', 'have a subagent do this', and 'delegate this work' as direct triggers. Prefer same-window pane orchestration unless the user explicitly asks for new windows or sessions."
---

# Tmux Codex Orchestrator

## Overview

Coordinate one controller pane plus one or more worker panes running Codex. Enforce a deterministic worker lifecycle:

1. `spawn`
2. `delegate-with-event-contract`
3. `wait-for-worker-signal`
4. `ingest-payload`
5. `continue-or-teardown`

Treat this as an orchestrator-only skill:

- Run tmux commands directly from the controller session.
- Do not ask the user to run scripts or shell wrappers.
- Keep the user in the foreground pane unless explicitly told otherwise.

## Delegation Stance

- Treat each worker like a capable coworker. Hand over an objective, relevant context, hard constraints, and the expected deliverable.
- Prefer high-freedom prompts. Let the worker inspect the repo, choose commands, sequence the work, and decide how to verify unless the task is fragile or the user requested exact steps.
- Do not pre-solve the task for the worker. If you already know every step to dictate, reconsider whether the work should be delegated at all.
- When a worker drifts or makes a mistake, send a short nudge that points at the wrong assumption, missing constraint, or sharper success criterion. Do not rewrite the whole plan unless the previous direction is unusable.
- Reserve low-freedom prompts for operational checks, reproductions, or other safety-critical sequences where exact commands materially reduce risk.

## Worker Isolation

- The controller owns orchestration. The worker owns execution of the delegated task.
- Do not name `$tmux-codex-orchestrator` in worker prompts unless the task is explicitly about testing or editing this skill.
- Do not tell the worker to read `SKILL.md`, start subagents, split panes, or orchestrate more workers unless the user explicitly asked for nested orchestration.
- Keep worker prompts task-focused. Mention only the minimum tmux contract the worker must satisfy: payload path, event channel, and when to signal.
- If a worker starts treating itself like an orchestrator, send a short correction such as: `You are the worker for this task. Do not start other agents or use the orchestration skill. Complete the task and use the provided payload/event handoff only.`

## Event Contract

- Prefer event-driven coordination over controller-side polling.
- For each controller-to-worker round trip, mint a fresh payload file and a fresh tmux channel:
  - `PAYLOAD="$(mktemp "/tmp/tmux_worker_${WORKER//[:.]/_}_payload_XXXXXX.txt")"`
  - `EVENT_CH="tmux_worker_${WORKER//[:.]/_}_event_$(date +%s%N)"`
- For the initial worker handoff, also mint a fresh prompt file:
  - `PROMPT_FILE="$(mktemp "/tmp/tmux_worker_prompt_${CONTROLLER//[:.]/_}_XXXXXX.txt")"`
- Prefer launching the worker with the initial prompt already attached on the Codex command line:
  - `tmux split-window ... "codex --no-alt-screen --dangerously-bypass-approvals-and-sandbox \"\$(cat '$PROMPT_FILE')\""`
- This avoids the fragile pattern of launching bare Codex and relying on a first newline to submit the initial prompt.
- Tell the worker to write the payload file first, then signal the channel with `tmux wait-for -S "$EVENT_CH"`.
- Require the first line of the payload to be one of:
  - `STATUS=done`
  - `STATUS=blocked`
- For `STATUS=done`, require a concise final summary with concrete outputs, files changed, verification run, and unresolved risks. This payload is the durable summary you ingest before teardown.
- For `STATUS=blocked`, require the exact missing decision, dependency, or question. After answering, mint a new payload/channel pair for the next round.
- Do not use `tmux send-keys` from the worker into the controller pane as the primary communication path. Use file-backed payloads plus `tmux wait-for`.

## Preflight

- Verify tmux connectivity and current location:
  - `echo "TMUX=${TMUX:-<empty>}"`
  - `tmux display-message -p 'session=#{session_name} window=#{window_name} pane=#{pane_index}'`
  - `CONTROLLER="$(tmux display-message -p '#{session_name}:#{window_index}.#{pane_index}')"`
  - `CONTROLLER_WIN="${CONTROLLER%.*}"`
- Refresh `CONTROLLER` immediately before every `split-window`.
- Always split from `$CONTROLLER`. Never hardcode pane targets like `demo:0.0`.
- Enforce same-window behavior by validating `${WORKER%.*} == $CONTROLLER_WIN` after each split.
- Default to pane splits in the current window. Create new windows/sessions only if explicitly requested.
- Keep the controller in foreground by using detached splits (`tmux split-window -d`).
- Use unique absolute prompt, payload, or summary file paths under `/tmp/` and unique tmux event channel names (never fixed names).
- Submit worker prompts with a reliable helper:
  - `submit_prompt() { local target="$1"; shift; tmux send-keys -t "$target" C-u; tmux send-keys -t "$target" "$*"; tmux send-keys -t "$target" C-m; sleep 0.4; tmux send-keys -t "$target" C-m; }`
- Use `tmux capture-pane` only for prompt-readiness checks or to diagnose a stuck worker.
- After the worker has accepted the handoff and the controller is waiting on `EVENT_CH`, do not poll the pane for normal progress updates.
- Prefer an explicit timeout around `tmux wait-for` (for example, `timeout 10m tmux wait-for "$EVENT_CH"`). Only inspect the worker pane if that timeout expires or the handoff is otherwise known to have failed.
- If the worker still shows queued text and no progress after prompt submission, send one more `C-m`.

## Lifecycle Commands

### Spawn a worker pane

- Prefer combining spawn plus first delegation in one step. Refresh the controller pane target, mint a unique prompt file, payload file, and event channel for the first handoff, then create the detached split in that exact pane/window:
  - `CONTROLLER="$(tmux display-message -p '#{session_name}:#{window_index}.#{pane_index}')"`
  - `CONTROLLER_WIN="${CONTROLLER%.*}"`
  - `PAYLOAD="$(mktemp "/tmp/tmux_worker_payload_${CONTROLLER//[:.]/_}_XXXXXX.txt")"`
  - `EVENT_CH="tmux_worker_event_$(date +%s%N)"`
  - `PROMPT_FILE="$(mktemp "/tmp/tmux_worker_prompt_${CONTROLLER//[:.]/_}_XXXXXX.txt")"`
  - Write the initial brief into `"$PROMPT_FILE"` with a heredoc or `printf`, including the current `PAYLOAD` and `EVENT_CH` values.
  - `WORKER="$(tmux split-window -d -h -t "$CONTROLLER" -P -F '#{session_name}:#{window_index}.#{pane_index}' "codex --no-alt-screen --dangerously-bypass-approvals-and-sandbox \"\$(cat '$PROMPT_FILE')\"")"`
  - `[ "${WORKER%.*}" = "$CONTROLLER_WIN" ] || { echo "worker spawned in wrong window: $WORKER (expected $CONTROLLER_WIN.*)"; exit 1; }`
- Wait briefly and capture only to verify prompt readiness if needed:
  - `tmux capture-pane -p -t "$WORKER" -S -80`
- If the user explicitly wants an idle worker with no initial task, launching bare Codex is still allowed:
  - `tmux split-window -d -h -t "$CONTROLLER" -P -F '#{session_name}:#{window_index}.#{pane_index}' 'codex --no-alt-screen --dangerously-bypass-approvals-and-sandbox'`

### Delegate a task

- For later rounds after the worker is already running, mint a fresh payload file and event channel for this handoff:
  - `PAYLOAD="$(mktemp "/tmp/tmux_worker_${WORKER//[:.]/_}_payload_XXXXXX.txt")"`
  - `EVENT_CH="tmux_worker_${WORKER//[:.]/_}_event_$(date +%s%N)"`
- Send a brief that covers the outcome, essential context, hard constraints, definition of done, and the event contract:
  - `submit_prompt "$WORKER" "Investigate and fix the failing login flow in /root/app. Work autonomously: inspect the codebase, make the changes you judge are needed, and run the relevant verification. Do not start other agents or use tmux orchestration beyond this handoff. When you either complete the task or hit a real blocker requiring controller input, write a short payload to ${PAYLOAD} using a shell command. The first line must be STATUS=done or STATUS=blocked. If done, include concrete outputs, files changed, verification run, and residual risk. If blocked, include the exact question or decision needed. After writing the file, run tmux wait-for -S ${EVENT_CH}. Then reply with exactly WORKER_SIGNALLED."`
- If the worker goes off course, prefer a narrow correction over a restart:
  - `submit_prompt "$WORKER" "Keep your current approach, but the failure is in token refresh rather than session creation. Re-focus there, rerun the relevant checks, and continue."`
- If queued text remains with no work progress, send one more Enter:
  - `tmux send-keys -t "$WORKER" C-m`

### Wait for worker signal

- Block on the event channel instead of polling for completion:
  - `timeout 10m tmux wait-for "$EVENT_CH"`
- Treat `tmux wait-for` as the primary synchronization path. Do not call `tmux capture-pane` while the wait is still within its timeout window.
- If the timeout expires, inspect once with `tmux capture-pane`, decide whether the worker is stuck, and only then send a recovery nudge such as another `C-m`.

### Ingest payload into controller

- Read the payload in the controller:
  - `sed -n '1,200p' "$PAYLOAD"`
- Branch on the first line:
  - `STATUS=done`: treat the payload as the final worker summary and durable memory for teardown.
  - `STATUS=blocked`: answer the worker, then mint a new `PAYLOAD` and `EVENT_CH` for the next round trip.

### Teardown worker pane

- Kill worker pane only after ingesting a `STATUS=done` payload:
  - `tmux kill-pane -t "$WORKER"`
- Re-check layout:
  - `tmux list-panes -t "$CONTROLLER_WIN" -F 'pane=#{pane_index} active=#{pane_active} cmd=#{pane_current_command}'`

## Multi-Worker Rules

- Use one unique payload file and one unique event channel per worker round trip.
- Delegate independent tasks across workers in parallel.
- Wait for and ingest each worker payload independently.
- Kill higher pane indexes first to avoid index shifts.

See [`references/commands.md`](references/commands.md) for direct command recipes.
