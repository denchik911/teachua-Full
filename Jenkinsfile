pipeline {
    agent any    // Jenkins бере код із Git; етапи нижче запускають контейнери

    options {
        timestamps()                                    // час біля кожного рядка логу
        disableConcurrentBuilds()                       // одна збірка гілки за раз (економимо памʼять)
        buildDiscarder(logRotator(numToKeepStr: '10'))  // зберігати останні 10 збірок
        timeout(time: 30, unit: 'MINUTES')              // не висіти вічно
    }

    environment {
        MAVEN_OPTS   = '-Xmx768m'                   // обмеження памʼяті Maven
        NODE_OPTIONS = '--max_old_space_size=1536'  // обмеження памʼяті Node
        CI           = 'false'                      // щоб попередження React не вважались помилками
        // Лише для CI-збірки. Справжня адреса вшивається під час розгортання.
        REACT_APP_ROOT_SERVER = 'http://localhost:8080'
    }

    stages {

        stage('Info') {
            steps {
                sh '''
                    echo "Branch: ${BRANCH_NAME}"
                    docker version --format 'Docker server: {{.Server.Version}}'
                '''
            }
        }

        stage('Backend: build') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-17'   // Java 17 + Maven всередині
                    reuseNode true                         // працювати в тій самій папці з кодом
                }
            }
            steps {
                dir('teachua-backend') {

                    sh 'mvn -B -Dmaven.repo.local=.m2/repository clean package'
                }
            }
            post {
                success {
                    archiveArtifacts artifacts: 'teachua-backend/target/*.war', fingerprint: true
                }
            }
        }

        stage('Frontend: build') {
            agent {
                docker {
                    image 'node:16'                        // Node 16 всередині
                    reuseNode true
                }
            }
            steps {
                dir('teachua-frontend') {
                    sh '''
                        sed -i 's|^//export const ROOT_SERVER.*|export const ROOT_SERVER = process.env.REACT_APP_ROOT_SERVER;|' src/config/ApplicationConfig.js
                        npm ci --cache .npm-cache
                        npm run build
                    '''
                }
            }
        }
    }

    post {
        success { echo 'CI passed' }
        failure { echo 'CI failed: read the log above, the first red error is the cause' }
    }
}
