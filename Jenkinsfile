pipeline {
    agent {
        label 'k8s-slave'
    }
    parameters {
        choice (name: 'buildOnly',
               choices: 'no\nyes',
               description: "Build the Application Only"
        )
        choice (name: 'scanOnly',
               choices: 'no\nyes',
               description: "only Scan the Application"
        )
        choice (name: 'dockerPush',
               choices: 'no\nyes',
               description: "Docker Build and push to registry"
        )
        choice (name: 'deployToDev',
               choices: 'no\nyes',
               description: "Deploy app in DEV"
        )
        choice (name: 'deployToTest',
               choices: 'no\nyes',
               description: "Deploy app in TEST"
        )
        choice (name: 'deployToStaging',
               choices: 'no\nyes',
               description: "Deploy app in STAGE"
        )
        choice (name: 'deployToProd',
               choices: 'no\nyes',
               description: "Deploy app in PROD"
        )
    }

    tools{
        maven 'Maven-3.8.8'
        jdk 'JDK-17'
    }

    environment {
        APPLICATION_NAME = "users"
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        SONAR_URL = "http://34.46.21.82:9000"
        SONAR_TOKEN = credentials('sonar_creds')
        DOCKER_HUB = "docker.io/i27anilb3"
        DOCKER_CREDS = credentials('docker_creds')
        MAHA_CREDS = credentials('maha_creds')
        // DOCKER_HOST_IP = "10.1.0.9"
    }  
    stages{
       stage ('Build') {
            when {
                anyOf {
                    expression {
                        params.buildOnly == 'yes'
                    }
                }
            }
            steps {
                script {
                    buildApp().call()
                }

            }
        }
//        stage('Unit Test') {
//      steps {
//          echo "Executing UNIX test cases for ${env.APPLICATION_NAME} application"
//          sh 'mvn test'
//      }
//      post {
//          always {
//              junit 'target/surefire-reports/*xml'
//              jacoco execPattern: 'target/jacoco.exec'
//          }
//      }
// }
        stage ('Sonar') {
            when {
                anyOf {
                    expression {
                        params.scanOnly == 'yes'
                    }
                }
            }
            //sqa_a25af99d06b87a263ccf7aed9033cd9d80b97b36
            steps {

                sh """
                echo "Starting Sonar Scan"
                mvn sonar:sonar \
                    -Dsonar.projectKey=i27-eureka \
                    -Dsonar.host.url=${env.SONAR_URL} \
                    -Dsonar.login=${SONAR_TOKEN} 
                """
            }
        }

        stage ('Docker Build and push') {
                when {
                    anyOf {
                        expression {
                            params.dockerPush == 'yes'
                        }
                    }
                }            
            steps {
                script  {
                    dockerBuildandPush().call()
                }

            }
        }

        stage ('Deploy To Dev') {
                 when {
                    anyOf {
                        expression {
                            params.deployToDev == 'yes'
                        }
                    }
                } 
            steps {
             
              script {
                imageValidation().call()
                dockerDeploy('dev', '5232', '8232').call()
              }
                
            }

        }
        stage ('Deploy To Test') {
                when {
                    anyOf {
                        expression {
                            params.deployToTest == 'yes'
                        }
                    }
                } 
            steps {
  
              script {
                imageValidation().call()
                dockerDeploy('test', '6232', '8232').call()
              }
                
            }  
        }   
        stage ('Deploy To Staging') {
                when {
                    anyOf {
                        expression {
                            params.deployToStaging == 'yes'
                        }
                    }
                }  
            steps {
              script {
                imageValidation().call()
                dockerDeploy('stage', '7232', '8232').call()
              }
                
            }   
        }   
        stage ('Deploy To PROD') {
            when {
                allOf {
                    anyOf {
                        expression {
                            params.deployToProd == 'yes'
                            // other condition as well
                        }
                    }
                    anyOf {
                        branch 'release/*'
                        // one more condition as well
                    }
                }
            }
            steps {
                timeout(time: 300, unit: 'SECONDS') {
                input message: "Deploying to ${env.APPLICATION_NAME} to production ???", ok: 'yes', submitter: 'mat'
                   script {
                       imageValidation().call()
                       dockerDeploy('PROD', '8232', '8232').call()
                    }
                
                }   
            }  
        }
    
    }
}

def dockerDeploy(envDeploy, hostPort, contPort) {
    return {
        echo "*******Deploy to $envDeploy********" 
        echo "env is : ${envDeploy}"
        echo " hostPort : ${hostPort}"
        echo " contPort : ${contPort}"
        //sh('sshpass -p $MAHA_CREDS_PSW ssh -o StrictHostKeyChecking=no $MAHA_CREDS_USR@$DOCKER_DEPLOY_HOST_IP docker pull $DOCKER_HUB/$APPLICATION_NAME:$GIT_COMMIT')
        sh "sshpass -p ${MAHA_CREDS_PSW} ssh -o StrictHostKeyChecking=no ${MAHA_CREDS_USR}@${DOCKER_DEPLOY_HOST_IP} docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
        try {
            sh "sshpass -p ${MAHA_CREDS_PSW} ssh -o StrictHostKeyChecking=no ${MAHA_CREDS_USR}@${DOCKER_DEPLOY_HOST_IP} docker stop ${env.APPLICATION_NAME}-$envDeploy"
            sh "sshpass -p ${MAHA_CREDS_PSW} ssh -o StrictHostKeyChecking=no ${MAHA_CREDS_USR}@${DOCKER_DEPLOY_HOST_IP} docker rm ${env.APPLICATION_NAME}-$envDeploy"
        }catch(err) 
        {
            echo "caught Error : $err"
        }
        sh "sshpass -p ${MAHA_CREDS_PSW} ssh -o StrictHostKeyChecking=no ${MAHA_CREDS_USR}@${DOCKER_DEPLOY_HOST_IP} docker run -d -p $hostPort:$contPort --name ${env.APPLICATION_NAME}-$envDeploy ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
    }
}

def imageValidation() {
    return {
        println ("pulling he docker Image")
        try {
            sh "docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
        } 
        catch (Exception e) {
            println("OOPS!!!!!, docker image with this tag doesnot exists, So creating the image")
            buildApp().call()
            dockerBuildandPush().call()
        }
    }
}

def buildApp() {
    return {
        echo "Bulding ${env.APPLICATION_NAME} application"
        sh 'mvn clean package -DskipTests=true'
        archiveArtifacts artifacts: 'target/*jar'
    }
}

def dockerBuildandPush() {
    return {
        echo "Starting Docker build stage"
        sh "cp ${WORKSPACE}/target/i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} ./.cicd/"
        echo "**************************** Building Docker Image ****************************"
        sh "docker build --force-rm --no-cache --build-arg JAR_SOURCE=i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT} ./.cicd"
        echo "********Docker login******"
        sh "docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}"
        echo "********Docker Push******"
        sh "docker push ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"

    }
}

