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

*******************************************************


cat << 'EOF' > script.groovy
import jenkins.model.*
import hudson.model.*
import hudson.tasks.Shell
import hudson.tasks.ArtifactArchiver

def instance = Jenkins.get()
def jobName = 'lesson6-artifact-demo'
def job = instance.getItem(jobName) ?: instance.createProject(FreeStyleProject, jobName)

// Raw multi-line shell script string
String checksumShellScript = '''#!/usr/bin/env bash
set -euo pipefail

echo "=========================================================="
echo "  HOURS 7-8: MD5 CHECKSUMS & ARTIFACT ARCHIVAL"
echo "=========================================================="

mkdir -p ./dist/packages

echo "--> Step 1: Building binary release artifact..."
echo "BUILD_BIN_v1.0_STABLE_DATA" > ./dist/packages/app-v1.bin

echo "--> Step 2: Generating MD5 checksum..."
md5sum ./dist/packages/app-v1.bin | awk '{print $1}' > ./dist/packages/app-v1.bin.md5

echo "--> Step 3: Verifying MD5 checksum integrity..."
EXPECTED_HASH=$(cat ./dist/packages/app-v1.bin.md5)
ACTUAL_HASH=$(md5sum ./dist/packages/app-v1.bin | awk '{print $1}')

if [ "$EXPECTED_HASH" != "$ACTUAL_HASH" ]; then
    echo "[ERROR] MD5 verification mismatch!" >&2
    exit 1
fi

echo "[SUCCESS] MD5 verification passed (${ACTUAL_HASH})."
'''

// Clear old build steps and attach shell builder
job.getBuildersList().clear()
job.getBuildersList().add(new Shell(checksumShellScript))

// Attach artifact archiver publisher post-build step
def archiver = new ArtifactArchiver('dist/packages/**/*')
archiver.setFingerprint(true)

job.getPublishersList().clear()
job.getPublishersList().add(archiver)

job.save()

println "LESSON6_HOURS7_8_PROVISIONED"
EOF

# Deploy Groovy script to Jenkins REST endpoint
curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:$API_TOKEN" \
  --data-urlencode "script@script.groovy"
rm script.groovy



