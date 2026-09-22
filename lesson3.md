1.	Create a public repository on GitHub named jenkins-webhook-demo
2.	Add a single file named Jenkinsfile inside the GitHub repo with this content

pipeline {
    agent any

    triggers {
        // Empty pollSCM registers the job for instant GitHub Webhook pushes 
        // without running background polling schedules!
        pollSCM('')
    }

    stages {
        stage('Validate Webhook Trigger') {
            steps {
                echo '=== Instant Webhook Event Received ==='
                sh 'echo "Execution Time: $(date)"'
                sh 'git log -1 --oneline'
            }
        }
    }
}


Install GitHub Plugin and restart Jenkin



3. lesson3-github-webhook:
o	Definition: Select Pipeline script from SCM.
o	Select “GitHub hook trigger for GITScm polling”
o	SCM: Select Git.
o	Repository URL: Enter your public GitHub repo URL (e.g., [https://github.com/sirgray/jenkins-webhook-demo.git](https://github.com/sirgray/jenkins-webhook-demo.git)).
o	Branch Specifier: */main or */master.


	
  
  
  In GitHub, go to Repository Settings → Webhooks → Add webhook.
	Payload URL: https://b275349ae0bf370f-1-8080.spca.r.killercoda.com/github-webhook/ (The trailing slash is required!)
	Content type: application/json
	Click Add webhook.




Pipeline Script (lesson3-parameterized-job):
Groovy
pipeline {
    agent any

    parameters {
        string(name: 'TARGET_ENV', defaultValue: 'staging', description: 'Deployment target environment')
        string(name: 'RELEASE_TAG', defaultValue: 'v1.0.0', description: 'Container image release tag')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Force full integration testing')
    }

    stages {
        stage('Inspect Parameters') {
            steps {
                echo "Deploying Release Tag: ${params.RELEASE_TAG}"
                echo "Target Environment: ${params.TARGET_ENV}"
                echo "Execute Test Suite: ${params.RUN_TESTS}"
            }
        }

        stage('Simulate Deployment') {
            steps {
                sh '''
                    echo "Deploying release ${RELEASE_TAG} to environment ${TARGET_ENV}..."
                    echo "Completed at $(date)"
                '''
            }
        }
    
curl -X POST "https://a374e65c7ffaefe5-1-8080.spca.r.killercoda.com/job/lesson3-parameterized-job/buildWithParameters?RELEASE_TAG=v2.5.1&TARGET_ENV=production&RUN_TESTS=false"


ADD USER

docker exec -u 0 jenkins bash -c 'mkdir -p /var/jenkins_home/init.groovy.d && cat << "EOF" > /var/jenkins_home/init.groovy.d/create-admin.groovy
import jenkins.model.*
import hudson.security.*

def instance = Jenkins.get()

def hudsonRealm = new HudsonPrivateSecurityRealm(false)
def user = hudsonRealm.createAccount("admin", "password")
user.save()
instance.setSecurityRealm(hudsonRealm)

def strategy = new FullControlOnceLoggedInAuthorizationStrategy()
strategy.setAllowAnonymousRead(false)
instance.setAuthorizationStrategy(strategy)

instance.save()
println "--> SUCCESS: User admin created successfully."
EOF
chown -R 1000:1000 /var/jenkins_home/init.groovy.d'

ADD TOKEN

docker exec jenkins bash -c '
curl -s -u admin:password -X POST "http://localhost:8080/me/descriptorByName/jenkins.security.ApiTokenProperty/generateNewToken?newTokenName=remote-trigger-token"
'

curl -X POST -i -u "admin:110ee17f01e1dd50d87dd877a31fe082b" \
  "http://localhost:8080/job/lesson3-parameterized-job/buildWithParameters?TARGET_ENV=production&RELEASE_TAG=v3.0.0"









