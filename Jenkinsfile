pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Checkout Kubernetes') {
            steps {
                git branch: 'Kubernetes',
                    url: 'https://github.com/denchik911/teachua-Full.git'

                sh 'git log -1 --oneline'
            }
        }

        stage('Build backend image') {
            steps {
                sh '''
                    docker build \
                      -t teachua-backend:ci-$BUILD_NUMBER \
                      ./teachua-backend
                '''
            }
        }

        stage('Build frontend image') {
            steps {
                sh '''
                    docker build \
                      -t teachua-frontend:ci-$BUILD_NUMBER \
                      ./teachua-frontend
                '''
            }
        }

        stage('Deliver images to Debian') {
            steps {
                sh '''
                    docker save -o teachua-images.tar \
                      teachua-backend:ci-$BUILD_NUMBER \
                      teachua-frontend:ci-$BUILD_NUMBER
                '''

                sshagent(credentials: ['teachua-k3s-ssh']) {
                    sh '''
                        scp \
                          -o BatchMode=yes \
                          -o StrictHostKeyChecking=yes \
                          -o ConnectTimeout=10 \
                          teachua-images.tar \
                          manager@192.168.50.139:/home/manager/labs/teachua-cicd/incoming/teachua-images.tar
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sshagent(credentials: ['teachua-k3s-ssh']) {
                    sh '''
                        ssh \
                          -o BatchMode=yes \
                          -o StrictHostKeyChecking=yes \
                          -o ConnectTimeout=10 \
                          manager@192.168.50.139 \
                          "sudo -n /usr/local/sbin/teachua-deploy $BUILD_NUMBER"
                    '''
                }
            }
        }
    }
}
