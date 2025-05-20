
Temporary Solution for `Current_batch`, `Daemon Failed to Start`, and `Errno` Issues: Automatic Node Restart

---

Installation

1. Update packages and install `expect`:

sudo apt update && sudo apt install expect -y

2. Create the autorestart script:


nano ~/autorestart_1.sh


Paste the script below, then save and exit with `Ctrl + O`, `Enter`, `Ctrl + X`:

```bash
#!/usr/bin/env bash

LOG_FILE="$HOME/rl-swarm/gensynnode.log"
RESTART_LOG="$HOME/gensyn_restart.log"
COUNTER_FILE="$HOME/gensyn_restart_count.txt"
PARAM_FILE="$HOME/gensyn_params.txt"
DEBUG_LOG="$HOME/debug_check.txt"

ask_params() {
    echo "[INFO] Please enter launch parameters manually."

    while true; do
        read -rp "Choose swarm type (A or B): " SWARM
        if [[ "$SWARM" == "A" || "$SWARM" == "B" ]]; then
            break
        else
            echo "Invalid input. Please enter A or B."
        fi
    done

    while true; do
        read -rp "Choose model size (0.5 / 1.5 / 7 / 32 / 72): " SIZE
        if [[ "$SIZE" =~ ^(0.5|1.5|7|32|72)$ ]]; then
            break
        else
            echo "Invalid input. Please enter one of: 0.5, 1.5, 7, 32, 72"
        fi
    done

    echo "$SWARM" > "$PARAM_FILE"
    echo "$SIZE" >> "$PARAM_FILE"
    echo "Saved parameters: $SWARM, $SIZE"
}

log() {
    echo -e "$1" | tee -a "$RESTART_LOG"
}

increment_counter() {
    count=$(cat "$COUNTER_FILE" 2>/dev/null || echo 0)
    count=$((count + 1))
    echo "$count" > "$COUNTER_FILE"
    echo "$count"
}

stop_node() {
    log "[STEP] Stopping node..."
    pkill -f run_rl_swarm.sh
    pkill -f train_single_gpu

    PID=$(netstat -tulnp 2>/dev/null | grep :3000 | awk '{print $7}' | cut -d'/' -f1)
    if [[ "$PID" =~ ^[0-9]+$ ]]; then
        sudo kill "$PID"
        log "[INFO] Killed process on port 3000: PID=$PID"
    else
        log "[WARN] Could not find valid PID for port 3000. Got: '$PID'"
    fi
}

start_node() {
    log "[STEP] Starting node inside screen..."

    if screen -list | grep -q "gensynnode"; then
        log "[INFO] Found old screen session 'gensynnode'. Closing it..."
        screen -S gensynnode -X quit
        sleep 2
    fi

    : > "$LOG_FILE"

    screen -S gensynnode -d -m bash -c "$HOME/start_node.expect $SWARM $SIZE"
}

has_critical_error() {
    awk '/Traceback \(most recent call last\)/ {t=1; c=0} t && c<50 { print; c++ }' "$LOG_FILE" |
    tee "$DEBUG_LOG" |
    grep -qF -e "UnboundLocalError: cannot access local variable 'current_batch'" \
             -e "hivemind.p2p.p2p_daemon_bindings.utils.P2PDaemonError: Daemon failed to start in 30.0 seconds" \
             -e "FileNotFoundError: [Errno 2] No such file or directory"
}

check_and_restart() {
    log "[INFO] Log monitoring started: $(date)"
    while true; do
        if has_critical_error; then
            RESTART_COUNT=$(increment_counter)
            log "[ERROR] Critical error found. Restarting node #$RESTART_COUNT..."

            stop_node
            sleep 60

            readarray -t PARAMS < "$PARAM_FILE"
            SWARM="${PARAMS[0]}"
            SIZE="${PARAMS[1]}"

            start_node
            log "[✓] Restart completed. Waiting 5 minutes before re-checking logs..."
            sleep 300
        fi
        sleep 10
    done
}

ask_params
readarray -t PARAMS < "$PARAM_FILE"
SWARM="${PARAMS[0]}"
SIZE="${PARAMS[1]}"

log "[INFO] First-time node launch..."
start_node

check_and_restart
```

3. Create the Expect script:

```bash
nano ~/start_node.expect
```

Paste the script below, then save and exit with `Ctrl + O`, `Enter`, `Ctrl + X`:

```tcl
#!/usr/bin/expect -f

set timeout 300
set swarm [lindex $argv 0]
set size  [lindex $argv 1]
set logfile "$env(HOME)/rl-swarm/gensynnode.log"

log_file -a $logfile
log_user 1

cd ~/rl-swarm
spawn ./run_rl_swarm.sh

expect {
    "select a swarm" {
        send "$swarm\r"
        exp_continue
    }
    "parameters" {
        send "$size\r"
        exp_continue
    }
    "Hugging Face Hub" {
        send "N\r"
    }
}
expect eof
```

4. Make both scripts executable:

```bash
chmod +x ~/autorestart_1.sh ~/start_node.expect
```

5. Run the script in a screen session:

```bash
screen -S autorestart bash ~/autorestart_1.sh
```

6. View logs:

```bash
tail -f ~/rl-swarm/gensynnode.log
```

---

If the node stops due to one of the following errors:
1. `UnboundLocalError: cannot access local variable 'current_batch'`
2. `hivemind.p2p.p2p_daemon_bindings.utils.P2PDaemonError: Daemon failed to start in 30.0 seconds`
3. `FileNotFoundError: [Errno 2] No such file or directory`

…it will automatically restart using the parameters chosen during the initial manual launch.

We believe in this project and hope the issues will be fixed soon!
