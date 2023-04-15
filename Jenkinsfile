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
                                docker build -t 50.16.170.178:8083/springapp:${VERSION} .
                                docker login -u admin -p $docker_password 50.16.170.178:8083 
                                docker push  50.16.170.178:8083/springapp:${VERSION}
                                docker rmi 50.16.170.178:8083/springapp:${VERSION}
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
                              sh 'helm datree test myapp/'
                         }
                    }
                }
            }
        }
        stage(" pushing the helm charts to nexus"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
                             dir('kubernetes/') {
                             sh '''
                                 helmversion=$( helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                                 tar -czvf  myapp-${helmversion}.tgz myapp/
                                 curl -u admin:$docker_password http://50.16.170.178:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v
                            '''
                        }
                    }
                }
            }
        }
        
        stage('deploying application on k8s cluster') {
            steps {
                script{
                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'kubernetes-config', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                        dir('kubernetes/') {
                          sh 'helm upgrade --install --set image.repository="50.16.170.178:8083/springapp" --set image.tag="${VERSION}" myjavaapp myapp/ ' 
                        }
                    }
                }
            }
        }
        stage('verifying app deployment') {
            steps {
                script{
                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'kubernetes-config', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                        sh 'kubectl run curl --image=curlimages/curl -i --rm --restart=Never -- curl myjavaapp-myapp:8080'
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