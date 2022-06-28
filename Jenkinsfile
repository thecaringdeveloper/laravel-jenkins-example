pipeline {
    agent any
    stages {
        stage("Verify tooling") {
            steps {
                sh '''
                    docker info
                    docker version
                    docker compose version
                '''
            }
        }
        stage("Verify SSH connection to server") {
            steps {
                sshagent(credentials: ['aws-ec2']) {
                    sh '''
                        ssh ec2-user@13.41.191.171 whoami
                    '''
                }
            }
        }        
        stage("Clear all running docker containers") {
            steps {
                script {
                    try {
                        sh 'docker rm -f $(docker ps -a -q)'
                    } catch (Exception e) {
                        echo 'No running container to clear up...'
                    }
                }
            }
        }
        stage("Start Docker") {
            steps {
                sh 'make up'
                sh 'docker compose ps'
            }
        }
        stage("Run Composer Install") {
            steps {
                sh 'docker compose run --rm composer install'
            }
        }
        stage("Populate .env file") {
            steps {
                dir("/var/lib/jenkins/workspace/env/laravel-docker-jenkins-example") {
                    fileOperations([fileCopyOperation(excludes: '', flattenFiles: true, includes: '.env', targetLocation: "${WORKSPACE}")])
                }
            }
        }        
        stage("Run Tests") {
            steps {
                sh 'docker compose run --rm artisan test'
            }
        }
    }
    post {
        success {
            sh 'cd /var/lib/jenkins/workspace/env/laravel-docker-jenkins-example'
            sh 'rm -rf artifact.zip'
            sh 'zip -r artifact.zip . -x "*node_modules**"'
            withCredentials([sshUserPrivateKey(credentialsId: "aws-ec2", keyFileVariable: 'ec2')]) {
                sh 'scp -v -o StrictHostKeyChecking=no -i ${key} /var/lib/jenkins/workspace/env/laravel-docker-jenkins-example/artifact.zip ec2-user@13.41.191.171:/home/ec2-user/artifact'
            }
        }
        always {
            sh 'docker compose down --remove-orphans -v'
            sh 'docker compose ps'
        }
    }
}