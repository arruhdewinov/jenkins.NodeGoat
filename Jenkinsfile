pipeline {
    agent none
    environment {
        DOCKERHUB_CREDENTIALS = credentials('accesdocker')
        SNYK_CREDENTIALS = credentials('accessnyk')
    }
    stages {
        stage('Secret Scanning Using Trufflehog') {
            agent {
                docker {
                    image 'trufflesecurity/trufflehog:latest'
                    args '-u root --entrypoint='
                }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'trufflehog filesystem . --exclude-paths trufflehog-excluded-paths.txt --fail > trufflehog-scan-result.txt'
                }
                sh 'cat trufflehog-scan-result.txt'
                archiveArtifacts artifacts: 'trufflehog-scan-result.txt'
            }
        }
        stage('Build') {
            agent {
              docker {
                  image 'node:lts-buster-slim'
                  args '-u root --network host'
              }
            }
            steps {
                sh 'npm install'
            }
        }
        stage('Test') {
            agent {
              docker {
                  image 'node:lts-buster-slim'
              }
            }
            steps {
                sh 'npm run test'
            }
        }
        stage('SCA Snyk Test') {
            agent {
              docker {
                  image 'snyk/snyk:node'
                  args '-u root --network host --env SNYK_TOKEN=$SNYK_CREDENTIALS_PSW --entrypoint='
              }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'snyk test > snyk-scan-report.txt'
                }
                sh 'cat snyk-scan-report.txt'
                archiveArtifacts artifacts: 'snyk-scan-report.txt'
            }
        }
        stage('SCA Retire Js') {
            agent {
              docker {
                  image 'node:lts-buster-slim'
              }
            }
            steps {
                sh 'npm install retire'
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh './node_modules/retire/lib/cli.js --outputpath retire-scan-report.txt'
                }
                sh 'cat retire-scan-report.txt'
                archiveArtifacts artifacts: 'retire-scan-report.txt'
            }
        }
 /*       stage('SCA OWASP Dependency Check') {
            agent {
              docker {
                  image 'owasp/dependency-check:latest'
                  args '-u root -v /var/run/docker.sock:/var/run/docker.sock --entrypoint='
              }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '/usr/share/dependency-check/bin/dependency-check.sh --scan . --project "NodeJS Goof" --format ALL'
                }
                archiveArtifacts artifacts: 'dependency-check-report.html'
                archiveArtifacts artifacts: 'dependency-check-report.json'
                archiveArtifacts artifacts: 'dependency-check-report.xml'
            }
        }*/
        stage('Build Docker Image and Push to Docker Registry') {
            agent {
                docker {
                    image 'docker:dind'
                    args '--user root --network host -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh 'docker build -t xenjutsu/nodegoat:0.1 .'
                sh 'docker push xenjutsu/nodegoat:0.1'
            }
        }
        stage('Deploy Docker Image') {
            agent {
                docker {
                    image 'kroniak/ssh-client'
                    args '--user root --network host'
                }
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: "AccesSVRApp", keyFileVariable: 'keyfile')]) {
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no root@202.169.43.183 "echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin"'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no root@202.169.43.183 docker pull xenjutsu/nodegoat:0.1'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no root@202.169.43.183 docker rm --force nodegoat'
                    sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no root@202.169.43.183 docker run -it --detach -p 4000:4000 --name nodegoat --network host xenjutsu/nodegoat:0.1'
                }
            }
        }
    }
}
