pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: golang
                    image: golang:1.23
                    command:
                    - sleep
                    args:
                    - 99d
                  - name: buildah
                    image: quay.io/buildah/stable:latest
                    securityContext:
                      privileged: true
                    tty: true
                    command:
                    - sleep
                    args:
                    - infinity
                  - name: trivy
                    image: aquasec/trivy:latest
                    tty: true
                    command:
                    - sleep
                    args:
                    - infinity
                  - name: grype
                    image: alpine:latest
                    tty: true
                    command:
                    - sleep
                    args:
                    - infinity
            '''
        }
    }

    environment {
        DOCKERHUB_USER = 'cloudbeesdemo'
        IMAGE_NAME = "${DOCKERHUB_USER}/hackers-api"
        TAR_FILE = "container-image.tar"
        REGISTRY = "docker.io"
        IMAGE_TAG = "3.0-${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                container('golang') {
                    sh '''
                        go install github.com/swaggo/swag/cmd/swag@latest
                        swag init
                        go test -coverprofile=coverage.out -v ./...
                        go build -buildvcs=false
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'coverage.out', allowEmptyArchive: true
                }
            }
        }

        stage('Build Container Image') {
            steps {
                container('buildah') {
                    sh """
                        buildah build -t ${env.IMAGE_NAME}:${env.IMAGE_TAG} .
                        buildah push ${env.IMAGE_NAME}:${env.IMAGE_TAG} docker-archive:./${env.TAR_FILE}
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'container-image.tar', allowEmptyArchive: true
                }
            }
        }

        stage('Security Scans') {
            parallel {
                stage('Trivy Container Scan') {
                    steps {
                        container('trivy') {
                            sh '''
                                trivy image --input container-image.tar --format sarif --output trivy-results.sarif
                                trivy image --input container-image.tar --format table
                            '''
                        }
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'trivy-results.sarif', allowEmptyArchive: true
                        }
                    }
                }
                stage('Grype Vulnerability Scan') {
                    steps {
                        container('grype') {
                            sh '''
                                # Install Grype in Alpine container
                                apk add --no-cache curl
                                curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin

                                # Run scans
                                grype docker-archive:container-image.tar -o sarif > grype-results.sarif
                                grype docker-archive:container-image.tar -o table
                            '''
                        }
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'grype-results.sarif', allowEmptyArchive: true
                        }
                    }
                }
            }
        }

        stage('Push Image to Registry') {
            steps {
                container('buildah') {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'REGISTRY_USER',
                        passwordVariable: 'REGISTRY_PASS'
                    )]) {
                        sh """
                            buildah pull docker-archive:./${env.TAR_FILE}
                            buildah tag ${env.IMAGE_NAME}:${env.IMAGE_TAG} ${env.REGISTRY}/${env.IMAGE_NAME}:${env.IMAGE_TAG}
                            buildah tag ${env.IMAGE_NAME}:${env.IMAGE_TAG} ${env.REGISTRY}/${env.IMAGE_NAME}:latest
                            buildah login -u ${REGISTRY_USER} -p ${REGISTRY_PASS} ${env.REGISTRY}
                            buildah push ${env.REGISTRY}/${env.IMAGE_NAME}:${env.IMAGE_TAG}
                            buildah push ${env.REGISTRY}/${env.IMAGE_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Register Build Artifact') {
            steps {
                registerBuildArtifactMetadata(
                    name: "hackers-api",
                    url: "docker.io/${env.IMAGE_NAME}:${env.IMAGE_TAG}",
                    version: "${env.IMAGE_TAG}",
                    digest: "${GIT_COMMIT}",
                    label: "demo",
                    type: "container"
                )
            }
        }
    }
}
