# 1. Create the worker shell payload
cat << 'EOF' > worker_payload.sh
#!/usr/bin/env bash
set -euo pipefail

echo "=========================================================="
echo "  LESSON 9 WORKER: AUDIT & METADATA GENERATOR"
echo "=========================================================="

TARGET_ENV="${TARGET_ENV:-staging}"
REQUEST_ID="${REQUEST_ID:-unknown}"
FORCE_FAIL="${FORCE_FAIL:-false}"

echo "--> Environment : ${TARGET_ENV}"
echo "--> Request ID  : ${REQUEST_ID}"
echo "--> Force Fail  : ${FORCE_FAIL}"
echo "--> Node Name   : ${NODE_NAME:-built-in}"

if [ "${FORCE_FAIL}" = "true" ]; then
    echo "[ERROR] Simulated failure triggered via FORCE_FAIL=true!"
    exit 1
fi

# Generate structured JSON report
cat << JSON_EOF > audit_report.json
{
  "request_id": "${REQUEST_ID}",
  "environment": "${TARGET_ENV}",
  "worker_build": "${BUILD_NUMBER}",
  "executed_on": "${NODE_NAME:-built-in}",
  "checks_passed": true
}
JSON_EOF

echo "[SUCCESS] audit_report.json generated in ${WORKSPACE}"
EOF

B64_WORKER=$(base64 -w 0 worker_payload.sh 2>/dev/null || base64 worker_payload.sh | tr -d '\n')
rm -f worker_payload.sh

********************************************

# 2. Create the idempotent Groovy provisioner
cat << EOF > provision_lesson9_worker.groovy
import jenkins.model.Jenkins
import hudson.model.FreeStyleProject
import hudson.model.ParametersDefinitionProperty
import hudson.model.StringParameterDefinition
import hudson.model.ChoiceParameterDefinition
import hudson.model.BooleanParameterDefinition
import hudson.model.labels.LabelAtom
import hudson.tasks.Shell
import hudson.tasks.ArtifactArchiver

def instance = Jenkins.get()
def jobName = 'lesson9-audit-worker'

// Idempotent lookup: reuse existing project to preserve build history
def job = instance.getItem(jobName)
if (job == null) {
    job = instance.createProject(FreeStyleProject, jobName)
    println "CREATED_NEW_JOB: \${jobName}"
} else {
    println "UPDATED_EXISTING_JOB: \${jobName}"
}

// Explicit LabelAtom binding (safe on fresh installs)
job.setAssignedLabel(new LabelAtom('built-in'))

// Reset and define parameters cleanly
job.removeProperty(ParametersDefinitionProperty.class)
def envParam = new ChoiceParameterDefinition('TARGET_ENV', ['dev', 'staging', 'prod'] as String[], 'Target deployment environment')
def reqParam = new StringParameterDefinition('REQUEST_ID', 'REQ-000', 'Tracking ID from upstream orchestrator')
def failParam = new BooleanParameterDefinition('FORCE_FAIL', false, 'Simulate a worker crash for testing')
job.addProperty(new ParametersDefinitionProperty(envParam, reqParam, failParam))

// Replace build steps cleanly without duplication
String decodedScript = new String("${B64_WORKER}".decodeBase64())
job.getBuildersList().clear()
job.getBuildersList().add(new Shell(decodedScript))

// Replace post-build artifact archiver
job.getPublishersList().clear()
job.getPublishersList().add(new ArtifactArchiver('audit_report.json'))

job.save()
println "LESSON9_WORKER_READY"
EOF

# 3. Deploy via REST scriptText
curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:${API_TOKEN}" \
  --data-urlencode "script@provision_lesson9_worker.groovy"

rm -f provision_lesson9_worker.groovy


*********************************

curl -s -u "admin:${API_TOKEN}" "http://localhost:8080/job/lesson9-audit-worker/api/json" \
  | python3 -c '
import sys, json
data = json.load(sys.stdin)
print("Job Name        :", data.get("name"))
print("Label Expression:", data.get("labelExpression"))
print("Next Build #    :", data.get("nextBuildNumber"))
'





