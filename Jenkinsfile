pipeline {
    agent any
    tools {
        go 'go-1.25.8'
    }
    environment {
        GO111MODULE = 'on'
        IMAGE_NAME = "ketem/go-jenkins"
        PATH = "/home/ketem/go/bin:${env.PATH}"
        FULL_VERSION = ""
    }

    triggers {
        // pollSCM('* * * * *')
        githubPush()
    }

    parameters {
        // --- NEW: The String parameter for the Git branch! ---
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Which Git branch should we build?')

        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run tests before building the application')
        booleanParam(name: 'PUSH_DOCKER_IMAGE', defaultValue: true, description: 'Push the Docker image to the registry after building')
        choice(name: 'ENVIRONMENT', choices: ['development', 'staging', 'production'], description: 'Select the deployment environment')
    }

    stages {
        stage('Test') {
            when {
                expression { return params.RUN_TESTS == true }
            }
            steps {
                // Notice how we replaced 'main' with params.BRANCH_NAME
                git branch: params.BRANCH_NAME, url: 'https://github.com/ketdev-og/go-jenkins.git'
                sh 'go test -v ./...'
            }
        }

        stage('Lint & Format Check') {
            steps {
                script {
                    // Formatting check with gofmt
                    def fmtOutput = sh(script: 'gofmt -l .', returnStdout: true).trim()
                    if (fmtOutput) {
                        error "The following files are not properly formatted:\n${fmtOutput}\nPlease run 'gofmt -w .' to fix formatting issues."
                    } else {
                        echo 'All files are properly formatted.'
                    }
                }
            }
        }

    

        stage('Security Scan') {
            when {
                expression { return params.ENVIRONMENT == 'production' }
            }
            steps {
                // Now Jenkins can find 'gosec' because it's in the PATH
                sh 'gosec -version'
                sh 'gosec ./...'
            }
        }

        stage('Static Analytics (Go Vet)') {
            steps {
                script {
                    def vetOutput = sh(script: 'go vet ./...', returnStatus: true)
                    if (vetOutput != 0) {
                        error 'go vet found issues in the code. Please fix them before proceeding.'
                 } else {
                        echo 'go vet did not find any issues.'
                    }
                }
            }
        }

        stage('Versioning') {
            steps {
                script {
                    def shortCommit = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()

                    def timeStamp = sh(script: 'date -u +"%Y-%m-%dT%H:%M:%SZ"', returnStdout: true).trim()

                    FULL_VERSION = "${shortCommit}-b${env.BUILD_NUMBER}-${timeStamp}"
                    echo "The full version string is: ${FULL_VERSION}"

                }
            }
        }

        stage('Docker build') {
            when {
                expression { return params.PUSH_DOCKER_IMAGE == true }
            }
            steps {
                script {
                    // 1. Grab the 7-character Git commit hash
                    def SHORT_COMMIT = sh(script: 'git log -1 --format="%h"', returnStdout: true).trim()
                    echo "Building Docker image for commit: ${SHORT_COMMIT}"

                    def ACTUAL_BRANCH = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
                    echo "The current Git branch is: ${ACTUAL_BRANCH}"

                    def COMMIT_MESSAGE = sh(script: 'git log -1 --format="%s"', returnStdout: true).trim()
                    echo "We are building the feature: ${COMMIT_MESSAGE}"

                    // 2. Inject it dynamically into the Docker build command!
                    app = docker.build("${IMAGE_NAME}:${FULL_VERSION}")

                // Optional Pro-Tip: You can also tag this exact same build as 'latest'
                // app = docker.build("ketem/go-jenkins:latest")
                }
            }
        }

        stage('Build') {
            steps {
                // Use the branch parameter here too!
                git branch: params.BRANCH_NAME, url: 'https://github.com/ketdev-og/go-jenkins.git'

                // --- NEW: Print the last 5 Git commits neatly on one line each ---
                sh 'git log -n 5 --oneline'

                sh 'go build -o go-webapp-sample .'
                echo "the current build number is: ${env.BUILD_NUMBER}"
            }
        }

        stage('Security: Image Scan (Trivy)') {
            steps {
                script {
                    echo 'Scanning Docker image for OS vulnerabilities...'
                    // --exit-code 1 tells Trivy to fail the Jenkins build if issues are found
                    // --severity HIGH,CRITICAL ignores the small stuff and focuses on the big holes
                    sh "trivy image --exit-code 1 --severity HIGH,CRITICAL --timeout 20m --no-progress ${IMAGE_NAME}:${FULL_VERSION}"
                }
            }
        }

        stage('Run') {
            steps {
                sh 'JENKINS_NODE_COOKIE=dontKillMe nohup ./go-webapp-sample > app.log 2>&1 &'
            }
        }

        stage('Deploy') {
            steps {
                script {
                    if (params.ENVIRONMENT == 'production') {
                        echo 'Deploying to production environment...'
                    } else if (params.ENVIRONMENT == 'staging') {
                        echo 'Deploying to staging environment...'
                    } else {
                        echo 'Deploying to development environment...'
                    }
                }
            }
        }
    }
}
