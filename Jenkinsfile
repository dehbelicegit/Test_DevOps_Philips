pipeline {
    agent none

    triggers {
        cron('H 2 * * *')
    }

    stages {

        stage('Checkout') {
            agent { label 'build' }
            steps {
                checkout scm
            }
        }

        stage('Check') {
            agent { label 'build' }
            steps {
                dir('calculator') {
                    sh 'make check'
                }
            }
        }

        stage('Build') {
            agent { label 'build' }
            steps {
                dir('calculator') {
                    sh 'make clean && make'
                }
            }
        }

        stage('Test') {
            agent { label 'test' }
            steps {
                dir('calculator') {
                    sh 'make -C tests clean && make unittest'
                }
            }
        }

        stage('Package') {
            agent { label 'build' }
            steps {
                sh 'tar -czf artifact.tar.gz -C calculator/src/bin .'
                archiveArtifacts artifacts: 'artifact.tar.gz', fingerprint: true
            }
        }
    }

    post {
        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
        failure {
            echo 'Pipeline FAILED - check the logs above for details'
        }
        success {
            echo 'Pipeline SUCCESS - artifact archived successfully'
        }
    }
}
