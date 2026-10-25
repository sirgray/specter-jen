# 1. Create local Groovy provisioning script
cat << 'EOF' > script.groovy
import jenkins.model.*
import hudson.model.*
import hudson.tasks.Shell

def instance = Jenkins.get()
def jobName = 'lesson6-artifact-demo'

def job = instance.getItem(jobName) ?: instance.createProject(FreeStyleProject, jobName)

// Register string parameter for target build environment
def envParam = new StringParameterDefinition('BUILD_ENV', 'STAGING', 'Target Environment')
job.removeProperty(ParametersDefinitionProperty.class)
job.addProperty(new ParametersDefinitionProperty(envParam))

def shellScript = '''#!/usr/bin/env bash
set -euo pipefail

echo "=========================================================="
echo "  HOURS 1-2: WORKSPACE ARTIFACT GENERATION"
echo "=========================================================="

echo "--> Step 1: Creating build workspace directories..."
mkdir -p ./output/reports

echo "--> Step 2: Generating build manifest artifact..."
cat << MANIFEST > ./output/build_manifest.json
{
  "job_name": "${JOB_NAME}",
  "build_number": "${BUILD_NUMBER}",
  "environment": "${BUILD_ENV}",
  "timestamp": "$(date -u +'%Y-%m-%dT%H:%M:%SZ')",
  "status": "SUCCESS"
}
MANIFEST

echo "--> Step 3: Generating execution log report..."
echo "[INFO] Build started for environment ${BUILD_ENV}" > ./output/reports/execution.log
echo "[INFO] Artifacts generated successfully at $(date)" >> ./output/reports/execution.log

echo -e "\n--> Step 4: Listing generated workspace files:"
ls -la ./output/
ls -la ./output/reports/
'''

job.getBuildersList().clear()
job.getBuildersList().add(new Shell(shellScript))
job.save()

println "LESSON6_HOURS1_2_PROVISIONED"
EOF


# 2. Deploy to Jenkins via /scriptText
curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:$API_TOKEN" \
  --data-urlencode "script@script.groovy"
rm script.groovy


BASH

# 1. Fetch upcoming build number dynamically
NEXT_BUILD_NUM=$(curl -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson6-artifact-demo/api/json" \
  | grep -o '"nextBuildNumber":[^,]*' | cut -d':' -f2)

echo "--> Upcoming Build Number: ${NEXT_BUILD_NUM}"

# 2. Trigger build with BUILD_ENV=PRODUCTION
curl -X POST -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson6-artifact-demo/buildWithParameters?BUILD_ENV=PRODUCTION"

sleep 3

# 3. Stream console output using dynamic build number
echo -e "\n=========================================================="
echo "  CONSOLE LOG OUTPUT (BUILD #${NEXT_BUILD_NUM})"
echo "=========================================================="
curl -s -u "admin:$API_TOKEN" http://localhost:8080/job/lesson6-artifact-demo/${NEXT_BUILD_NUM}/consoleText


****************************************

cat << 'EOF' > script.groovy
import jenkins.model.*
import hudson.model.*
import hudson.tasks.Shell

def instance = Jenkins.get()
def jobName = 'lesson6-artifact-demo'
def job = instance.getItem(jobName)

def validationShellScript = '''#!/usr/bin/env bash
set -euo pipefail

# 1. Setup Cleanup Trap
teardown() {
    echo -e "\n[CLEANUP] Post-execution state check finished."
}
trap teardown EXIT

echo "=========================================================="
echo "  HOURS 3-4: ARTIFACT INTEGRITY & FAILURE GATES"
echo "=========================================================="

MANIFEST_PATH="./output/build_manifest.json"

echo "--> Step 1: Generating Build Manifest..."
mkdir -p ./output
cat << 'MANIFEST' > ./output/build_manifest.json
{
  "job_name": "lesson6-artifact-demo",
  "status": "COMPLETED",
  "checksum_verified": true
}
MANIFEST

echo "--> Step 2: Validating file existence and size..."
if [ ! -f "${MANIFEST_PATH}" ]; then
    echo "[ERROR] Manifest file '${MANIFEST_PATH}' was not created!" >&2
    exit 1
fi

if [ ! -s "${MANIFEST_PATH}" ]; then
    echo "[ERROR] Manifest file '${MANIFEST_PATH}' is empty (0 bytes)!" >&2
    exit 1
fi

echo "[CHECK PASSED] File exists and is non-empty."

echo "--> Step 3: Validating content integrity..."
if ! grep -q '"status": "COMPLETED"' "${MANIFEST_PATH}"; then
    echo "[ERROR] Manifest status validation failed!" >&2
    exit 1
fi

echo "[CHECK PASSED] Manifest status string verified."
echo "[SUCCESS] All workspace artifacts passed integrity gates."
'''

job.getBuildersList().clear()
job.getBuildersList().add(new Shell(validationShellScript))
job.save()

println "LESSON6_HOURS3_4_PROVISIONED"
EOF

BASH

curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:$API_TOKEN" \
  --data-urlencode "script@script.groovy"
rm script.groovy



TASK - DEPLOY AND GET OUTPUT













*******************************************************

# Download the entire output directory as output.zip
curl -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson6-artifact-demo/ws/output/*zip*/output.zip" \
  -o output.zip

# Unzip and inspect locally
unzip -q output.zip
ls -la output/

echo -e "\n\n--> 2. Downloading Manifest Artifact directly over HTTP:"
curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson6-artifact-demo/ws/output/build_manifest.json"







