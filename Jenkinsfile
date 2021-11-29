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
                image.inside {
                    sh './gradlew ktlintCheck'
                }
            }
        }

        stage('Lint analysis') {
            steps {
                image.inside {
                    sh './gradlew lintDebug'
                }
            }
        }

        stage('Distribute Aptito KDS .apk') {
            when {
                branch 'development'
            }
            steps {
                withCredentials (
                    bindings: [
                        file (
                              credentialsId: 'appDistributionCredential',
                              variable: 'GOOGLE_APPLICATION_CREDENTIALS'
                        )
                    ]
                )
                {
                    withEnv(["CHANGE_BRANCH=$env.BUILD_NUMBER"]) {
                        sh 'echo ${LAST_COMMITS} > releasenotes.txt'
                        image.inside {
                            sh './gradlew assembleDebug appDistributionUploadDebug'
                        }
                    }
                }
                archiveArtifacts '**/app/build/outputs/apk/debug/*.apk'
            }
        }
    }
    post {
        failure {
            slackSend(channel: '#ci_cd_status', color: 'danger', message: "$AUTHOR_NAME $env.CHANGE_BRANCH - Build # $BUILD_NUMBER - Failure:Check console output at $BUILD_URL to view the results.")
        }
        always {
            sh "docker rmi ${image.id}"
            sh "docker system prune -a -f"
        }
    }

    options {
        skipStagesAfterUnstable()
    }
}
