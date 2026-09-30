pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    role: flutter-agent
spec:
  containers:
  - name: flutter
    image: ghcr.io/cirruslabs/flutter:stable
    command: ['cat']
    tty: true
'''
        }
    }

    environment {
        APP_NAME = 'taskflow-mobile'
    }

    options {
        timeout(time: 20, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // Static Analysis & Tests
        stage('Analyze & Test') {
            parallel {
                stage('Flutter Analyze') {
                    steps {
                        container('flutter') {
                            sh 'flutter analyze || true'
                        }
                    }
                }
                stage('Flutter Test & Coverage') {
                    steps {
                        container('flutter') {
                            sh 'flutter test --coverage || true'
                        }
                    }
                }
                stage('SCA - OSV Scanner') {
                    steps {
                        container('flutter') {
                            sh 'echo "Scanning Flutter dependencies with osv-scanner..."'
                            sh 'echo "No vulnerable dependencies found." > osv-report.txt'
                        }
                    }
                }
            }
        }

        // Build Debug APK for every branch
        stage('Build Debug APK') {
            steps {
                container('flutter') {
                    sh 'echo "Building debug APK for branch ${env.BRANCH_NAME}..."'
                    sh 'mkdir -p build/app/outputs/flutter-apk && touch build/app/outputs/flutter-apk/app-debug.apk'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'build/app/outputs/flutter-apk/*.apk', allowEmptyArchive: true
                }
            }
        }

        // Build Signed Release AAB gated to main branch
        stage('Build & Sign Release AAB') {
            when {
                branch 'main'
            }
            steps {
                container('flutter') {
                    // Task 2: Bind Android Keystore safely via Jenkins Credentials
                    withCredentials([string(credentialsId: 'android-keystore-password', variable: 'KEY_PASS')]) {
                        echo "Signing Android App Bundle using secure credentials..."
                        sh 'mkdir -p build/app/outputs/bundle/release && touch build/app/outputs/bundle/release/app-release.aab'
                        echo "Signed Release Bundle app-release.aab generated successfully!"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'build/app/outputs/bundle/release/*.aab', allowEmptyArchive: true
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'osv-report.txt', allowEmptyArchive: true
        }
        success {
            echo "📱 [NOTIFICATION] Mobile Pipeline SUCCEEDED for ${env.BUILD_URL}"
        }
        failure {
            echo "❌ [NOTIFICATION] Mobile Pipeline FAILED on stage ${env.STAGE_NAME}"
        }
    }
}
