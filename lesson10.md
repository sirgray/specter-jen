verify_stage() {
    local ENV_NAME="$1"
    local CORR_ID="REL-$(date +%s)-${ENV_NAME}"

    local Q_URL
    Q_URL=$(curl -s -D - -o /dev/null -X POST \
      "http://localhost:8080/job/lesson9-audit-worker/buildWithParameters?delay=0sec&TARGET_ENV=${ENV_NAME}&REQUEST_ID=${CORR_ID}&FORCE_FAIL=false" \
      | grep -i "^Location:" | awk '{print $2}' | tr -d '\r\n')

    sleep 3

    local B_NUM
    B_NUM=$(curl -s "${Q_URL}api/json?tree=executable[number]" | sed -n 's/.*"number":\([0-9]*\).*/\1/p')
    [ -z "${B_NUM}" ] && B_NUM=$(curl -s "http://localhost:8080/job/lesson9-audit-worker/lastBuild/api/json?tree=number" | sed -n 's/.*"number":\([0-9]*\).*/\1/p')

    local RES
    RES=$(curl -s "http://localhost:8080/job/lesson9-audit-worker/${B_NUM}/api/json?tree=result" | sed -n 's/.*"result":"\([^"]*\)".*/\1/p')
    [ "${RES}" != "SUCCESS" ] && return 1

    curl -s -f -o "out_${ENV_NAME}.json" "http://localhost:8080/job/lesson9-audit-worker/${B_NUM}/artifact/audit_report.json" || return 1
    grep -q "\"request_id\"[[:space:]]*:[[:space:]]*\"${CORR_ID}\"" "out_${ENV_NAME}.json" || return 1
    grep -q "\"environment\"[[:space:]]*:[[:space:]]*\"${ENV_NAME}\"" "out_${ENV_NAME}.json" || return 1

    return 0
}

verify_stage "staging" && verify_stage "prod"




ERROR_LINE=$(curl -s "http://localhost:8080/job/lesson9-audit-worker/${BUILD_NUM}/consoleText" \
  | grep -E "\[ERROR\]|FATAL|Exception" | head -n 1)

echo "[WARN] Attempt ${ATTEMPT} failed on Build #${BUILD_NUM}: ${ERROR_LINE:-Unknown error}"
sleep 2


NODE_JSON=$(curl -s -g "http://localhost:8080/computer/api/json?tree=computer[displayName,offline,numExecutors,idle]")
echo "${NODE_JSON}"

***********************************************


Independent Task Set 1: Declarative Job Provisioning & Idempotency (Lessons 1–4)

Objective
Provision a brand-new Freestyle job named lab-provision-check purely via Groovy over HTTP (POST /scriptText), configure it with string/boolean parameters and a workspace artifact publisher, and prove that re-running your provisioning logic is idempotent (updates in place without throwing duplicate-item exceptions or wiping build history).
Requirements
Write a standalone Bash script (task1_provision.sh) using only curl, grep, sed, and awk (no API tokens, no plugins) that performs the following:
1.	Idempotent Groovy Provisioning (POST /scriptText):
o	Check if lab-provision-check already exists in Jenkins.get(). If it does not exist, create it as a FreeStyleProject; if it already exists, retrieve and update it in place.
o	Attach a ParametersDefinitionProperty containing two parameters:
	APP_VERSION (StringParameterDefinition, default "1.0.0")
	DRY_RUN (BooleanParameterDefinition, default false)
o	Configure a Shell build step that:
	Cleans any old manifest.txt (rm -f manifest.txt).
	Writes version=${APP_VERSION} and dry_run=${DRY_RUN} into manifest.txt.
o	Configure an ArtifactArchiver for manifest.txt with onlyIfSuccessful = true.
o	Call job.save() and print confirmation from Groovy.
2.	Idempotency Proof:
o	Execute your provisioning curl payload twice in a row.
o	Verify via REST (GET /job/lab-provision-check/api/json?tree=name,property[parameterDefinitions[name,defaultParameterValue[value]]]) that the job exists once and both parameters reflect their configured defaults.





Independent Task Set 2: Queue Lifecycle, Parameter Injection & Artifact Extraction (Lessons 5–8)

