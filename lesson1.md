docker run -d \
  --name my-jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts-jdk17

--> 8080

docker exec my-jenkins cat /var/jenkins_home/secrets/initialAdminPassword


WITHOUT WIZARD

docker rm -f my-jenkins && docker volume rm jenkins_home

docker run -d \
  --name my-jenkins \
  -p 8080:8080 -p 50000:50000 \
  -e JAVA_OPTS="-Djenkins.install.runSetupWizard=false -Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true" \
  jenkins/jenkins:lts-jdk17


***************************************

New Item →Name: Hello-World-Freestyle →Select Freestyle project 

echo "Hello World from Jenkins!"
echo "Current date is: $(date)"

****************************************

New item Name: My-First-Pipeline →Select Pipeline 

pipeline {
    agent any

    stages {
        stage('Compile') {
            steps {
                echo 'Compiling source code...'
                sh 'echo "Simulating code compilation complete."'
            }
        }
        stage('Test') {
            steps {
                echo 'Running automated tests...'
                sh 'echo "All 15 tests passed!"'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying application...'
                sh 'echo "App successfully deployed to Staging!"'
            }
        }
    }
}
 ADD DEPLOY TO SEVERAL ENV.


 **********************************

 docker run -d \
  --name my-jenkins \
  -p 8080:8080 -p 50000:50000 \
  -e JAVA_OPTS="-Djenkins.install.runSetupWizard=false -Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true" \
  jenkins/jenkins:lts-jdk17


2.	Repository name: jenkins-lesson-2

3.	git config --global user.name "sirgray"
git config --global user.email "sergei@web-ace.com"

git clone https://github.com/sirgray/jenkins-lesson-2.git
cd jenkins-lesson-2


nano Jenkinsfile

pipeline {
    agent any

    stages {
        stage('Checkout & Verification') {
            steps {
                echo 'Checking workspace directory...'
                sh 'ls -la'
            }
        }
        stage('Test App') {
            steps {
                echo 'Simulating test execution on code pulled from Git...'
                sh 'echo "Code tests passed successfully!"'
            }
        }
    }
}

Save and exit (Ctrl + O, Enter, Ctrl + X in nano)

git add Jenkinsfile
git commit -m "Add initial Jenkinsfile"
# Note: GitHub will prompt for username and Personal Access Token (PAT) or password
git push origin main



1.Create a new Pipeline Item
2.Configure Pipeline Definition

3. add stage to Jenkinsfile via terminal

4.	Create a file named version.txt in the Git repository containing v1.0.0.
5.	Write a Jenkinsfile that:
 Reads and prints the content of version.txt using the shell command sh 'cat version.txt'.