# 1. Create the orchestrator shell script
cat << 'EOF' > orchestrator_payload.sh
#!/usr/bin/env bash
set -euo pipefail

JENKINS_URL="http://localhost:8080"
CHILD_JOB="lesson9-audit-worker"
ENV_CHOICE="${TARGET_ENV:-staging}"
SIMULATE_FAIL="${FORCE_FAIL:-false}"
TRACKING_ID="ORD-${BUILD_NUMBER}-$(date +%s)"

echo "=========================================================="
echo "  LESSON 9 ORCHESTRATOR (BUILD #${BUILD_NUMBER})"
echo "=========================================================="

# 1. Obtain CSRF Crumb cleanly
CRUMB=$(curl -s -u "admin:${API_TOKEN}" \
  "${JENKINS_URL}/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,\":\",//crumb)" 2>/dev/null || true)

# 2. Trigger parameterized child job and capture response headers
echo "--> Triggering '${CHILD_JOB}' [ENV=${ENV_CHOICE}, FAIL=${SIMULATE_FAIL}, ID=${TRACKING_ID}]..."
HEADER_FILE="/tmp/headers_${BUILD_NUMBER}.txt"

TRIGGER_URL="${JENKINS_URL}/job/${CHILD_JOB}/buildWithParameters?TARGET_ENV=${ENV_CHOICE}&REQUEST_ID=${TRACKING_ID}&FORCE_FAIL=${SIMULATE_FAIL}"

if [ -n "${CRUMB}" ]; then
    curl -s -D "${HEADER_FILE}" -X POST -H "${CRUMB}" -u "admin:${API_TOKEN}" "${TRIGGER_URL}" > /dev/null
else
    curl -s -D "${HEADER_FILE}" -X POST -u "admin:${API_TOKEN}" "${TRIGGER_URL}" > /dev/null
fi

HTTP_STATUS=$(grep -i "^HTTP/" "${HEADER_FILE}" | tail -n 1 | awk '{print $2}' | tr -d '\r\n')
QUEUE_URL=$(grep -i "^Location:" "${HEADER_FILE}" | awk '{print $2}' | tr -d '\r\n')
rm -f "${HEADER_FILE}"

if [ "${HTTP_STATUS}" != "201" ] || [ -z "${QUEUE_URL}" ]; then
    echo "[ERROR] Trigger failed! HTTP Status: ${HTTP_STATUS}"
    exit 1
fi

echo "--> Queue Item Created: ${QUEUE_URL}"

# 3. Poll Queue State Machine until executable number is assigned
CHILD_BUILD_NUM=""
WAIT_SECONDS=0
MAX_QUEUE_WAIT=45

while [ -z "${CHILD_BUILD_NUM}" ]; do
    if [ "${WAIT_SECONDS}" -ge "${MAX_QUEUE_WAIT}" ]; then
        echo "[ERROR] Timed out waiting in queue after ${MAX_QUEUE_WAIT}s."
        exit 1
    fi

    QUEUE_JSON=$(curl -s -u "admin:${API_TOKEN}" "${QUEUE_URL}api/json")

    # Check if item was cancelled while queued
    IS_CANCELLED=$(echo "${QUEUE_JSON}" | python3 -c 'import sys, json; print(json.load(sys.stdin).get("cancelled", False))' 2>/dev/null || echo "False")
    if [ "${IS_CANCELLED}" = "True" ]; then
        echo "[ERROR] Downstream job was cancelled while in queue!"
        exit 1
    fi

    CHILD_BUILD_NUM=$(echo "${QUEUE_JSON}" | python3 -c 'import sys, json; exe=json.load(sys.stdin).get("executable"); print(exe.get("number") if exe else "")' 2>/dev/null || true)

    if [ -z "${CHILD_BUILD_NUM}" ]; then
        sleep 1
        WAIT_SECONDS=$((WAIT_SECONDS + 1))
    fi
