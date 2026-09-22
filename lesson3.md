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