Objective
Build a self-contained execution and verification runner against a freshly provisioned job (lab-runner-check) that exercises the full asynchronous Jenkins execution lifecycle without relying on /lastBuild/ shortcuts.
Requirements
Write a standalone Bash script (task2_lifecycle.sh) that:
1.	Self-Contained Setup: Creates a minimal Freestyle job lab-runner-check via /scriptText that accepts a TOKEN string parameter, sleeps 2 seconds, writes TOKEN=<value> to token.out, and archives token.out.
2.	Immediate Queue Dispatch: Triggers lab-runner-check with delay=0sec and a dynamically generated TOKEN=TOK-$(date +%s) via buildWithParameters.
3.	Deterministic Queue-to-Build Pinning:
o	Captures the Location: HTTP response header (QUEUE_URL).
o	Polls ${QUEUE_URL}api/json?tree=executable[number] until a concrete build number is assigned (including a 404 fast-eviction fallback).
4.	Polling Until Completion (building == false):
o	Queries /job/lab-runner-check/${BUILD_NUM}/api/json?tree=building,result in a loop until "building":false.
5.	Round-Trip Verification:
o	Downloads /job/lab-runner-check/${BUILD_NUM}/artifact/token.out using curl -f and asserts that the file content matches the exact TOKEN string sent in Step 2.



Unguided Challenge 1: The Self-Cleaning Drift-Repair Provisioner

Scenario
Your team manages a shared Jenkins controller where developers occasionally log into the Web UI and manually alter job configurations—changing default parameter values, breaking build shell commands, or disabling onlyIfSuccessful on artifact archivers.
You need an automated Drift-Repair Script (challenge1_drift_repair.sh) that detects and heals configuration drift automatically without deleting existing build numbers.
Acceptance Criteria
1.	Bootstrap: Your script must first provision a job named drift-target-job via /scriptText with:
o	Parameter ENV_TIER (default "staging")
o	Shell step writing tier=${ENV_TIER} to tier.txt
o	ArtifactArchiver archiving tier.txt (onlyIfSuccessful = true)
2.	Simulate Manual UI Drift: In the same script, send a second Groovy snippet to /scriptText that simulates a developer corrupting the job—changing ENV_TIER's default to "broken-dev" and setting onlyIfSuccessful = false.
3.	Automated Audit & Self-Heal:
o	Query drift-target-job's REST API (-sg + ?tree=) to detect that ENV_TIER's default is no longer "staging".
o	Automatically re-run your idempotent Groovy desired-state block to restore ENV_TIER="staging" and onlyIfSuccessful = true.
o	Trigger a default build (buildWithParameters?delay=0sec), resolve its build number via Location:, and prove via the downloaded tier.txt artifact that the healed job once again outputs tier=staging.



Unguided Challenge 2: The Multi-Job Fan-Out & Aggregation Controller

Scenario
A microservice release requires running two completely independent validation jobs (svc-auth-check and svc-billing-check) concurrently, waiting for both to finish, and generating a unified release verdict only if both pass and produce valid artifacts.
Acceptance Criteria
1.	Zero-Dependency Setup: Your script (challenge2_fanout.sh) must create both svc-auth-check and svc-billing-check via /scriptText from scratch:
o	svc-auth-check accepts CORR_ID, sleeps 2 seconds, writes AUTH_OK:${CORR_ID} to auth.txt, and archives auth.txt.
o	svc-billing-check accepts CORR_ID, sleeps 3 seconds, writes BILLING_OK:${CORR_ID} to billing.txt, and archives billing.txt.
2.	Concurrent Fan-Out: Generate a single CORR_ID=FAN-$(date +%s) and dispatch both jobs back-to-back with delay=0sec, capturing QUEUE_AUTH and QUEUE_BILLING independently.
3.	Dual Queue Resolution & Polling: Resolve both queue URLs to their exact build numbers (BUILD_AUTH and BUILD_BILLING) and wait until both builds report "building":false and "result":"SUCCESS".
4.	Unified Contract Report: Download both auth.txt and billing.txt, verify that both contain the exact CORR_ID, and print a consolidated summary report listing both build numbers, durations (duration in ms via ?tree=duration), and verified artifact contents.