done

echo "--> Executor Assigned! Child Build #${CHILD_BUILD_NUM} is active."

# 4. Poll Build Execution until building == false
while true; do
    BUILD_JSON=$(curl -s -u "admin:${API_TOKEN}" "${JENKINS_URL}/job/${CHILD_JOB}/${CHILD_BUILD_NUM}/api/json")
    IS_BUILDING=$(echo "${BUILD_JSON}" | python3 -c 'import sys, json; print(json.load(sys.stdin).get("building", True))')

    if [ "${IS_BUILDING}" = "False" ]; then
        BUILD_RESULT=$(echo "${BUILD_JSON}" | python3 -c 'import sys, json; print(json.load(sys.stdin).get("result", "UNKNOWN"))')
        echo "--> Child Build #${CHILD_BUILD_NUM} Finished with Result: ${BUILD_RESULT}"
        break
    fi
    sleep 2
done

# 5. Propagate Failure if Child Did Not Succeed
if [ "${BUILD_RESULT}" != "SUCCESS" ]; then
    echo "[ERROR] Downstream job failed (${BUILD_RESULT}). Aborting parent pipeline."
    exit 1
fi

# 6. Retrieve & Validate Artifact over REST
ARTIFACT_URL="${JENKINS_URL}/job/${CHILD_JOB}/${CHILD_BUILD_NUM}/artifact/audit_report.json"
curl -s -f -u "admin:${API_TOKEN}" -o "ingested_audit.json" "${ARTIFACT_URL}"

echo "=========================================================="
echo "  VERIFIED ARTIFACT PAYLOAD (ingested_audit.json)"
echo "=========================================================="
cat ingested_audit.json
echo ""

# Verify tracking ID round-trip
RETURNED_ID=$(python3 -c 'import json; print(json.load(open("ingested_audit.json")).get("request_id"))')
if [ "${RETURNED_ID}" != "${TRACKING_ID}" ]; then
    echo "[ERROR] Artifact mismatch! Expected ${TRACKING_ID}, got ${RETURNED_ID}"
    exit 1
fi

echo "[SUCCESS] Round-trip correlation ID (${RETURNED_ID}) verified!"
EOF



# Inject API_TOKEN safely into payload before encoding
sed -i "s/\${API_TOKEN}/${API_TOKEN}/g" orchestrator_payload.sh
B64_ORCH=$(base64 -w 0 orchestrator_payload.sh 2>/dev/null || base64 orchestrator_payload.sh | tr -d '\n')
rm -f orchestrator_payload.sh



*******************************


# 2. Provision Parent Job via Groovy
cat << EOF > provision_lesson9_parent.groovy
import jenkins.model.Jenkins
import hudson.model.FreeStyleProject
import hudson.model.ParametersDefinitionProperty
import hudson.model.ChoiceParameterDefinition
import hudson.model.BooleanParameterDefinition
import hudson.model.labels.LabelAtom
import hudson.tasks.Shell

def instance = Jenkins.get()
def jobName = 'lesson9-orchestrator'

def job = instance.getItem(jobName) ?: instance.createProject(FreeStyleProject, jobName)
job.setAssignedLabel(new LabelAtom('built-in'))

job.removeProperty(ParametersDefinitionProperty.class)
def envParam = new ChoiceParameterDefinition('TARGET_ENV', ['staging', 'prod'] as String[], 'Environment to audit')
def failParam = new BooleanParameterDefinition('FORCE_FAIL', false, 'Pass failure flag to downstream worker')
job.addProperty(new ParametersDefinitionProperty(envParam, failParam))

String decodedScript = new String("${B64_ORCH}".decodeBase64())
job.getBuildersList().clear()
job.getBuildersList().add(new Shell(decodedScript))

job.save()
println "LESSON9_PARENT_READY"
EOF

curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:${API_TOKEN}" \
  --data-urlencode "script@provision_lesson9_parent.groovy"

rm -f provision_lesson9_parent.groovy

*************************************************

