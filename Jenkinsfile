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
                                docker build -t 3.80.51.26:8083/springapp:${VERSION} .
                                docker login -u admin -p $docker_password 3.80.51.26:8083 
                                docker push  3.80.51.26:8083/springapp:${VERSION}
                                docker rmi 3.80.51.26:8083/springapp:${VERSION}
                            '''
                    }
                }
            }
        }
    }
}
// withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) - the password of docker hosted reop is encrypted
// 54.91.157.218:8083/springapp:${VERSION}      54.91.157.218:8083/springapp - image name, ${VERSION} - tag (here build number is tag)
// docker build -t 54.91.157.218:8083/springapp:${VERSION} .     - build the image            
// docker login -u admin -p $docker-password 54.91.157.218:8083  - login into the docker hosted repo in nexus
// docker push  54.91.157.218:8083/springapp:${VERSION}          - push the image to the repo
// docker rmi 54.91.157.218:8083/springapp:${VERSION}            - deleting the image from jenkins to clean up space