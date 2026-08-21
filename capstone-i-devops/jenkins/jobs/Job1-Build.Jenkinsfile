pipeline {
    agent { label 'test' }
    parameters { string(name: 'SOURCE_BRANCH', defaultValue: 'develop', description: 'Branch to build: develop or master') }
    stages {
        stage('Checkout') {
            steps {
                git branch: params.SOURCE_BRANCH, url: 'https://github.com/maniyaraman744-hue/projects.git'
            }
        }
        stage('Build Docker image') {
            steps {
                dir('capstone-i-devops') {
                    sh 'docker build --pull -t abode-web:${BUILD_NUMBER} .'
                }
            }
        }
        stage('Trigger Job2-Test') {
            steps {
                build job: 'Job2-Test', parameters: [string(name: 'SOURCE_BRANCH', value: params.SOURCE_BRANCH)]
            }
        }
    }
}
