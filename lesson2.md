
pipeline {
    agent any

    stages {
        stage('Display Git Details') {
            steps {
                echo "Building Commit Hash: ${env.GIT_COMMIT}"
                echo "Building Branch Name: ${env.GIT_BRANCH}"
                echo "Jenkins Build Number: ${env.BUILD_NUMBER}"
                echo "Jenkins Job Name:     ${env.JOB_NAME}"
            }
        }
    }
}

!!!! FYI



#!/usr/bin/env bash
set -e

echo "=== 1. Updating OS & Installing Docker on Host ==="
sudo apt-get update && sudo apt-get install -y curl git ca-certificates

if ! command -v docker &> /dev/null; then
    sudo curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh
    sudo usermod -aG docker $USER
    rm -f get-docker.sh
fi

echo "=== 2. Cleaning Up Existing Container ==="
docker rm -f jenkins 2>/dev/null || true

echo "=== 3. Launching Jenkins Container ==="
docker run -d \
  --name jenkins \
  --restart always \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e JAVA_OPTS="-Djenkins.install.runSetupWizard=false -Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true" \
  jenkins/jenkins:lts-jdk17

echo "Waiting for Jenkins to initialize..."
sleep 15

echo "=== 4. Installing Docker CLI & Fixing Socket Permissions inside Container ==="
# Install docker CLI binary inside the container
docker exec -u 0 jenkins apt-get update
docker exec -u 0 jenkins apt-get install -y docker.io
# Grant access to socket
docker exec -u 0 jenkins chmod 666 /var/run/docker.sock

echo "=== 5. Pre-installing Jenkins Plugins ==="
docker exec jenkins jenkins-plugin-cli --plugins workflow-aggregator git docker-workflow
docker restart jenkins

echo "=== DONE! Jenkins is ready at http://$(curl -s ifconfig.me):8080 ==="

!!!! INITIAL SCRIPT


SCRIPT BREAK DOWN:

Section 0: Script Safety Guardrails
•	#!/usr/bin/env bash: Tells the system to execute this file using the Bash interpreter located in the user's environment path.
•	set -e: The "fail-fast" flag. If any command fails (returns a non-zero exit code), the script immediately stops executing instead of ploughing ahead and causing compound errors.
Section 1: Host Machine Preparation & Docker Engine Setup
•	sudo apt-get update && sudo apt-get install -y curl git ca-certificates: Updates the Linux package lists and installs essential tools needed for downloading files (curl), version control (git), and secure Web traffic (ca-certificates).
•	if ! command -v docker &> /dev/null; then ... fi: A conditional check. It checks if Docker is already installed on the server; if not, it runs the installation steps inside the block:
o	curl -fsSL [https://get.docker.com](https://get.docker.com) -o get-docker.sh: Downloads the official Docker automated installation script silently.
o	sudo sh get-docker.sh: Executes the installer to install Docker Engine on the host OS.
o	sudo usermod -aG docker $USER: Adds the current logged-in user to the docker user group so you can run Docker commands without typing sudo every time.
o	rm -f get-docker.sh: Cleans up and deletes the installer script after execution.
Section 2: Workspace Cleanup
•	docker rm -f jenkins 2>/dev/null || true: Guarantees a clean slate. Force-deletes (-f) any pre-existing container named jenkins. 2>/dev/null || true ensures that if no container named jenkins exists, the error message is hidden and the script doesn't stop.
Section 3: Launching the Jenkins Container
•	docker run -d \: Runs the container in detached mode (in the background).
•	--name jenkins \: Assigns the human-readable container name jenkins.
•	--restart always \: Configures Docker to automatically restart Jenkins if the server reboots or if the container crashes.
•	-p 8080:8080 -p 50000:50000 \:
o	8080:8080: Maps host port 8080 to container port 8080 (for the Web UI).
o	50000:50000: Maps host port 50000 to container port 50000 (for incoming Jenkins agent build nodes).
•	-v jenkins_home:/var/jenkins_home \: Creates a named Docker volume to store all pipeline jobs, configurations, and build history so data isn't lost when containers restart.
•	-v /var/run/docker.sock:/var/run/docker.sock \: Docker-out-of-Docker (DooD) Mount. Shares the host's Docker socket with the container so Jenkins can launch sibling Docker containers for build agents.
•	-e JAVA_OPTS="-Djenkins.install.runSetupWizard=false -Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true" \: Passes environment variables to Java to:
1.	Bypass the initial admin password / plugin setup wizard entirely.
2.	Disable CSRF Crumb Protection to prevent request-header token errors during automated testing.
•	jenkins/jenkins:lts-jdk17: Specifies the official Jenkins Long Term Support (LTS) image running Java 17.
•	sleep 15: Pauses execution for 15 seconds to give the Jenkins JVM time to initialize before trying to execute commands inside it.
Section 4: Enabling Containerized Docker Capabilities
•	docker exec -u 0 jenkins apt-get update: Executes a command inside the running jenkins container as root (-u 0) to update its internal Debian package lists.
•	docker exec -u 0 jenkins apt-get install -y docker.io: Installs the Docker CLI executable inside the Jenkins container. (This fixes the exit code 127: Command Not Found error when running docker steps in pipelines).
•	docker exec -u 0 jenkins chmod 666 /var/run/docker.sock: Grants read/write permissions on the mounted host socket file so the default jenkins non-root user inside the container can send commands to the host Docker daemon.
Section 5: Automated Plugin Management
•	docker exec jenkins jenkins-plugin-cli --plugins workflow-aggregator git docker-workflow: Uses the official Jenkins plugin CLI inside the container to pre-install essential plugins (workflow-aggregator for Pipelines, git for SCM, and docker-workflow for Docker pipeline steps) non-interactively.
•	docker restart jenkins: Restarts the Jenkins container so it loads the newly installed plugins into memory.
Section 6: Output Verification
•	echo "=== DONE! Jenkins is ready at http://$(curl -s ifconfig.me):8080 ===": Fetches the server's public IP address using ifconfig.me and prints out the final URL where students can access their ready-to-use Jenkins instance.


pipeline {
    agent any

    stages {
        stage('Environment Check') {
            steps {
                echo "Running on node: ${env.NODE_NAME}"
                sh 'uname -a'
                sh 'docker --version'
            }
        }
        stage('Hello World') {
            steps {
                echo 'Jenkins setup verified successfully!'
            }
        }
    }
}

!!!!! pipeline from UI



git clone https://github.com/sirgray/jenkins-lesson-2.git

cd jenkins-lesson-2

git config --global user.name "sirgray"
git config --global user.email "sergei@web-ace.com"

# 1. Create a simple test.sh script if you haven't already
echo '#!/bin/sh' > test.sh
echo 'echo "Hello from Git Pipeline!"' >> test.sh

# 2. Stage all files
git add .

# 3. Create initial commit
git commit -m "Add Jenkinsfile and test script"



# Create Jenkinsfile
cat << 'EOF' > Jenkinsfile
pipeline {
    agent any

    stages {
        stage('Checkout & Run') {
            steps {
                sh 'chmod +x test.sh'
                sh './test.sh'
            }
        }
    }
}
EOF

!!!!! --> Push to git







