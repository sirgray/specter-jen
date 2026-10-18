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

# Fallback string parsing (grep + cut) if jq is not present
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

println "LESSON5_HOURS3_4_PROVISIONED"
EOF

curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:$API_TOKEN" \
  --data-urlencode "script@script.groovy"
rm script.groovy


RUN Bash:

JSON_RELEASE='{"event":"release","repo":"org/core-service","action":"published"}'

curl -X POST -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson5-webhook-demo/buildWithParameters" \
  --data-urlencode "PAYLOAD_JSON=${JSON_RELEASE}"

sleep 3

curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson5-webhook-demo/2/consoleText"




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







