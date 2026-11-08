pipeline {
    agent any

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()
    }

    environment {
        ARTIFACTORY_URL  = 'http://192.168.2.10:8082/artifactory'
        ARTIFACTORY_REPO = 'im6ull-local'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Inspect') {
            steps {
                sh '''
                    echo "branch : $(git rev-parse --abbrev-ref HEAD)"
                    echo "commit : $(git rev-parse --short HEAD)"
                    echo "tracked: $(git ls-files | wc -l) files"
                '''
            }
        }

        stage('Package') {
            steps {
                sh '''
                    set -e
                    SHORT=$(git rev-parse --short HEAD)
                    VER="v$(date +%Y%m%d)-${SHORT}"
                    OUT="output/${VER}"
                    mkdir -p "$OUT"
                    git archive --format=tar.gz --prefix="im6ull-${VER}/" \
                        -o "$OUT/im6ull-${VER}.tar.gz" HEAD
                    {
                        echo "version=${VER}"
                        echo "commit=$(git rev-parse HEAD)"
                        echo "branch=$(git rev-parse --abbrev-ref HEAD)"
                        echo "built=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
                        echo "jenkins=${BUILD_NUMBER}"
                        sha256sum "$OUT"/*.tar.gz
                    } > "$OUT/manifest.txt"
                    cat "$OUT/manifest.txt"
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: 'output/**', fingerprint: true
                }
            }
        }

        stage('Upload to JFrog') {
            when {
                expression {
                    // 由 Jenkins 参数控制；Artifactory 未就绪时默认跳过
                    return params.UPLOAD_TO_JFROG
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'artifactory-admin',
                    usernameVariable: 'ART_USER',
                    passwordVariable: 'ART_PASS'
                )]) {
                    sh '''
                        set -e
                        SHORT=$(git rev-parse --short HEAD)
                        VER="v$(date +%Y%m%d)-${SHORT}"
                        OUT="output/${VER}"
                        curl -fsS -u "${ART_USER}:${ART_PASS}" \
                            -T "${OUT}/im6ull-${VER}.tar.gz" \
                            "${ARTIFACTORY_URL}/${ARTIFACTORY_REPO}/im6ull/${VER}/im6ull-${VER}.tar.gz"
                        curl -fsS -u "${ART_USER}:${ART_PASS}" \
                            -T "${OUT}/manifest.txt" \
                            "${ARTIFACTORY_URL}/${ARTIFACTORY_REPO}/im6ull/${VER}/manifest.txt"
                        echo "uploaded to ${ARTIFACTORY_URL}/${ARTIFACTORY_REPO}/im6ull/${VER}/"
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
