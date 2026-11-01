
cat << 'EOF' > script.groovy
import jenkins.model.*
import hudson.model.*
import hudson.tasks.Shell

def instance = Jenkins.get()
def jobName = 'lesson7-parallel-demo'

def job = instance.getItem(jobName) ?: instance.createProject(FreeStyleProject, jobName)

// Register parallel execution thread count parameter
def threadsParam = new StringParameterDefinition('PARALLEL_THREADS', '3', 'Number of parallel worker subshells')
job.removeProperty(ParametersDefinitionProperty.class)
job.addProperty(new ParametersDefinitionProperty(threadsParam))

String parallelShellScript = '''#!/usr/bin/env bash
set -euo pipefail

echo "=========================================================="
echo "  HOURS 1-2: ASYNCHRONOUS SUBSHELL PARALLELISM"
echo "=========================================================="

# Directory for subshell thread logs
mkdir -p ./logs/threads

# Arrays to capture Process IDs (PIDs) and worker indices
PIDS=()
WORKERS=(${PARALLEL_THREADS})

echo "--> Step 1: Spawning ${PARALLEL_THREADS} parallel subshell tasks..."

worker_task() {
    local worker_id=$1
    local log_file="./logs/threads/worker_${worker_id}.log"
    
    echo "[WORKER ${worker_id}] Task started at $(date +%H:%M:%S)" > "${log_file}"
    
    # Simulate non-blocking asynchronous work with varying execution times
    local sleep_time=$(( (RANDOM % 3) + 1 ))
    sleep "${sleep_time}"
    
    echo "[WORKER ${worker_id}] Completed work in ${sleep_time}s" >> "${log_file}"
}

# Loop to spawn background subshells
for i in $(seq 1 "${PARALLEL_THREADS}"); do
    worker_task "${i}" &
    PID=$!
    PIDS+=("${PID}")
    echo "  [SPAWNED] Worker Thread ${i} assigned PID ${PID}"
done

echo -e "\n--> Step 2: Synchronizing subshell completion (Waiting on PIDs: ${PIDS[*]})..."

# Track failure states across threads
FAILED=0

for idx in "${!PIDS[@]}"; do
    PID="${PIDS[$idx]}"
    WORKER_NUM=$((idx + 1))
    
    if wait "${PID}"; then
        echo "  [SYNC SUCCESS] Worker ${WORKER_NUM} (PID ${PID}) finished cleanly."
    else
        echo "  [SYNC FAILURE] Worker ${WORKER_NUM} (PID ${PID}) exited with non-zero status!" >&2
        FAILED=1
    fi
done

if [ "${FAILED}" -ne 0 ]; then
    echo "[ERROR] One or more parallel subshells failed execution!" >&2
    exit 1
fi

echo -e "\n--> Step 3: Inspecting aggregated parallel thread logs:"
for log in ./logs/threads/*.log; do
    echo "--- File: ${log} ---"
    cat "${log}"
done

echo "[SUCCESS] All parallel tasks synchronized successfully."
'''

job.getBuildersList().clear()
job.getBuildersList().add(new Shell(parallelShellScript))
job.save()

println "LESSON7_HOURS1_2_PROVISIONED"
EOF

# Deploy to Jenkins via /scriptText
curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:$API_TOKEN" \
  --data-urlencode "script@script.groovy"
rm script.groovy




