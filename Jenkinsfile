pipeline{
    agent any 
    environment{
        VERSION = "${env.BUILD_ID}"
    }
    stages{
        stage("sonar quality check"){
            agent {
                docker {
                    image 'openjdk:11'
                }
            }
            steps{
                script{
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                            sh 'chmod +x gradlew'
                            sh './gradlew sonarqube'
                    }

                    timeout(time: 1, unit: 'HOURS') {
                      def qg = waitForQualityGate()
                      if (qg.status != 'OK') {
                           error "Pipeline aborted due to quality gate failure: ${qg.status}"
                      }
                    }

                }  
            }
        }
        stage("docker build & docker push"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
                             sh '''
                                docker build -t 34.228.41.252:8083/springapp:${VERSION} .
                                docker login -u admin -p $docker_password 34.228.41.252:8083 
                                docker push  34.228.41.252:8083/springapp:${VERSION}
                                docker rmi 34.228.41.252:8083/springapp:${VERSION}
                            '''
                    }
                }
            }
        }
        stage('indentifying misconfigs using datree in helm charts'){
            steps{
                script{

                    dir('kubernetes/') {
                         withEnv(['DATREE_TOKEN=46928342-3820-46bd-a262-7bbf1aae5cc3']) {
                              sh 'helm datree test /root/.jenkins/workspace/Java_gradle_application/kubernetes/myapp/'
                         }
                    }
                }
            }
        }
    }
    post {
		always {
			mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL de build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "priyabratapatra1997@gmail.com";  
		}
	}
}
// withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) - the password of docker hosted reop is encrypted
// 54.91.157.218:8083/springapp:${VERSION}      54.91.157.218:8083/springapp - image name, ${VERSION} - tag (here build number is tag)
// docker build -t 54.91.157.218:8083/springapp:${VERSION} .     - build the image            
// docker login -u admin -p $docker-password 54.91.157.218:8083  - login into the docker hosted repo in nexus
// docker push  54.91.157.218:8083/springapp:${VERSION}          - push the image to the repo
// docker rmi 54.91.157.218:8083/springapp:${VERSION}            - deleting the image from jenkins to clean up space