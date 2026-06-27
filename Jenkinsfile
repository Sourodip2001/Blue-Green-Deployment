pipeline {

    agent any

    tools{

        jdk 'jdk17'

        maven 'maven3'

    }

    

     parameters {

        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')

        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')

        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')

    }

    

    environment{

        IMAGE_NAME = "sourodip290301/bankapp"

        TAG = "${params.DOCKER_TAG}"  

        SCANNER_HOME = tool 'sonar-scanner'

        KUBE_NAMESPACE = 'webapps'

    }

    stages {

        stage('git-checkout') {

            steps {

               git branch: 'main', url: 'https://github.com/Sourodip2001/Blue-Green-Deployment.git'

            }

        }

        stage('Check Java') {

    steps {

        sh '''

        java -version

        javac -version

        mvn -version

        echo $JAVA_HOME

        '''

        }

    }

        stage('compile') {

            steps {

                sh "mvn compile"

            }

        }

        stage('Test cases') {

            steps {

                sh "mvn test -DskipTests=true"

            }

        }

        stage('trivy fs scan') {

            steps {

                sh "trivy fs --format table -o fs.html ."

            }

        }

        stage('SonarQube Analysis') {

            steps {

                withSonarQubeEnv('sonar') {

                    sh """

                    $SCANNER_HOME/bin/sonar-scanner \

                    -Dsonar.projectKey=Multitier \

                    -Dsonar.projectName=Multitier \

                    -Dsonar.sources=. \

                    -Dsonar.java.binaries=target/classes

                    """

                }

            }

        }

        stage('quality gate check') {

            steps {

                timeout(time: 1, unit: 'HOURS') {

                    waitForQualityGate abortPipeline: false

                }

            }

        }

        stage('build') {

            steps {

                sh "mvn package -DskipTests=true"

            }

        }

        stage('publih artifact nexus') {

            steps {

                withMaven(globalMavenSettingsConfig: 'maven-settings', maven: 'maven3', traceability: true) {

                    sh "mvn deploy -DskipTests=true"

                }

            }

        }

        stage('docker build') {

            steps {

               script{

                   withDockerRegistry(credentialsId: 'docker-cred') {

                        sh "docker build -t ${IMAGE_NAME}:${TAG} ."

                    }

               }

            }

        }

        stage('docker image scan ') {

            steps {

                sh "trivy image --format table -o fs.html ${IMAGE_NAME}:${TAG}"

            }

        }

        stage('docker push') {

            steps {

               script{

                   withDockerRegistry(credentialsId: 'docker-cred') {

                        sh "docker push  ${IMAGE_NAME}:${TAG} "

                    }

               }

            }

        }

        stage('Deploy MySQL Deployment and Service') {

            steps {

                script {

                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://9567B9CD70AD9A26916033598A0B226B.gr7.us-east-1.eks.amazonaws.com') {

                        sh "kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}"  // Ensure you have the MySQL deployment YAML ready

                    }

                }

            }

        }

        stage('Deploy SVC-APP') {

            steps {

                script {

                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://9567B9CD70AD9A26916033598A0B226B.gr7.us-east-1.eks.amazonaws.com') {

                        sh """ if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then

                                kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}

                              fi

                        """

                   }

                }

            }

        }

        stage('Deploy to Kubernetes') {

            steps {

                script {

                    def deploymentFile = ""

                    if (params.DEPLOY_ENV == 'blue') {

                        deploymentFile = 'app-deployment-blue.yml'

                    } else {

                        deploymentFile = 'app-deployment-green.yml'

                    }



                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://9567B9CD70AD9A26916033598A0B226B.gr7.us-east-1.eks.amazonaws.com') {

                        sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"

                    }

                }

            }

        }

        stage('Switch Traffic Between Blue & Green Environment') {

            when {

                expression { return params.SWITCH_TRAFFIC }

            }

            steps {

                script {

                    def newEnv = params.DEPLOY_ENV



                    // Always switch traffic based on DEPLOY_ENV

                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://9567B9CD70AD9A26916033598A0B226B.gr7.us-east-1.eks.amazonaws.com') {

                        sh '''

                            kubectl patch service bankapp-service -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"bankapp\\", \\"version\\": \\"''' + newEnv + '''\\"}}}" -n ${KUBE_NAMESPACE}

                        '''

                    }

                    echo "Traffic has been switched to the ${newEnv} environment."

                }

            }

        }

         stage('Verify Deployment') {

            steps {

                script {

                    def verifyEnv = params.DEPLOY_ENV

                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://9567B9CD70AD9A26916033598A0B226B.gr7.us-east-1.eks.amazonaws.com') {

                        sh """

                        kubectl get pods -l version=${verifyEnv} -n ${KUBE_NAMESPACE}

                        kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}

                        """

                    }

                }

            }

        }

    

    }

}