Bash
# 1. Get upcoming build ID
NEXT_BUILD_NUM=$(curl -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson6-artifact-demo/api/json" \
  | sed -n 's/.*"nextBuildNumber":\([0-9]*\).*/\1/p')

if [ -z "${NEXT_BUILD_NUM}" ]; then
  NEXT_BUILD_NUM=1
fi

echo "--> Triggering Build #${NEXT_BUILD_NUM}..."

# 2. Trigger build
curl -X POST -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson6-artifact-demo/buildWithParameters"

sleep 4

# 3. Stream console output
echo "=========================================================="
echo "  CONSOLE LOG OUTPUT (BUILD #${NEXT_BUILD_NUM})"
echo "=========================================================="
curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson6-artifact-demo/${NEXT_BUILD_NUM}/consoleText"

# 4. Fetch the permanently archived MD5 file over REST
echo -e "\n=========================================================="
echo "  FETCHING ARCHIVED ARTIFACT VIA REST API"
echo "=========================================================="
curl -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson6-artifact-demo/${NEXT_BUILD_NUM}/artifact/dist/packages/app-v1.bin.md5"
echo ""


*****************************************

# 1. Generate and deploy Capstone Provisioning Script
cat << 'EOF' > capstone_provision.groovy
import jenkins.model.*
import hudson.model.*
import hudson.tasks.Shell

def instance = Jenkins.get()
def jobName = 'lesson6-capstone-pipeline'

def job = instance.getItem(jobName) ?: instance.createProject(FreeStyleProject, jobName)

def paramApp = new StringParameterDefinition('APP_NAME', 'billing-service', 'Application Name')
def paramVer = new StringParameterDefinition('RELEASE_VERSION', 'v3.1.0', 'Release Version')

job.removeProperty(ParametersDefinitionProperty.class)
job.addProperty(new ParametersDefinitionProperty(paramApp, paramVer))

def capstoneShell = '''#!/usr/bin/env bash
set -euo pipefail

teardown() {
    echo -e "\n[TEARDOWN] Workspace inspection completed."
}
trap teardown EXIT

echo "=========================================================="
echo "  HOURS 9-10: CAPSTONE ARTIFACT INSPECTION PIPELINE"
echo "=========================================================="

echo "--> Step 1: Creating distribution artifacts..."
mkdir -p ./dist

cat << MANIFEST > ./dist/release.json
{
  "app": "${APP_NAME}",
  "version": "${RELEASE_VERSION}",
  "verified": true
}
MANIFEST

echo "--> Step 2: Executing file validation gates..."
if [ ! -s "./dist/release.json" ]; then
    echo "[ERROR] Release manifest missing or empty!" >&2
    exit 1
fi

if ! grep -q '"verified": true' "./dist/release.json"; then
    echo "[ERROR] Manifest verification flag check failed!" >&2
    exit 1
fi

echo "[SUCCESS] Release manifest integrity verified."

echo -e "\n--> Step 3: Manifest Artifact Contents:"
cat ./dist/release.json
'''

job.getBuildersList().clear()
job.getBuildersList().add(new Shell(capstoneShell))
job.save()

println "LESSON6_CAPSTONE_PROVISIONED"
EOF

# 2. Deploy payload to Jenkins
curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:$API_TOKEN" \
  --data-urlencode "script@capstone_provision.groovy"
rm capstone_provision.groovy


Bash (do step by step)

# 1. Query upcoming build ID
NEXT_BUILD_NUM=$(curl -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson6-capstone-pipeline/api/json" \
  | grep -o '"nextBuildNumber":[^,]*' | cut -d':' -f2)

echo "--> Triggering Capstone Build #${NEXT_BUILD_NUM}..."

# 2. Trigger build with custom parameters
curl -X POST -s -u "admin:$API_TOKEN" \
  "http://localhost:8080/job/lesson6-capstone-pipeline/buildWithParameters?APP_NAME=payment-gateway&RELEASE_VERSION=v4.0.0"

sleep 3

# 3. Stream console output for the build
echo -e "\n=========================================================="
echo "  CAPSTONE CONSOLE LOG (BUILD #${NEXT_BUILD_NUM})"
echo "=========================================================="
curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson6-capstone-pipeline/${NEXT_BUILD_NUM}/consoleText"

# 4. Fetch the generated artifact directly from the workspace via REST API
echo -e "\n=========================================================="
echo "  FETCHING ARTIFACT VIA WORKSPACE REST API (/ws/dist/release.json)"
echo "=========================================================="
curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson6-capstone-pipeline/ws/dist/release.json"
echo ""


******************************************************************

TASK:

•	Prerequisites: Jenkins controller operational on http://localhost:8080.
•	Objective: Provision a job headlessly via Groovy that creates runtime artifacts, validates file size and structure, and exposes the artifacts for remote HTTP retrieval.

Lab Challenge Overview
You are tasked with deploying and validating a pipeline named lab-artifact-verifier.

Requirements

1.	Job Configuration:
o	Job Type: FreeStyleProject
o	Parameters:
	MODULE_NAME: String parameter (Default: auth-module)
	BUILD_NUMBER_ID: String parameter (Default: 101)

2.	Execution Steps (set -euo pipefail):
o	Teardown Trap: Print [LAB-TEARDOWN] Workspace check completed. on exit.
o	Artifact Generation: Create a directory ./build_out/ and generate ./build_out/manifest.json:
JSON
{
  "module": "<MODULE_NAME>",
  "build_id": "<BUILD_NUMBER_ID>",
  "status": "PASS"
}
o	Integrity Gate:
	Verify ./build_out/manifest.json exists and is non-empty (-s).
	Fail with exit code 1 and log to stderr if "status": "PASS" is missing.
o	Console Output: Print the contents of manifest.json.












