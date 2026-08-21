pipeline {
    agent { label 'test' }
    parameters { string(name: 'SOURCE_BRANCH', defaultValue: 'develop', description: 'Branch to test') }
    stages {
        stage('Checkout and build') {
            steps {
                git branch: params.SOURCE_BRANCH, url: 'https://github.com/maniyaraman744-hue/projects.git'
                dir('capstone-i-devops') {
                    sh 'docker build --pull -t abode-test:${BUILD_NUMBER} .'
                }
            }
        }
        stage('Test Apache application') {
            steps {
                sh '''
                    docker rm -f abode-test || true
                    docker run -d --name abode-test -p 8081:80 abode-test:${BUILD_NUMBER}
                    for attempt in 1 2 3 4 5 6; do curl -fsS http://localhost:8081/ && exit 0; sleep 2; done
                    exit 1
                '''
            }
        }
        stage('Trigger production for master only') {
            when { expression { params.SOURCE_BRANCH == 'master' } }
            steps {
                build job: 'Job3-Prod', parameters: [string(name: 'SOURCE_BRANCH', value: 'master')]
            }
        }
    }
    post { always { sh 'docker rm -f abode-test || true' } }
}
