
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


********************************************************

cat << 'EOF' > script.groovy
import jenkins.model.*
import hudson.model.*
import hudson.tasks.Shell

def instance = Jenkins.get()
def jobName = 'lesson7-parallel-demo'
def job = instance.getItem(jobName)

String retryShellScript = '''#!/usr/bin/env bash
set -euo pipefail

echo "=========================================================="
echo "  HOURS 5-6: NON-BLOCKING TRANSIENT RETRY LOOPS"
echo "=========================================================="

# Simulated unstable service invocation
unstable_service_call() {
    local attempt=$1
    # Succeeds on attempt 3
    if [ "${attempt}" -lt 3 ]; then
        echo "[SERVICE] Connection refused (HTTP 503 Service Unavailable)" >&2
        return 1
    fi
    echo "[SERVICE] HTTP 200 OK - Data synchronized successfully."
    return 0
}

# Generic retry wrapper function
retry_operation() {
    local max_attempts=$1
    local delay=$2
    local attempt=1
    local exit_code=0

    while [ "${attempt}" -le "${max_attempts}" ]; do
        echo "--> [ATTEMPT ${attempt}/${max_attempts}] Executing target operation..."
        
        # Disable set -e temporarily to evaluate subshell exit codes safely
        set +e
        unstable_service_call "${attempt}"
        exit_code=$?
        set -e

        if [ "${exit_code}" -eq 0 ]; then
            echo "[RETRY SUCCESS] Operation succeeded on attempt ${attempt}!"
            return 0
        fi

        echo "[RETRY WARNING] Attempt ${attempt} failed with exit code ${exit_code}." >&2
        if [ "${attempt}" -lt "${max_attempts}" ]; then
            echo "  [BACKOFF] Waiting ${delay}s before retry..."
            sleep "${delay}"
            # Exponential backoff calculation
            delay=$(( delay * 2 ))
        fi
        attempt=$(( attempt + 1 ))
    done

    echo "[RETRY FATAL] Operation failed after ${max_attempts} attempts." >&2
    return "${exit_code}"
}

# Execute operation with 4 max attempts, starting with a 1-second delay
retry_operation 4 1
'''

job.getBuildersList().clear()
job.getBuildersList().add(new Shell(retryShellScript))
job.save()

println "LESSON7_HOURS5_6_PROVISIONED"
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

# Wait for retries and backoff sleeps to finish
sleep 6

curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson7-parallel-demo/${NEXT_BUILD_NUM}/consoleText"






cat << 'EOF' > script.groovy
import jenkins.model.*
import hudson.model.*
import hudson.tasks.Shell

def instance = Jenkins.get()
def jobName = 'lesson7-parallel-demo'
def job = instance.getItem(jobName)

String fallbackShellScript = '''#!/usr/bin/env bash
set -euo pipefail

mkdir -p ./build_out

echo "=========================================================="
echo "  HOURS 7-8: MULTI-STAGE FALLBACK GATES"
echo "=========================================================="

PRIMARY_REGION="us-east-1"
FAILOVER_REGION="us-west-2"

primary_deployment() {
    echo "--> Attempting deployment to Primary Region (${PRIMARY_REGION})..."
    # Simulate primary site failure
    return 1
}

secondary_fallback_deployment() {
    echo "--> [FALLBACK ACTIVATED] Deploying to Secondary Region (${FAILOVER_REGION})..."
    echo '{"active_site": "'"${FAILOVER_REGION}"'", "status": "FAILOVER_SUCCESS"}' > ./build_out/deployment.json
    echo "[FALLBACK SUCCESS] Secondary site online and routing traffic."
}

# Execute primary deployment step
PRIMARY_FAILED=0
set +e
primary_deployment
if [ $? -ne 0 ]; then
    PRIMARY_FAILED=1
fi
set -e

# Route execution based on primary status flag
if [ "${PRIMARY_FAILED}" -eq 1 ]; then
    echo "[WARNING] Primary region deployment failed!" >&2
    secondary_fallback_deployment
else
    echo '{"active_site": "'"${PRIMARY_REGION}"'", "status": "PRIMARY_SUCCESS"}' > ./build_out/deployment.json
    echo "[SUCCESS] Primary region deployment completed."
fi

echo -e "\n--> Final Deployment State Manifest:"
cat ./build_out/deployment.json
'''

job.getBuildersList().clear()
job.getBuildersList().add(new Shell(fallbackShellScript))
job.save()

println "LESSON7_HOURS7_8_PROVISIONED"
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

sleep 4

curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson7-parallel-demo/${NEXT_BUILD_NUM}/consoleText"


***********************************************************


Capstone Objectives

Synthesize all concepts learned in Hours 1–8 by provisioning and executing a self-healing parallel processing engine named lesson7-capstone-pipeline.
Laboratory Specifications

	Provisioning: Deployed headlessly via Groovy using /scriptText.
    
	Parameters:
	SERVICE_COUNT: String parameter (Default: 3).
	SIMULATE_FAILURE: String parameter (Default: false).
	Execution Logic (set -euo pipefail):
	Process Cleanup Trap: Bind an EXIT trap to kill lingering background subshell PIDs.
	Parallel Processing: Launch ${SERVICE_COUNT} background subshell workers in parallel.
	Transient Retries: Each worker attempts service synchronization using a retry loop (3 attempts maximum). If SIMULATE_FAILURE=true, worker 2 exceeds its retry limit.
	Timeout Watchdog: Run a parallel watchdog process to ensure no subshell exceeds a 5-second runtime window.
	Fallback Routing: If any worker subshell fails completely, trigger a global fallback recovery routine that writes an emergency status file (./build_out/emergency.json) and flags the build appropriately.
REST Workspace Verification: Query and display the generated status manifest over HTTP via /ws/.




Extension Challenge: Multithreaded Log Aggregator & Health Auditing

•	Prerequisites: Completion of Lesson 7 Capstone (lesson7-capstone-pipeline).
•	Objective: Provision an independent auditing job named lesson7-audit-aggregator that launches parallel worker processes, captures individual stdout/stderr log files in real-time, generates an aggregated MD5 integrity audit manifest, and exposes all outputs over the REST API.

Challenge Requirements
1.	Job Setup:
o	Job Name: lesson7-audit-aggregator
o	Type: FreeStyleProject
o	Parameters:
	WORKER_COUNT: String parameter (Default: 4)
	SIMULATE_LATENCY: String parameter (Default: true)
2.	Execution Script (set -euo pipefail):
o	Signal Handler Trap: Register an EXIT signal trap that executes cleanup() to kill any dangling subshell PIDs stored in a Bash array.
o	Parallel Execution & File Redirection: Spawns ${WORKER_COUNT} parallel subshells. Each worker:
	Writes stdout logs directly to ./logs/worker_${ID}.log.
	Simulates work (if SIMULATE_LATENCY=true, sleeps for a random duration between 1 and 3 seconds).
	Writes an individual status JSON payload to ./logs/worker_${ID}.json:
JSON
{
  "worker_id": "<ID>",
  "pid": "<PID>",
  "status": "COMPLETED"
}
o	Synchronization & Failure Gate:
	Use a wait loop across all stored PIDs to confirm clean completion.
	If any subshell fails, log the failing PID to stderr and flag the pipeline.
o	Aggregated Manifest Generation:
	Merge all individual JSON payloads into a single runtime manifest: ./build_out/audit_summary.json.
	Compute an MD5 checksum of ./build_out/audit_summary.json and save it to ./build_out/audit_summary.json.md5.
o	Console Output: Print the aggregated summary manifest and the MD5 hash digest.