# 1. Fetch upcoming build ID dynamically
NEXT_BUILD_NUM=$(curl -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson7-parallel-demo/api/json" \
  | sed -n 's/.*"nextBuildNumber":\([0-9]*\).*/\1/p')

if [ -z "${NEXT_BUILD_NUM}" ]; then
  NEXT_BUILD_NUM=1
fi

echo "--> Target Build Number: ${NEXT_BUILD_NUM}"

# 2. Trigger build with PARALLEL_THREADS=4
curl -X POST -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson7-parallel-demo/buildWithParameters?PARALLEL_THREADS=4"

# Wait for queue ingestion and parallel subshell execution
sleep 5

# 3. Stream console output
echo "=========================================================="
echo "  CONSOLE LOG OUTPUT (BUILD #${NEXT_BUILD_NUM})"
echo "=========================================================="
curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson7-parallel-demo/${NEXT_BUILD_NUM}/consoleText"


***********************************************************





cat << 'EOF' > script.groovy
import jenkins.model.*
import hudson.model.*
import hudson.tasks.Shell

def instance = Jenkins.get()
def jobName = 'lesson7-parallel-demo'
def job = instance.getItem(jobName)

String timeoutShellScript = '''#!/usr/bin/env bash
set -euo pipefail

# Maximum allowable execution window in seconds
TIMEOUT_LIMIT=3

echo "=========================================================="
echo "  HOURS 3-4: PROCESS ISOLATION & TIMEOUT GUARDS"
echo "=========================================================="

# Global PID cleanup registry
ACTIVE_PIDS=()

# Comprehensive termination handler to kill orphaned processes
cleanup_processes() {
    local exit_code=$?
    echo -e "\n[CLEANUP] Executing process isolation sweep (Exit Code: ${exit_code})..."
    for pid in "${ACTIVE_PIDS[@]}"; do
        if kill -0 "${pid}" 2>/dev/null; then
            echo "  [TERMINATING] Terminating lingering process PID ${pid}..."
            kill -9 "${pid}" 2>/dev/null || true
        fi
    done
}

# Bind cleanup handler to EXIT and SIGTERM signals
trap cleanup_processes EXIT SIGTERM

# Simulate long-running worker subshell
long_running_worker() {
    echo "[WORKER] Initialized hanging work thread (PID $$)..."
    # Intentionally exceeds the 3-second limit
    sleep 10
    echo "[WORKER] Task finished successfully."
}

# 1. Launch worker in background
long_running_worker &
WORKER_PID=$!
ACTIVE_PIDS+=("${WORKER_PID}")
echo "--> Worker spawned with PID: ${WORKER_PID}"

# 2. Launch watchdog process in background
watchdog() {
    local target_pid=$1
    local limit=$2
    sleep "${limit}"
    if kill -0 "${target_pid}" 2>/dev/null; then
        echo -e "\n[TIMEOUT EXCEEDED] Worker PID ${target_pid} passed ${limit}s threshold!" >&2
        echo "[TIMEOUT EXCEEDED] Dispatching SIGTERM to force worker termination..." >&2
        kill -15 "${target_pid}" 2>/dev/null || true
    fi
}

watchdog "${WORKER_PID}" "${TIMEOUT_LIMIT}" &
WATCHDOG_PID=$!
ACTIVE_PIDS+=("${WATCHDOG_PID}")
echo "--> Watchdog initialized with PID: ${WATCHDOG_PID} (Limit: ${TIMEOUT_LIMIT}s)"

# 3. Wait for worker process
echo "--> Main process entering synchronization block..."
if wait "${WORKER_PID}"; then
    echo "[SUCCESS] Worker completed before timeout threshold."
    # Kill watchdog since worker completed naturally
    kill -9 "${WATCHDOG_PID}" 2>/dev/null || true
else
    echo "[FAILURE] Worker execution interrupted or terminated by watchdog!" >&2
    exit 1
fi
'''

job.getBuildersList().clear()
job.getBuildersList().add(new Shell(timeoutShellScript))
job.save()

println "LESSON7_HOURS3_4_PROVISIONED"
EOF

curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:$API_TOKEN" \
  --data-urlencode "script@script.groovy"
rm script.groovy






NEXT_BUILD_NUM=$(curl -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson7-parallel-demo/api/json" \
  | sed -n 's/.*"nextBuildNumber":\([0-9]*\).*/\1/p')

if [ -z "${NEXT_BUILD_NUM}" ]; then
  NEXT_BUILD_NUM=1
fi

curl -X POST -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson7-parallel-demo/buildWithParameters"

# Wait for watchdog to trigger (3-second timeout limit + queue buffers)
sleep 6

curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson7-parallel-demo/${NEXT_BUILD_NUM}/consoleText"




