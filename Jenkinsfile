pipeline {
    agent none

    triggers {
        cron('H H * * *')
    }

    stages {

        stage('Checkout') {
            agent { label 'build' }
            steps {
                git 'https://github.com/dehbelicegit/Test_DevOps_Philips.git'
            }
        }

        stage('Build') {
            agent { label 'build' }
            steps {
                sh '''
                mkdir -p build
                cd build
                cmake ..
                make
                '''
            }
        }

        stage('Test') {
            agent { label 'test' }
            steps {
                sh '''
                cd build
                ctest --output-on-failure
                '''
            }
        }

        stage('Package') {
            agent any
            steps {
                sh 'tar -czf artifact.tar.gz build/'
            }
        }

        stage('Archive') {
            agent any
            steps {
                archiveArtifacts artifacts: '*.tar.gz'
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