JENKINS_URL="http://localhost:8080"
JOB_NAME="lesson9-audit-worker"
TRACKING_ID="ORD-$(date +%s)"

echo "=========================================================="
echo "  ANGLE 3: END-TO-END CONTRACT VALIDATION"
echo "=========================================================="

# 1. Trigger with delay=0sec (skips 5s quiet period) & capture Queue URL
QUEUE_URL=$(curl -s -D - -o /dev/null -X POST \
  "${JENKINS_URL}/job/${JOB_NAME}/buildWithParameters?delay=0sec&TARGET_ENV=prod&REQUEST_ID=${TRACKING_ID}&FORCE_FAIL=false" \
  | grep -i "^Location:" | awk '{print $2}' | tr -d '\r\n')

echo "--> Dispatched REQUEST_ID=${TRACKING_ID} -> Queue: ${QUEUE_URL}"
sleep 3

# 2. Resolve exact Build Number from the Queue Item (fallback to lastBuild if already evicted)
BUILD_NUM=$(curl -s "${QUEUE_URL}api/json?tree=executable[number]" \
  | sed -n 's/.*"number":\([0-9]*\).*/\1/p')
[ -z "${BUILD_NUM}" ] && BUILD_NUM=$(curl -s "${JENKINS_URL}/job/${JOB_NAME}/lastBuild/api/json?tree=number" \
  | sed -n 's/.*"number":\([0-9]*\).*/\1/p')

RESULT=$(curl -s "${JENKINS_URL}/job/${JOB_NAME}/${BUILD_NUM}/api/json?tree=result" \
  | sed -n 's/.*"result":"\([^"]*\)".*/\1/p')
echo "--> Exact Build #${BUILD_NUM} Result: ${RESULT}"

# 3. Download artifact from that exact build and verify contract
curl -s -f -o "verified_audit.json" "${JENKINS_URL}/job/${JOB_NAME}/${BUILD_NUM}/artifact/audit_report.json"

echo -e "\n=== DOWNLOADED ARTIFACT (verified_audit.json) ==="
cat verified_audit.json
echo ""

RETURNED_ID=$(sed -n 's/.*"request_id"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p' verified_audit.json)

if [ "${RETURNED_ID}" = "${TRACKING_ID}" ]; then
    echo "[VERIFIED] Round-trip contract passed! Sent '${TRACKING_ID}' == Received '${RETURNED_ID}'."
else
    echo "[FAILED] Contract mismatch! Expected '${TRACKING_ID}', got '${RETURNED_ID}'."
fi

*********************************

PRACTICE

Task Set 1: Guided Reinforcement (Core Mechanics)
Complete these three tasks using the jobs created in Lesson 9:
1.	Idempotency Verification:
o	Run snippet (provision_lesson9_worker.groovy) and verify that the Groovy output prints UPDATED_EXISTING_JOB: lesson9-audit-worker instead of CREATED_NEW_JOB.
o	Query http://localhost:8080/job/lesson9-audit-worker/api/json and confirm that nextBuildNumber is still 3 (meaning historical builds #1 and #2 were preserved).
2.	Tree-Filtered Artifact Inspection:
o	Using the ?tree= REST query technique, write a single curl + python3 one-liner that queries lesson9-audit-worker and prints the number, result, and artifacts[relativePath] for all historical builds.
o	Expected observation: Build #1 lists audit_report.json under artifacts, whereas Build #2 has an empty artifacts list ([]).
3.	Environment Parameter Routing:
o	Trigger lesson9-orchestrator with TARGET_ENV=dev and FORCE_FAIL=false.
o	Confirm in the console output that the downloaded ingested_audit.json contains "environment": "dev".


