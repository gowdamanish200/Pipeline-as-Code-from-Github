pipeline {
    agent any
    environment {
        // Define environment variables
        PROJECT_NAME = 'SampleProject'
        DEPLOY_ENV = 'production'
        SLACK_CHANNEL = '#build-notifications'
        BUILD_DIR = 'build'
    }
    options {
        // Pipeline-level options
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }
    triggers {
        // Periodic build trigger (nightly)
        cron('H H(0-3) * * *')
    }
    stages {
        stage('Initialize') {
            steps {
                script {
                    // Clean workspace before starting
                    deleteDir()
                }
                echo "Initializing build for ${PROJECT_NAME} in ${DEPLOY_ENV} environment."
            }
        }
        
        stage('Checkout Code') {
            steps {
                // Checkout from source control (e.g., Git)
                git branch: 'main', url: 'https://github.com/user/repository.git'
            }
        }
        
        stage('Build') {
            parallel {
                stage('Build Backend') {
                    agent { label 'backend-builder' }
                    steps {
                        dir('backend') {
                            sh './gradlew build'
                        }
                    }
                }
                stage('Build Frontend') {
                    agent { label 'frontend-builder' }
                    steps {
                        dir('frontend') {
                            sh 'npm install'
                            sh 'npm run build'
                        }
                    }
                }
            }
        }
        
        stage('Unit Tests') {
            parallel {
                stage('Backend Tests') {
                    steps {
                        dir('backend') {
                            sh './gradlew test'
                        }
                    }
                    post {
                        always {
                            junit '**/backend/build/test-results/test/*.xml'
                        }
                    }
                }
                stage('Frontend Tests') {
                    steps {
                        dir('frontend') {
                            sh 'npm run test'
                        }
                    }
                    post {
                        always {
                            junit '**/frontend/test-results/**/*.xml'
                        }
                    }
                }
            }
        }
        
        stage('Integration Tests') {
            steps {
                script {
                    // Only run integration tests on main branch
                    if (env.BRANCH_NAME == 'main') {
                        echo "Running integration tests"
                        sh './scripts/run_integration_tests.sh'
                    } else {
                        echo "Skipping integration tests on non-main branches"
                    }
                }
            }
        }
        
        stage('Package') {
            steps {
                dir('backend') {
                    sh './gradlew assemble'
                    archiveArtifacts artifacts: 'build/libs/*.jar', fingerprint: true
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                script {
                    if (DEPLOY_ENV == 'production') {
                        echo "Deploying to production environment."
                        sh './scripts/deploy.sh production'
                    } else {
                        echo "Deploying to staging environment."
                        sh './scripts/deploy.sh staging'
                    }
                }
            }
        }
    }
    post {
        success {
            echo 'Build completed successfully.'
            slackSend(channel: "${SLACK_CHANNEL}", message: "SUCCESS: ${PROJECT_NAME} build ${env.BUILD_NUMBER}")
        }
        failure {
            echo 'Build failed.'
            slackSend(channel: "${SLACK_CHANNEL}", message: "FAILURE: ${PROJECT_NAME} build ${env.BUILD_NUMBER}")
        }
        always {
            cleanWs()
        }
    }
}
