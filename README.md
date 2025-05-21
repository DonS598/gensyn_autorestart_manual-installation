The script starts the node itself. So you can run the node from the script with the command **screen -S autorestart bash ~/autorestart_bash.sh**

HOWEVER, this script is not designed for the initial installation of the node. The script only works with an already installed node.

**Installation**

1. Create the first script:

`nano ~/start_gensyn_bash.sh`

Paste the script below, then save and exit with `Ctrl + O`, `Enter`, `Ctrl + X`:

```
#!/bin/bash

cd ~/rl-swarm || exit 1
python3 -m venv .venv
source .venv/bin/activate

# Чтение параметров
SWARM="$1"
SIZE="$2"

# Запуск с автоматическими ответами
{
    echo "Y"        # Testnet
    sleep 1
    echo "$SWARM"   # A or B
    sleep 1
    echo "$SIZE"    # 0.5, 1.5, etc.
    sleep 1
    echo "N"        # Push to HF
} | ./run_rl_swarm.sh
```

Make scrypt executable 
`chmod +x ~/start_gensyn_bash.sh`


3. Create the second script:

`nano ~/autorestart_bash.sh`

Paste the script below, then save and exit with `Ctrl + O`, `Enter`, `Ctrl + X`:

```
#!/bin/bash

LOG_FILE="$HOME/rl-swarm/gensynnode.log"
RESTART_LOG="$HOME/gensyn_restart.log"
PARAM_FILE="$HOME/gensyn_params.txt"
COUNTER_FILE="$HOME/gensyn_restart_count.txt"

rm -f "$PARAM_FILE"

ask_params() {
    echo "Choose swarm type (A or B):"
    read SWARM
    echo "Choose model size (0.5 / 1.5 / 7 / 32 / 72):"
    read SIZE
    echo "$SWARM" > "$PARAM_FILE"
    echo "$SIZE" >> "$PARAM_FILE"
}

stop_node() {
    echo "[STEP] Stopping node..." | tee -a "$RESTART_LOG"
    pkill -f run_rl_swarm.sh
    pkill -f train_single_gpu
}

start_node() {
    echo "[STEP] Starting node..." | tee -a "$RESTART_LOG"
    readarray -t PARAMS < "$PARAM_FILE"
    SWARM="${PARAMS[0]}"
    SIZE="${PARAMS[1]}"
    
    > "$LOG_FILE"

    screen -S gensynnode -dm bash -c "~/start_gensyn_bash.sh $SWARM $SIZE >> $LOG_FILE 2>&1"
}

has_error() {
    grep -qE "current.?batch|UnboundLocalError|Daemon failed to start|FileNotFoundError" "$LOG_FILE"
}

increment_counter() {
    count=$(cat "$COUNTER_FILE" 2>/dev/null || echo 0)
    count=$((count + 1))
    echo "$count" > "$COUNTER_FILE"
    echo "$count"
}

monitor() {
    echo "[INFO] Monitoring started: $(date)" | tee -a "$RESTART_LOG"
    while true; do
        if has_error; then
            CNT=$(increment_counter)
            echo "[ERROR] Restarting node #$CNT..." | tee -a "$RESTART_LOG"
            stop_node
            sleep 10
            start_node
            echo "[✓] Restarted at $(date)" | tee -a "$RESTART_LOG"
            sleep 300
        fi
        sleep 15
    done
}

ask_params
stop_node
start_node
monitor
```

4. Make second script executable:

`chmod +x ~/autorestart_bash.sh`

5. Run the script in a screen session:

`screen -S autorestart bash ~/autorestart_bash.sh`

6. View logs:

`tail -f ~/rl-swarm/gensynnode.log`

**Description**

If the node stops due to one of the following errors:
1. `UnboundLocalError: cannot access local variable 'current_batch'`
2. `hivemind.p2p.p2p_daemon_bindings.utils.P2PDaemonError: Daemon failed to start in 30.0 seconds`
3. `FileNotFoundError: [Errno 2] No such file or directory`

…it will automatically restart using the parameters chosen during the initial manual launch.

We believe in this project and hope the issues will be fixed soon)