Task Set 2: Applied Engineering Challenge (Extending the Pattern)
Implement two new production requirements without installing any Jenkins plugins:
1.	Challenge A — Artifact Checksum Verification (SHA-256):
o	Update worker_payload.sh so that after generating audit_report.json, it computes its SHA-256 hash and writes it to a second file named audit_report.sha256 (sha256sum audit_report.json > audit_report.sha256).
o	Update ArtifactArchiver to archive both files (new ArtifactArchiver('audit_report.json,audit_report.sha256')).
o	Update orchestrator_payload.sh to download both files via REST and run sha256sum -c audit_report.sha256 before reading the JSON. If the checksum fails, abort with exit 1.
2.	Challenge B — Multi-Environment Fan-Out Loop:
o	Write an external Bash script that loops over for ENV in dev staging prod; do ... done, triggers lesson9-orchestrator sequentially for each environment, and prints a final summary table showing the parent build number, environment name, and final build result (SUCCESS/FAILURE).


Challenge 1: The Flash-Sale Burst & Queue Isolation Incident
Scenario
It is 5 minutes before a major flash sale. Two automated upstream services—the Checkout Service (TARGET_ENV=prod) and the Staging Smoke-Test Bot (TARGET_ENV=staging)—trigger lesson9-audit-worker within the exact same second.
Yesterday, during a similar traffic spike, your team used /lastBuild/api/json to poll results. Because both triggers arrived almost simultaneously, the Staging Smoke-Test Bot accidentally read the Checkout Service's audit_report.json artifact, causing a false deployment gate alarm.
Your Mission
Write a pure-Bash verification script (challenge5_burst_isolation.sh) that proves your orchestrator prevents cross-talk under concurrent burst load:
1.	Simultaneous Dispatch: Fire two buildWithParameters?delay=0sec requests back-to-back (without sleeping between them):
o	Request A: TARGET_ENV=staging, REQUEST_ID=BURST-STG-<epoch>, FORCE_FAIL=false
o	Request B: TARGET_ENV=prod, REQUEST_ID=BURST-PRD-<epoch>, FORCE_FAIL=false
2.	Independent Queue Pinning: Capture both Location: headers (QUEUE_A and QUEUE_B) and resolve each one to its own distinct build number (BUILD_A and BUILD_B), including the 404 fast-eviction fallback if a queue item completes before polling.
3.	Zero Cross-Talk Assertion: Download audit_report.json from both BUILD_A and BUILD_B into separate local files (audit_A.json and audit_B.json), and programmatically assert:
o	BUILD_A is not equal to BUILD_B
o	audit_A.json contains BURST-STG-<epoch> and "environment": "staging"
o	audit_B.json contains BURST-PRD-<epoch> and "environment": "prod"


Challenge 2: The Silent Latency Degradation & Health Gate
Scenario
Your release pipeline enforces a strict Service Level Agreement (SLA): lesson9-audit-worker must complete in under 8,000 ms (8 seconds), and at least 3 out of the last 5 builds must be healthy (SUCCESS) before a production promotion is permitted.
Earlier this week, an infrastructure slowdown caused builds to still exit 0 (SUCCESS), but take 25 seconds—causing downstream API timeouts. Relying only on "result":"SUCCESS" missed the regression completely.
Your Mission
Write a pure-Bash pre-release health & SLA gate (challenge6_sla_audit.sh) that queries Jenkins' REST telemetry without downloading full build histories:
1.	Dispatch & Measure: Trigger a fresh TARGET_ENV=prod build (REQUEST_ID=SLA-<epoch>, delay=0sec), resolve its build number via Location:, and wait for completion.
2.	SLA Latency Check: Query /${BUILD_NUM}/api/json?tree=number,result,duration and assert two conditions:
o	result equals SUCCESS
o	duration (in milliseconds) is strictly less than 8000 ([ "${DURATION}" -lt 8000 ])
3.	Sliding-Window Health Audit: Query /job/lesson9-audit-worker/api/json?tree=builds[number,result,duration]{0,5} (using Jenkins' {0,5} server-side slice so only the 5 most recent runs are transferred). Count how many of those 5 runs ended in SUCCESS vs. FAILURE, and print a clear PROMOTION APPROVED or PROMOTION BLOCKED decision based on whether SUCCESS >= 3 and the SLA passed.










