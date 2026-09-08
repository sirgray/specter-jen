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





