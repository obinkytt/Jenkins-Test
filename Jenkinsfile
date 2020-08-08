pipeline {
     agent any
     stages {
         stage('Build and send initial slack message') {
              steps {
                 bat 'echo Building...'
                  slackSend message: "Build Started - ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"
              }
         }
         stage('Lint Dockerfile') {
            steps {
                script {
                    docker.image('hadolint/hadolint:latest-debian').inside() {
                          bat 'hadolint ./Dockerfile | tee -a hadolint_lint.txt'
                           bat '''
                                lintErrors=$(stat --printf="%s"  hadolint_lint.txt)
                                if [ "$lintErrors" -gt "0" ]; then
                                    echo "Errors have been found, please see below"
                                    cat hadolint_lint.txt
                                    exit 1
                                else
                                    echo "There are no erros found on Dockerfile!!"
                                fi
                            '''
                    }
                }
            }
        }
         stage('Security Scan') {
              steps { 
                 aquaMicroscanner imageName: 'node:12.13.1-stretch-slim', notCompliesCmd: 'exit 1', onDisallowed: 'fail', outputFormat: 'html'
              }
         }     
         stage('Build Docker Image') {
              steps {
                  bat 'docker build -t capstone-app-sagarnil .'
              }
         }
         stage('Push Docker Image') {
              steps {
                  withDockerRegistry([url: "", credentialsId: "dockerhub"]) {
                      bat "docker tag capstone-app-sagarnil sagarnildass/capstone-app-sagarnil"
                      bat 'docker push sagarnildass/capstone-app-sagarnil'
                  }
              }
         }
         stage('Deploying') {
              steps{
                  echo 'Deploying to AWS...'
                  withAWS(credentials: 'aws-static', region: 'ap-south-1') {
                     bat "aws eks --region ap-south-1 update-kubeconfig --name capstoneclustersagarnil"
                      bat "kubectl config use-context arn:aws:eks:ap-south-1:960920920983:cluster/capstoneclustersagarnil"
                      bat "kubectl apply -f capstone-k8s.yaml"
                      bat "kubectl get nodes"
                      bat "kubectl get deployments"
                      bat "kubectl get pod -o wide"
                      bat "kubectl get service/capstone-app-sagarnil"
                  }
              }
        }
        stage('Checking if app is up') {
              steps{
                  echo 'Checking if app is up...'
                  withAWS(credentials: 'aws-static', region: 'ap-south-1') {
                     bat "curl ad0e6a88870a9477989eb79393197b59-2120449898.ap-south-1.elb.amazonaws.com:9080"
                     slackSend(message: "The app is up at: ad0e6a88870a9477989eb79393197b59-2120449898.ap-south-1.elb.amazonaws.com:9080", sendAsText: true)
                  }
               }
        }
        stage('Checking rollout') {
              steps{
                  echo 'Checking rollout...'
                  withAWS(credentials: 'aws-static', region: 'ap-south-1') {
                     bat "kubectl rollout status deployments/capstone-app-sagarnil"
                  }
              }
        }
        stage("Cleaning up") {
              steps{
                    echo 'Cleaning up...'
                    bat "docker system prune"
              }
        }
     }
}
