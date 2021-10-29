#! /usr/bin/env groovy

pipeline {
    agent any

    environment {
        AUTHOR_NAME = sh(script: "git --no-pager show -s --format='%ae'", returnStdout: true).trim()
        LAST_COMMITS = sh( script: 'git --no-pager log -5 --pretty="%ad: %s"', returnStdout: true ).toString()
    }

    stages {
        stage('Initialization') {
            steps {
                script {
                    buildName env.CHANGE_BRANCH
                }
                script {
                    image = docker.build("image:$env.BUILD_NUMBER")
                }
            }
        }
        stage('KtLint check') {
            steps {
                script {
                    image.inside {
                        sh './gradlew ktlintCheck'
                    }
                }
            }
        }

        stage('Lint analysis') {
            steps {
                script {
                    image.inside {
                        sh './gradlew lint'
                    }
                }
            }
        }

        stage('Distribute Clover KDS (Aptito) .apk') {
            when {
                branch 'development'
            }
            steps {
                script {
                    withCredentials (
                    bindings: [
                        file (
                              credentialsId: 'appDistributionCredential',
                              variable: 'GOOGLE_APPLICATION_CREDENTIALS'
                        )
                    ]
                )
                    {
                        withEnv(["BUILD_NAME=$env.BUILD_NUMBER"]) {
                            sh 'echo ${LAST_COMMITS} > releasenotes.txt'
                            image.inside {
                                sh './gradlew assembleAptitoDebug appDistributionUploadAptitoDebug'
                            }
                        }
                    }
                }
            }
        }

        stage('Distribute Clover KDS (Order Out) .apk') {
            when {
                branch 'development'
            }
            steps {
                script {
                    withCredentials (
                        bindings: [
                           file (
                                  credentialsId: 'appDistributionCredential',
                                  variable: 'GOOGLE_APPLICATION_CREDENTIALS'
                            )
                        ]
                    )
                    {
                        withEnv(["BUILD_NAME=$env.BUILD_NUMBER"]) {
                            sh 'echo ${LAST_COMMITS} > releasenotes.txt'
                            image.inside {
                                sh './gradlew assembleOrderoutDebug appDistributionUploadOrderoutDebug'
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        failure {
            slackSend(channel: '#ci_cd_status', color: 'danger', message: "$AUTHOR_NAME $env.CHANGE_BRANCH - Build # $BUILD_NUMBER - Failure:Check console output at $BUILD_URL to view the results.")
        }
        always {
            images = docker.image("image:$env.BUILD_NUMBER")
            sh "docker rmi ${images.id}"
        }
    }

    options {
        skipStagesAfterUnstable()
    }
}
