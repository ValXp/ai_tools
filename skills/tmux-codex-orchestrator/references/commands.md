# Command Recipes (Direct Orchestration)

Use these commands from the controller pane. Do not require user-run scripts.

## Spawn one worker with the initial task attached

```bash
# Always derive the live controller pane/window from tmux state.
CONTROLLER="$(tmux display-message -p '#{session_name}:#{window_index}.#{pane_index}')"
CONTROLLER_WIN="${CONTROLLER%.*}"

submit_prompt() {
  local target="$1"
  shift
  tmux send-keys -t "$target" C-u
  tmux send-keys -t "$target" "$*"
  tmux send-keys -t "$target" C-m
  sleep 0.4
  tmux send-keys -t "$target" C-m
}

PAYLOAD="$(mktemp "/tmp/tmux_worker_payload_${CONTROLLER//[:.]/_}_XXXXXX.txt")"
EVENT_CH="tmux_worker_event_$(date +%s%N)"
PROMPT_FILE="$(mktemp "/tmp/tmux_worker_prompt_${CONTROLLER//[:.]/_}_XXXXXX.txt")"

cat > "$PROMPT_FILE" <<EOF
Investigate and fix the failing login flow in /root/app. Work autonomously: inspect the repo, make the changes you judge are needed, and run the relevant verification. Do not start other agents or use tmux orchestration beyond this handoff. When you either complete the task or hit a real blocker requiring controller input, write a short payload to ${PAYLOAD} using a shell command. The first line must be STATUS=done or STATUS=blocked. If done, include concrete outputs, files changed, verification run, and residual risk. If blocked, include the exact question or decision needed. After writing the file, run tmux wait-for -S ${EVENT_CH}.
EOF

WORKER="$(tmux split-window -d -h -t "$CONTROLLER" -P -F '#{session_name}:#{window_index}.#{pane_index}' \
  "codex --no-alt-screen --dangerously-bypass-approvals-and-sandbox \"\$(cat '$PROMPT_FILE')\"")"
[ "${WORKER%.*}" = "$CONTROLLER_WIN" ] || { echo "worker spawned in wrong window: $WORKER"; exit 1; }
```

## Spawn an idle worker only when you truly need one

```bash
WORKER="$(tmux split-window -d -h -t "$CONTROLLER" -P -F '#{session_name}:#{window_index}.#{pane_index}' \
  'codex --no-alt-screen --dangerously-bypass-approvals-and-sandbox')"
[ "${WORKER%.*}" = "$CONTROLLER_WIN" ] || { echo "worker spawned in wrong window: $WORKER"; exit 1; }
```

## Delegate a later task with event contract

```bash
PAYLOAD="$(mktemp "/tmp/tmux_worker_${WORKER//[:.]/_}_payload_XXXXXX.txt")"
EVENT_CH="tmux_worker_${WORKER//[:.]/_}_event_$(date +%s%N)"

submit_prompt "$WORKER" \
  "Investigate and fix the failing login flow in /root/app. Work autonomously: inspect the repo, make the changes you judge are needed, and run the relevant verification. Do not start other agents or use tmux orchestration beyond this handoff. When you either complete the task or hit a real blocker requiring controller input, write a short payload to ${PAYLOAD} using a shell command. The first line must be STATUS=done or STATUS=blocked. If done, include concrete outputs, files changed, verification run, and residual risk. If blocked, include the exact question or decision needed. After writing the file, run tmux wait-for -S ${EVENT_CH}."
```

## Nudge worker without micromanaging

```bash
submit_prompt "$WORKER" \
  "Keep your current approach, but the failure is in token refresh rather than session creation. Re-focus there, rerun the relevant checks, and continue."
```

## Wait for worker signal

```bash
timeout 10m tmux wait-for "$EVENT_CH"
```

## Ingest payload in controller

```bash
sed -n '1,200p' "$PAYLOAD"

# Optional quick branch check:
head -n 1 "$PAYLOAD"
```

## Continue after blocker with a fresh handoff

```bash
NEXT_PAYLOAD="$(mktemp "/tmp/tmux_worker_${WORKER//[:.]/_}_payload_XXXXXX.txt")"
NEXT_EVENT_CH="tmux_worker_${WORKER//[:.]/_}_event_$(date +%s%N)"

submit_prompt "$WORKER" \
  "Use this controller decision and continue: switch to the refresh-token path. For the next handoff, write your payload to ${NEXT_PAYLOAD}. The first line must be STATUS=done or STATUS=blocked. After writing the file, run tmux wait-for -S ${NEXT_EVENT_CH}."

timeout 10m tmux wait-for "$NEXT_EVENT_CH"
sed -n '1,200p' "$NEXT_PAYLOAD"
```

## Teardown worker pane

```bash
tmux kill-pane -t "$WORKER"
tmux list-panes -t "$CONTROLLER_WIN" -F 'pane=#{pane_index} active=#{pane_active} cmd=#{pane_current_command}'
```

## Capture worker output only for diagnostics

```bash
# Only use this after the wait timeout expires or a handoff has clearly failed.
tmux capture-pane -p -t "$WORKER" -S -180
```

## Two workers (same window)

```bash
CONTROLLER="$(tmux display-message -p '#{session_name}:#{window_index}.#{pane_index}')"
CONTROLLER_WIN="${CONTROLLER%.*}"

F1="$(mktemp "/tmp/tmux_worker_prompt_${CONTROLLER//[:.]/_}_XXXXXX.txt")"
P1="$(mktemp "/tmp/tmux_worker_payload_${CONTROLLER//[:.]/_}_XXXXXX.txt")"
E1="tmux_worker_event_$(date +%s%N)"

cat > "$F1" <<EOF
Own the first independent task. Inspect context, choose the approach, and run the relevant verification. Write STATUS=done or STATUS=blocked to ${P1}, then signal with tmux wait-for -S ${E1}.
EOF

W1="$(tmux split-window -d -h -t "$CONTROLLER" -P -F '#{session_name}:#{window_index}.#{pane_index}' \
  "codex --no-alt-screen --dangerously-bypass-approvals-and-sandbox \"\$(cat '$F1')\"")"
[ "${W1%.*}" = "$CONTROLLER_WIN" ] || { echo "W1 spawned in wrong window: $W1"; exit 1; }

F2="$(mktemp "/tmp/tmux_worker_prompt_${CONTROLLER//[:.]/_}_XXXXXX.txt")"
P2="$(mktemp "/tmp/tmux_worker_payload_${CONTROLLER//[:.]/_}_XXXXXX.txt")"
E2="tmux_worker_event_$(date +%s%N)"

cat > "$F2" <<EOF
Own the second independent task. Inspect context, choose the approach, and run the relevant verification. Write STATUS=done or STATUS=blocked to ${P2}, then signal with tmux wait-for -S ${E2}.
EOF

W2="$(tmux split-window -d -v -t "$W1" -P -F '#{session_name}:#{window_index}.#{pane_index}' \
  "codex --no-alt-screen --dangerously-bypass-approvals-and-sandbox \"\$(cat '$F2')\"")"
[ "${W2%.*}" = "$CONTROLLER_WIN" ] || { echo "W2 spawned in wrong window: $W2"; exit 1; }

timeout 10m tmux wait-for "$E1"
timeout 10m tmux wait-for "$E2"
sed -n '1,200p' "$P1"
sed -n '1,200p' "$P2"
```
