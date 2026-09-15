
pipeline {
    agent any

    stages {
        stage('Build Artifact') {
            steps {
                // Generate a sample build output file
                sh 'mkdir -p build/libs'
                sh 'echo "Application Version 1.0.0 Build Output" > build/libs/app-1.0.0.jar'
                sh 'echo "Log output" > build/libs/build.log'
            }
        }

        stage('Archive Outputs') {
            steps {
                // Preserve .jar files while ignoring logs
                archiveArtifacts artifacts: 'build/libs/*.jar', allowEmptyArchive: false
            }
        }
    }
}

**FIRST TASK**

1.	Create a job named practice-lab-4.
2.	Configure 3 stages using agent { docker { ... } }:
o	Stage 1 (OS Check): Run alpine:latest and execute cat /etc/os-release.
o	Stage 2 (Go Check): Run golang:1.21-alpine and execute go version.
o	Stage 3 (Workspace Cleanup): Run alpine:latest and create/verify a file in the workspace.


**SECOND TASK - for file with test.sh**

1.	Add a second stage to your Git repository's Jenkinsfile named 'System Info'.
2.	Add a step executing sh 'df -h' to display server disk usage.
3.	Commit, push to Git, and click Build Now in Jenkins to verify that the changes automatically pull from Git.






