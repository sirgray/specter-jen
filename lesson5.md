# 1. Create local Groovy script
cat << 'EOF' > script.groovy
import jenkins.model.*
import hudson.model.*
import hudson.tasks.Shell

def instance = Jenkins.get()
def jobName = 'lesson5-webhook-demo'

def job = instance.getItem(jobName) ?: instance.createProject(FreeStyleProject, jobName)

// Define parameter string to hold raw JSON payload
def payloadParam = new StringParameterDefinition(
    'PAYLOAD_JSON', 
    '{"event":"default"}', 
    'Raw JSON payload delivered by inbound webhook'
)

job.removeProperty(ParametersDefinitionProperty.class)
job.addProperty(new ParametersDefinitionProperty(payloadParam))

def shellScript = '''#!/usr/bin/env bash
set -euo pipefail

echo "=========================================================="
echo "  HOURS 1-2: WEBHOOK ENDPOINT INGESTION"
echo "=========================================================="
echo "Received Raw Payload:"
echo "${PAYLOAD_JSON}"
echo "=========================================================="
'''

job.getBuildersList().clear()
job.getBuildersList().add(new Shell(shellScript))
job.save()

println "LESSON5_HOURS1_2_PROVISIONED"
EOF

# 2. Deploy to Jenkins via /scriptText
curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:$API_TOKEN" \
  --data-urlencode "script@script.groovy"
rm script.groovy




RUN BASH:


JSON_DATA='{"event":"push","ref":"refs/heads/main","actor":"devops-admin"}'

echo "--> Sending Webhook Payload..."
curl -X POST -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson5-webhook-demo/buildWithParameters" \
  --data-urlencode "PAYLOAD_JSON=${JSON_DATA}"

sleep 3

echo -e "\n--> Console Output:"
curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson5-webhook-demo/1/consoleText"

**********************************************






# 1. Re-provision lesson5-webhook-demo with the Hours 3-4 Parsing Script
cat << 'EOF' > script.groovy
import jenkins.model.*
import hudson.model.*
import hudson.tasks.Shell

def instance = Jenkins.get()
def jobName = 'lesson5-webhook-demo'
def job = instance.getItem(jobName)

def parsingShellScript = '''#!/usr/bin/env bash
set -euo pipefail

echo "=========================================================="
echo "  HOURS 3-4: JSON EXTRACTION AND PARSING"
echo "=========================================================="

# Extract fields safely using grep and cut
EVENT_TYPE=$(echo "${PAYLOAD_JSON}" | grep -o '"event":"[^"]*' | cut -d'"' -f4 || echo "unknown")
REPOSITORY=$(echo "${PAYLOAD_JSON}" | grep -o '"repo":"[^"]*' | cut -d'"' -f4 || echo "unknown")
ACTION=$(echo "${PAYLOAD_JSON}" | grep -o '"action":"[^"]*' | cut -d'"' -f4 || echo "none")

echo "Parsed Event      : ${EVENT_TYPE}"
echo "Parsed Repository : ${REPOSITORY}"
echo "Parsed Action     : ${ACTION}"

echo -e "\n--> Evaluating Workflow Routing:"
if [ "${EVENT_TYPE}" = "release" ]; then
    echo "[ACTION] Production release detected. Initializing artifact deployment..."
elif [ "${EVENT_TYPE}" = "push" ]; then
    echo "[ACTION] Source code update detected. Running unit testing suite..."
else
    echo "[INFO] Event '${EVENT_TYPE}' requires no secondary tasks."
fi
'''

job.getBuildersList().clear()
job.getBuildersList().add(new Shell(parsingShellScript))
job.save()

println "LESSON5_HOURS3_4_REPROVISIONED"
EOF

# Deploy Groovy script to Jenkins
curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:$API_TOKEN" \
  --data-urlencode "script@script.groovy"
rm script.groovy

# 2. Trigger Build #3 with the Release Payload
JSON_RELEASE='{"event":"release","repo":"org/core-service","action":"published"}'

curl -X POST -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson5-webhook-demo/buildWithParameters" \
  --data-urlencode "PAYLOAD_JSON=${JSON_RELEASE}"

echo "Waiting 3 seconds..."
sleep 3

# Fetch the updated console log for Build #3
echo -e "\n=========================================================="
echo "  CONSOLE LOG OUTPUT (BUILD #3)"
echo "=========================================================="
curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson5-webhook-demo/3/consoleText"


NEXT BUILD ID DYNAMIC:

