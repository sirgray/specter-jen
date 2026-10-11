#!/usr/bin/env bash
set -euo pipefail

echo "================================================================="
echo "Headless Security Setup & Token Setup"
echo "================================================================="

# 1. Clean up old container and volume
echo "--> Cleaning up old instances..."
docker rm -f my-jenkins 2>/dev/null || true
docker volume rm jenkins_home 2>/dev/null || true

# 2. Run Jenkins using YOUR exact command
echo "--> Spawning Jenkins container..."
docker run -d \
  --name my-jenkins \
  -p 8080:8080 -p 50000:50000 \
  -e JAVA_OPTS="-Djenkins.install.runSetupWizard=false -Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true" \
  jenkins/jenkins:lts-jdk17

echo "--> Waiting 20 seconds for Jenkins JVM to fully boot..."
sleep 20

# 3. Execute Groovy directly inside JVM using plainValue property
echo "--> Injecting Security & Generating API Token via JVM Script Console..."
API_TOKEN=$(curl -s -X POST "http://localhost:8080/scriptText" \
  --data-urlencode "script=
import jenkins.model.*
import hudson.security.*
import jenkins.security.ApiTokenProperty

def instance = Jenkins.get()

// 1. Setup Admin User & Security
def hudsonRealm = new HudsonPrivateSecurityRealm(false)
def adminUser = hudsonRealm.createAccount('admin', 'admin123')
adminUser.save()
instance.setSecurityRealm(hudsonRealm)

def strategy = new FullControlOnceLoggedInAuthorizationStrategy()
strategy.setAllowAnonymousRead(false)
instance.setAuthorizationStrategy(strategy)

// 2. Generate API Token using correct plainValue accessor
def apiTokenProperty = adminUser.getProperty(ApiTokenProperty.class)
def tokenResult = apiTokenProperty.tokenStore.generateNewToken('MasterKey')
adminUser.save()
instance.save()

// 3. Output raw token value to STDOUT
print tokenResult.plainValue
" | tr -d '\r\n')

export API_TOKEN

echo "================================================================="
echo "  GENERATED CREDENTIALS"
echo "================================================================="
echo "Username  : admin"
echo "API Token : $API_TOKEN"
echo "================================================================="

# 4. Verify Authentication via REST API
echo "--> Verifying REST API access with generated token..."
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" -u "admin:$API_TOKEN" http://localhost:8080/api/json)

if [ "$HTTP_STATUS" -eq 200 ]; then
  echo "--> SUCCESS: API Authentication Verified (HTTP 200 OK)!"
else
  echo "--> ERROR: API returned HTTP status $HTTP_STATUS"
  exit 1
fi





cat << 'EOF' > job-env.xml
<?xml class="1.0" encoding="UTF-8"?>
<flow-definition plugin="workflow-job">
  <description>Lesson 4 Environment Variables Job</description>
  <definition class="org.jenkinsci.plugins.workflow.cps.CpsFlowDefinition" plugin="workflow-cps">
    <script>
pipeline {
    agent any

    // Custom environment variables accessible globally in this pipeline
    environment {
        APP_NAME     = "ECommerceAPI"
        DEPLOY_STAGE = "staging"
        MAX_RETRIES  = "3"
    }

    stages {
        stage('Inspect Environment') {
            steps {
                echo "=== Reading Built-in Variables ==="
                echo "Job Name     : ${env.JOB_NAME}"
                echo "Build Number : ${env.BUILD_NUMBER}"
                echo "Workspace    : ${env.WORKSPACE}"
                
                echo "=== Reading Custom Variables ==="
                echo "App Name     : ${env.APP_NAME}"
                echo "Target Stage : ${env.DEPLOY_STAGE}"
            }
        }

        stage('Override Variable Locally') {
            environment {
                // Overriding DEPLOY_STAGE specifically for this stage
                DEPLOY_STAGE = "production-canary"
            }
            steps {
                echo "Overridden DEPLOY_STAGE inside Stage: ${env.DEPLOY_STAGE}"
            }
        }

        stage('Verify Global Value Intact') {
            steps {
                echo "DEPLOY_STAGE value back in main scope: ${env.DEPLOY_STAGE}"
            }
        }
    }
}
    </script>
    <sandbox>true</sandbox>
  </definition>
</flow-definition>
EOF




# 1. Create the Groovy script safely
cat << 'EOF' > script.groovy
import jenkins.model.*
import hudson.model.*
import hudson.tasks.Shell
import hudson.slaves.EnvironmentVariablesNodeProperty

def instance = Jenkins.get()
def jobName = 'lesson4-env-demo'

// Get or create Freestyle Job
def job = instance.getItem(jobName) ?: instance.createProject(FreeStyleProject, jobName)

// Define multi-line shell step with environment variables inline
def shellScript = '''#!/usr/bin/env bash
export APP_ENV="STAGING"
export DB_PORT="5432"

echo "--- BUILT-IN JENKINS VARIABLES ---"
echo "Job Name: ${JOB_NAME}"
echo "Build Number: ${BUILD_NUMBER}"
echo "Workspace: ${WORKSPACE}"
echo "--- CUSTOM ENVIRONMENT BLOCK VARIABLES ---"
echo "App Environment: ${APP_ENV}"
echo "Database Port: ${DB_PORT}"
'''

job.getBuildersList().clear()
job.getBuildersList().add(new Shell(shellScript))
job.save()
instance.reload()

println "JOB_SUCCESSFULLY_PROVISIONED"
EOF

# 2. Execute Groovy Script via cURL
curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:$API_TOKEN" \
  --data-urlencode "script@script.groovy"
rm script.groovy

# 3. Trigger build & stream log
echo -e "\n--> Triggering Build..."
curl -X POST -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson4-env-demo/build"

echo "--> Waiting 3 seconds..."
sleep 3

echo -e "\n--> CONSOLE OUTPUT:"
curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson4-env-demo/1/consoleText"



------ In case of broken token-------:

# Re-authenticate using the admin password to get a valid API token
API_TOKEN=$(curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:admin123" \
  --data-urlencode "script=
import jenkins.model.*
import jenkins.security.ApiTokenProperty

def user = User.get('admin', false, [:])
def apiTokenProperty = user.getProperty(ApiTokenProperty.class)
def tokenResult = apiTokenProperty.tokenStore.generateNewToken('MasterKey')
user.save()
print tokenResult.plainValue
" | tr -d '\r\n')

export API_TOKEN

echo "Token Value: $API_TOKEN"

--------------------------------------------







# 1. Create the Groovy provisioner script locally
cat << 'EOF' > script.groovy
import jenkins.model.*
import hudson.model.*
import hudson.tasks.Shell

def instance = Jenkins.get()
def jobName = 'lesson5-shell-demo'

def job = instance.getItem(jobName) ?: instance.createProject(FreeStyleProject, jobName)

def multiLineShell = '''#!/usr/bin/env bash
set -euo pipefail

echo "=========================================================="
echo "  LESSON 5: MULTI-LINE SHELL SCRIPTING & HEREDOCS"
echo "=========================================================="

# 1. System Inspection
echo "--> Step 1: System Diagnostics"
echo "Hostname    : $(hostname)"
echo "Kernel      : $(uname -r)"
echo "Current Directory : $(pwd)"

# 2. File Generation via Heredoc
echo -e "\n--> Step 2: Generating Environment Payload"
cat << 'PAYLOAD' > config.env
APP_NAME="Jenkins Automation Pipeline"
DEPLOY_TARGET="Production Edge"
TIMESTAMP="$(date +'%Y-%m-%d %H:%M:%S')"
PAYLOAD

echo "Generated File Contents:"
cat config.env

# 3. Reading and processing the generated payload
echo -e "\n--> Step 3: Parsing Payload"
source config.env
echo "Loaded APP_NAME      : ${APP_NAME}"
echo "Loaded DEPLOY_TARGET : ${DEPLOY_TARGET}"

echo "=========================================================="
echo "  EXECUTION COMPLETED SUCCESSFULLY"
echo "=========================================================="
'''

job.getBuildersList().clear()
job.getBuildersList().add(new Shell(multiLineShell))
job.save()

println "JOB_LESSON5_PROVISIONED"
EOF

# 2. Deploy job to Jenkins via Script Console
curl -s -X POST "http://localhost:8080/scriptText" \
  -u "admin:$API_TOKEN" \
  --data-urlencode "script@script.groovy"
rm script.groovy

# 3. Trigger build & stream console output
echo -e "\n--> Triggering Build for lesson5-shell-demo..."
curl -X POST -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson5-shell-demo/build"

echo "--> Waiting 3 seconds for execution..."
sleep 3

echo -e "\n=========================================================="
echo "  BUILD CONSOLE OUTPUT"
echo "=========================================================="
curl -s -u "admin:$API_TOKEN" "http://localhost:8080/job/lesson5-shell-demo/1/consoleText"

------------------!!!!! RERUN IF FAILS---------------------






