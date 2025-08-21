pipeline {
    agent any

    environment {
        SNAP_REPO           = 'app-snapshot'
        NEXUS_VERSION       = "nexus3"
        NEXUS_PROTOCOL      = "http"
        NEXUS_URL           = "172.31.94.48:8081"
        NEXUS_REPOSITORY    = "app-release"
        NEXUS_REPOGRP_ID    = "app-group"
        NEXUS_CREDENTIAL_ID = "nexuslogin"
        ARTVERSION          = "${env.BUILD_ID}"
        SONARSERVER         = 'sonarserver'
        SONARSCANNER        = 'sonarscanner'
    }

    stages {
        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
            environment {
                scannerHome = tool 'sonarscanner'
            }
            steps {
                withSonarQubeEnv("${SONARSERVER}") {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile-repo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                // Uncomment if you want to enforce quality gate
                // timeout(time: 10, unit: 'MINUTES') {
                //    waitForQualityGate abortPipeline: true
                // }
            }
        }

        stage("Publish to Nexus Repository Manager") {
            steps {
                script {
                    // Extract project details from pom.xml using Maven help:evaluate
                    def groupId    = sh(script: "mvn help:evaluate -Dexpression=project.groupId -q -DforceStdout", returnStdout: true).trim()
                    def artifactId = sh(script: "mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout", returnStdout: true).trim()
                    def version    = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                    def packaging  = sh(script: "mvn help:evaluate -Dexpression=project.packaging -q -DforceStdout", returnStdout: true).trim()

                    echo "Project Info → ${groupId}:${artifactId}:${version} (${packaging})"

                    // Find artifact
                    filesByGlob = findFiles(glob: "target/*.${packaging}")
                    artifactPath = filesByGlob[0].path

                    if (fileExists(artifactPath)) {
                        echo "*** File: ${artifactPath}, group: ${groupId}, packaging: ${packaging}, version ${version}, ARTVERSION ${ARTVERSION}"
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: NEXUS_REPOGRP_ID,
                            version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: artifactId,
                                 classifier: '',
                                 file: artifactPath,
                                 type: packaging],
                                [artifactId: artifactId,
                                 classifier: '',
                                 file: "pom.xml",
                                 type: "pom"]
                            ]
                        )
                    } else {
                        error "*** File: ${artifactPath}, could not be found"
                    }
                }
            }
        }
    }
}

