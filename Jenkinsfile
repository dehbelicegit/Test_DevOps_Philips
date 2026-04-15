pipeline {
    agent none

    triggers {
        cron('H H * * *')
    }

    stages {

        stage('Checkout') {
            agent { label 'build' }
            steps {
                checkout scm
            }
        }

        stage('Build') {
            agent { label 'build' }
            steps {
                dir('calculator') {
                    sh '''
                    mkdir -p build
                    cd build
                    cmake ..
                    make
                    '''
                }
            }
        }

        stage('Test') {
            agent { label 'test' }
            steps {
                dir('calculator/build') {
                    sh '''
                    ctest --output-on-failure
                    '''
                }
            }
        }

        stage('Package') {
            agent { label 'build' }
            steps {
                sh '''
                tar -czf artifact.tar.gz calculator/build/
                '''
            }
        }

        stage('Archive') {
            agent any
            steps {
                archiveArtifacts artifacts: 'artifact.tar.gz'
            }
        }
    }

    post {
        failure {
            echo 'Pipeline FAILED'
        }
        success {
            echo 'Pipeline SUCCESS'
        }
    }
}
