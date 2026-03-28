pipeline {
    agent { label 'athome' }

    environment {
        REGISTRY = 'ghcr.io'
        IMAGE_NAME = 'mainnika/ghost'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Load .env') {
            steps {
                script {
                    def envVars = readFile('.env').trim().split('\n')
                    envVars.each { line ->
                        def (key, value) = line.tokenize('=')
                        env[key.trim()] = value.trim()
                    }
                }
            }
        }

        stage('Set up QEMU') {
            steps {
                sh 'docker run --rm --privileged multiarch/qemu-user-static --reset -p yes'
            }
        }

        stage('Setup Docker Buildx') {
            steps {
                sh '''
                    docker buildx create --name multiarch-builder --use || docker buildx use multiarch-builder
                    docker buildx inspect --bootstrap
                '''
            }
        }

        stage('Login to Registry') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'ghcr-io',
                    usernameVariable: 'REGISTRY_USER',
                    passwordVariable: 'REGISTRY_PASSWORD'
                )]) {
                    sh 'echo "$REGISTRY_PASSWORD" | docker login $REGISTRY -u "$REGISTRY_USER" --password-stdin'
                }
            }
        }

        stage('Build and Push') {
            matrix {
                axes {
                    axis {
                        name 'BASE'
                        values 'ubi'
                    }
                    axis {
                        name 'VERSION'
                        values '5'
                    }
                }
                stages {
                    stage('Build and Push Image') {
                        steps {
                            script {
                                def gitShortSha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                                def tags = [
                                    "${REGISTRY}/${IMAGE_NAME}:${env.GHOST_VERSION}-${BASE}",
                                    "${REGISTRY}/${IMAGE_NAME}:${VERSION}-${BASE}",
                                    "${REGISTRY}/${IMAGE_NAME}:sha-${gitShortSha}",
                                ].join(',')

                                sh """
                                    docker buildx build \
                                        --platform linux/amd64,linux/arm64 \
                                        --build-arg GHOST_CLI_VERSION=${env.GHOST_CLI_VERSION} \
                                        --build-arg GHOST_VERSION=${env.GHOST_VERSION} \
                                        --tag ${REGISTRY}/${IMAGE_NAME}:${env.GHOST_VERSION}-${BASE} \
                                        --tag ${REGISTRY}/${IMAGE_NAME}:${VERSION}-${BASE} \
                                        --tag ${REGISTRY}/${IMAGE_NAME}:sha-${gitShortSha} \
                                        --network host \
                                        --push \
                                        ./${VERSION}/${BASE}
                                """
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker buildx rm multiarch-builder || true'
            sh 'docker logout $REGISTRY || true'
        }
    }
}
