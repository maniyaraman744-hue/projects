pipeline {
    agent { label 'prod' }
    parameters { string(name: 'SOURCE_BRANCH', defaultValue: 'master', description: 'Must remain master for production') }
    stages {
        stage('Verify branch') {
            steps {
                script {
                    if (params.SOURCE_BRANCH != 'master') { error('Production deploys master only.') }
                }
            }
        }
        stage('Checkout and build') {
            steps {
                git branch: 'master', url: 'https://github.com/maniyaraman744-hue/projects.git'
                dir('capstone-i-devops') { sh 'docker build --pull -t abode-web:prod .' }
            }
        }
        stage('Deploy and verify') {
            steps {
                sh '''
                    docker rm -f abode-production || true
                    docker run -d --name abode-production -p 80:80 --restart unless-stopped abode-web:prod
                    for attempt in 1 2 3 4 5 6; do curl -fsS http://localhost/ && exit 0; sleep 2; done
                    exit 1
                '''
            }
        }
    }
}