# 1. Get the upcoming build number from Jenkins API
NEXT_BUILD_NUM=$(curl -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson5-webhook-demo/api/json" \
  | grep -o '"nextBuildNumber":[^,]*' | cut -d':' -f2)

echo "--> Next Build ID will be: ${NEXT_BUILD_NUM}"

# 2. Trigger the build
curl -X POST -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson5-webhook-demo/buildWithParameters" \
  --data-urlencode 'PAYLOAD_JSON={"event":"release","repo":"org/core-service","action":"published"}'

sleep 3

# 3. Fetch logs dynamically using $NEXT_BUILD_NUM
echo -e "\n=========================================================="
echo "  CONSOLE LOG FOR BUILD #${NEXT_BUILD_NUM}"
echo "=========================================================="
curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson5-webhook-demo/${NEXT_BUILD_NUM}/consoleText"


*******************************************************

cat << 'EOF' > script.groovy
import jenkins.model.*
import hudson.model.*
import hudson.tasks.Shell

def instance = Jenkins.get()
def jobName = 'lesson5-webhook-demo'
def job = instance.getItem(jobName)

// Add AUTH_TOKEN definition alongside PAYLOAD_JSON
def tokenParam = new StringParameterDefinition('AUTH_TOKEN', '', 'Shared authentication token')
def payloadParam = new StringParameterDefinition('PAYLOAD_JSON', '{}', 'Webhook payload')

job.removeProperty(ParametersDefinitionProperty.class)
job.addProperty(new ParametersDefinitionProperty(tokenParam, payloadParam))

def securedShellScript = '''#!/usr/bin/env bash
set -euo pipefail

# Define post-execution cleanup trap
cleanup() {
    echo -e "\n[CLEANUP] Purging execution memory and temporary files..."
}
trap cleanup EXIT

EXPECTED_TOKEN="secure-webhook-secret-99"

echo "=========================================================="
echo "  HOURS 5-6: SECURITY & TOKEN VALIDATION"
echo "=========================================================="

echo "--> Step 1: Validating Authentication Token..."
if [ "${AUTH_TOKEN:-}" != "${EXPECTED_TOKEN}" ]; then
    echo "[SECURITY ERROR] Invalid or missing auth token! Access denied." >&2
    exit 1
fi

echo "[SUCCESS] Token verified. Access granted."

EVENT_TYPE=$(echo "${PAYLOAD_JSON}" | grep -o '"event":"[^"]*' | cut -d'"' -f4 || echo "unknown")
echo "--> Step 2: Executing task for event type: ${EVENT_TYPE}"
'''

job.getBuildersList().clear()
job.getBuildersList().add(new Shell(securedShellScript))
job.save()

println "LESSON5_HOURS5_6_PROVISIONED"
EOF

curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:$API_TOKEN" \
  --data-urlencode "script@script.groovy"
rm script.groovy


1.	Unauthorized Execution Test (Invalid Token):
Bash:

NEXT_BUILD_NUM=$(curl -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson5-webhook-demo/api/json" \
  | grep -o '"nextBuildNumber":[^,]*' | cut -d':' -f2)

curl -X POST -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson5-webhook-demo/buildWithParameters?AUTH_TOKEN=wrong-token" \
  --data-urlencode 'PAYLOAD_JSON={"event":"push"}'

sleep 3
curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson5-webhook-demo/${NEXT_BUILD_NUM}/consoleText"

2.	Authorized Execution Test (Valid Token):
Bash:

NEXT_BUILD_NUM=$(curl -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson5-webhook-demo/api/json" \
  | grep -o '"nextBuildNumber":[^,]*' | cut -d':' -f2)

curl -X POST -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson5-webhook-demo/buildWithParameters?AUTH_TOKEN=secure-webhook-secret-99" \
  --data-urlencode 'PAYLOAD_JSON={"event":"push"}'

sleep 3
curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson5-webhook-demo//${NEXT_BUILD_NUM}/consoleText"


**************************************************

•	Prerequisites: Access to the terminal environment with $API_TOKEN set.

•	Objective: Headlessly provision a webhook receiver job via /scriptText, inject authentication and JSON parsing logic, and verify both authorized and unauthorized trigger attempts via cURL.
Lab Requirements

You are tasked with building a webhook receiver job named lab-webhook-receiver.

Specifications
1.	Job Configuration:
o	Job Type: FreeStyleProject
o	Parameters:
	SECRET_KEY: String parameter (Expected value: lab-secret-123)
	EVENT_PAYLOAD: String parameter containing JSON input

2.	Shell Execution Steps (set -euo pipefail):
o	Cleanup Hook: Register a trap on EXIT that prints [LAB] Teardown complete.
o	Authentication: Validate SECRET_KEY. If it does not equal lab-secret-123, print [ERROR] Invalid secret! to stderr (>&2) and exit with status code 1.
o	Parsing: Extract action and component from EVENT_PAYLOAD.
o	Output: Print Processing action '<action>' for component '<component>'












