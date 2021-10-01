#! /usr/bin/env groovy 

import java.util.regex.Pattern


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
        stage('Compile'){
            steps{
                script{
                    image.inside {
                        sh './gradlew assembleDebug' 
                    }
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

        stage('Unit tests') {
            steps {
                script {
                    image.inside {
                        sh './gradlew testDebugUnitTest testgDebugUnitTest'
                    }
                }
            }
        }

        stage('Lint analysis') {
            steps {
                script {
                    image.inside {
                        sh './gradlew lintDebug'
                    }
                }
            }
        }

        stage('Build APK') {
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
                    script {
                        def featureName = Pattern
                        .compile("\\[FEATURE.+\\]|\\[BUGFIX.+\\]|\\[HOTFIX.+\\]")
                        .matcher(sh(script: 'git --no-pager log -5 --pretty="%ad: %s"', returnStdout: true)
                        .toString())
                        .findAll()
                        .first()
                        withEnv(["CHANGE_BRANCH=${featureName}"]) {
                            sh 'echo ${LAST_COMMITS} > releasenotes.txt'
                            image.inside {
                                sh './gradlew assembleDebug appDistributionUploadDebug'
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'master'
            }
            environment {
                SIGNING_KEYSTORE = credentials('my-app-signing-keystore')
                SIGNING_KEY_PASSWORD = credentials('my-app-signing-password')
            }
            post {
                success {
                    mail(to: 'beta-testers@example.com', subject: 'New build available!', body: 'Check it out!')
                }
            }
            steps {
                sh './gradlew assembleRelease'
                archiveArtifacts '**/*.apk'
                androidApkUpload(googleCredentialsId: 'Google Play', apkFilesPattern: '**/*-release.apk', trackName: 'beta')
            }
        }
    }
    post {
        failure {
            slackSend(channel: '#ci_cd_status', color: 'danger', message: "$AUTHOR_NAME $env.CHANGE_BRANCH - Build # $BUILD_NUMBER - Failure:Check console output at $BUILD_URL to view the results.")
        }
        always {
            sh "docker rmi ${image.id}"
        }
    }
    options {
        skipStagesAfterUnstable()
    }
}
