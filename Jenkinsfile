pipeline { 
    environment { 
        registry = "wahaj123/python" 
        registryCredential = 'dockerhub' 
        dockerImage = 'wahaj123/python' 
    }
    agent any 
    stages { 
        stage('Cloning Code Repo') { 
            steps { 
                dir("code") { 
                    git branch: 'main', 
                    credentialsId: 'github_ssh', 
                    url: 'git@github.com:wahaj123/flask_kubernetes.git'
                }
            }
        }

        stage('Cloning Kubernetes Repo') { 
            steps { 
                dir("kubernetes") { 
                    git branch: 'main', 
                    credentialsId: 'github_ssh', 
                    url: 'git@github.com:wahaj123/Kubernetes_app.git'
                }
            }
        }
        
        // stage ('Test'){
        //     steps {
        //         sh 'cd code && python3 -m unittest test.py'
        //     }
        // }

        stage("test PythonEnv") {
            steps {
                withPythonEnv('python3') {
                    sh 'cd code && pip install pytest'
                    sh 'cd code && pytest test.py'
                }
            }
        }

        stage('Code Analysis - SONARQUBE') {
            steps {
                script{
                    env.sonarHome= tool name: 'sonarqube', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                }
                withSonarQubeEnv('sonarqube') {
                    sh """
                       ${sonarHome}/bin/sonar-scanner \
                       -Dsonar.projectKey=flask-jenkins  \
                       -Dsonar.sources=. \
                    """
                }
            }
        }

        stage('DockerFile Liniting - HADOLINT ') {
            steps {
                sh 'cd code && docker run --rm -i -v ${PWD}/.hadolint.yml:/.hadolint.yaml hadolint/hadolint < Dockerfile'
            }
        }

        stage('Building Docker image - DOCKER PLUGIN') { 
            steps {
                script {
                    dockerImage = docker.build (registry + ":$BUILD_NUMBER", "./code") 
                }           
            }
        }
        
        stage ('Docker Image Vulnerability Scan - TRIVY '){
            steps{
                script{
                    sh 'trivy image --exit-code 0 --severity CRITICAL,HIGH  $registry:$BUILD_NUMBER'
                }
            }
        }
 
        stage('Push Docker image to Regisrty - DOCKERHUB/ECR/ACR') { 
            steps { 
                script { 
                    docker.withRegistry( '', registryCredential ) { 
                        dockerImage.push() 
                    }
                } 
            }
        }

        stage('Docker Image Tag update') { 
            steps { 
                sh "cd kubernetes/kubernetes && git checkout dev && kustomize edit set image $registry:$BUILD_NUMBER" 
            }
        }

        stage('Update Kubernetes Repo to Trigger Argo CD ') { 
            steps { 
                sshagent(['github_ssh']) {
                    sh "cd kubernetes && git commit -am 'Publish new version' && git push || echo 'no changes'"
                    sh "cd kubernetes && git push --set-upstream origin dev"
                    echo "$registry:$BUILD_NUMBER"
                }
            }
        }

        stage('Delete Local image locally') { 
            steps { 
                sh "docker rmi $registry:$BUILD_NUMBER"
            }
        } 
    }
}