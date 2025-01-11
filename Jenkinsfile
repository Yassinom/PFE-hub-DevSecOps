pipeline {
    agent any
    tools {
        jdk 'jdk21'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git checkout') {
            steps {
                git branch: 'features', credentialsId: 'git-cred', url: 'https://github.com/SMBullet/DevSecOps'
            }
        }
        
        stage('Verify Workspace') {
            steps {
                sh 'ls -la' // Check workspace
                sh 'ls -la services/service_auth'
                sh 'ls -la services/service_project'
            }
        }

        // Function to Compile, Build, Scan, and Push for a Service
        stage('Process Microservices') {
            steps {
                script {
                    def services = ['service_auth', 'service_project']

                    for (service in services) {
                        echo "Processing $service..."

                        stage("Compile $service") {
                            dir("services/${service}") {
                                sh "mvn compile"
                            }
                        }

                        stage("Trivy FS Scan $service") {
                            dir("services/${service}") {
                                sh "mkdir -p /tmp/trivy/fsreports/${service}"
                                sh "trivy fs --format table -o /tmp/trivy/fsreports/${service}/fs_\$(date +%Y_%m_%d_%H_%M_%S).html ."
                            }
                        }

                        stage("SonarQube Analysis $service") {
                            withSonarQubeEnv('sonar-server') {
                                dir("services/${service}") {
                                    sh """
                                        $SCANNER_HOME/bin/sonar-scanner \
                                        -Dsonar.projectName=DevSecOps_${service} \
                                        -Dsonar.projectKey=DevSecOps_${service} \
                                        -Dsonar.java.binaries=. \
                                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                                    """
                                }
                            }
                        }

                        stage("Build $service") {
                            dir("services/${service}") {
                                sh "mvn package"
                            }
                        }

                        stage("Publish Artifacts $service") {
                            withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: 'jdk21', maven: 'maven3', traceability: true) {
                                dir("services/${service}") {
                                    sh "mvn deploy"
                                }
                            }
                        }

                        stage("Docker Build & Tag $service") {
                            withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                                sh "docker build -t yassinom/pfe_maroc_archive:${service} services/${service}"
                            }
                        }

                        stage("Trivy Image Scan $service") {
                            sh "mkdir -p /tmp/trivy/imagereports/${service}"
                            sh "trivy image --format table -o /tmp/trivy/imagereports/${service}/image_\$(date +%Y_%m_%d_%H_%M_%S).html yassinom/pfe_maroc_archive:${service}"
                        }

                        stage("Docker Push Image $service") {
                            withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                                sh "docker push yassinom/pfe_maroc_archive:${service}"
                            }
                        }
                    }
                }
                
                
            }
        }
        
        stage("DAST Scan : OWASP ZAP (auth MS)") {
            steps {
                script {
                    echo "---------------------------------------------"
                    echo "Checking for existing containers..."
                    
                    // Check and remove authMS container if it exists
                    sh '''
                        if [ $(docker ps -aq -f name=authMS) ]; then
                            echo "Stopping and removing existing authMS container"
                            docker stop authMS
                            docker rm authMS
                        fi
                    '''
                    
                    echo "Deploying the container : pfe_maroc_archive:service_auth"
                    sh "docker run --network host -d -p 5050:5050 --name authMS yassinom/pfe_maroc_archive:service_auth"
                    sh "sleep 25"
                    echo "--------Container successfully deployed----------"
                    sh "mkdir -p /tmp/zapreports"
        
                    //launching the ZAP DAST Testing 
                    sh "/opt/zap/zap.sh -cmd -port 8090 -quickurl http://localhost:5050 -quickout /tmp/AuthMSReport_\$(date +%Y_%m_%d_%H_%M_%S).html"                       
                
                    // Cleanup
                    sh "docker stop authMS"
                    sh "docker rm authMS"
                }
            }
        }
                
                
                
        stage("DAST Scan : OWASP ZAP (Project MS)") {
            steps {
                script {
                    echo "---------------------------------------------"
                    echo "Checking for existing containers..."
                    
                    // Check and remove projectMS container if it exists
                    sh '''
                        if [ $(docker ps -aq -f name=projectMS) ]; then
                            echo "Stopping and removing existing projectMS container"
                            docker stop projectMS
                            docker rm projectMS
                        else
                            echo "No existing projectMS container found"
                        fi
                    '''
            
                    echo "Deploying the container : pfe_maroc_archive:service_project"
                    sh "docker run --network host -d -p 6060:6060 --name projectMS yassinom/pfe_maroc_archive:service_project"
                    sh "sleep 25"
                    echo "--------Container successfully deployed----------"
                    
                    //launching the ZAP DAST Testing 
                    sh "/opt/zap/zap.sh -cmd -port 8010 -quickurl http://localhost:6060 -quickout /tmp/ProjectMSReport_\$(date +%Y_%m_%d_%H_%M_%S).html"    
                    
                    // Cleanup
                    sh "docker stop projectMS"
                    sh "docker rm projectMS"
                }
            }
        }







        stage('Deploying Apps to Kubernetes') {
            steps {
                script {
                    sh '''
                    echo "--------------------------------------"
                    echo "--------------------------------------"
                    echo "Cleaning up Kubernetes resources..."
                    kubectl delete all --all -n default --kubeconfig=/usr/share/jenkins/kubeconfig
                    echo "--------------------------------------"
                    echo "--------------------------------------"
                    echo "Deploying updated services to Kubernetes..."
                    kubectl apply -f deploymentservice.yml --kubeconfig=/usr/share/jenkins/kubeconfig
                    '''
                }
            }
        }
    }
}
