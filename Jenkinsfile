pipeline {
    // All stages run on the docker-agent node
    agent {
        label 'docker-agent'
    }

    environment {
        // Must be localhost because of DooD 
        NEXUS_MR_REGISTRY   = 'localhost:5000'
        NEXUS_MAIN_REGISTRY = 'localhost:5001'

        // Short Git commit SHA — used as the image tag
        SHORT_SHA = sh(
            script: 'git rev-parse --short=8 HEAD',
            returnStdout: true
        ).trim()
    }

    options {
        // Keep only the last 10 builds to save disk space
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Timestamp every log line
        timestamps()
        // Abort if a stage takes more than 30 minutes
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {

        // ----------------------------------------------------------------
        // MR / Pull Request pipeline
        // Runs when this build is triggered by a pull/merge request
        // ----------------------------------------------------------------

        stage('Checkstyle') {
            when { changeRequest() }
            steps {
                // Pull a Maven container — no Maven installed on the agent itself
                docker.image('maven:3.9-eclipse-temurin-21').inside('-v $HOME/.m2:/root/.m2') {
                    sh './mvnw checkstyle:checkstyle -q'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: '**/checkstyle-result.xml',
                                     allowEmptyArchive: true
                    recordIssues(
                        tools: [checkStyle(pattern: '**/checkstyle-result.xml')],
                        qualityGates: [[threshold: 1, type: 'TOTAL_HIGH', unstable: true]]
                    )
                }
            }
        }

        stage('Test') {
            when { changeRequest() }
            steps {
                docker.image('maven:3.9-eclipse-temurin-21').inside('-v $HOME/.m2:/root/.m2') {
                    sh './mvnw test -q'
                }
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Build') {
            when { changeRequest() }
            steps {
                docker.image('maven:3.9-eclipse-temurin-21').inside('-v $HOME/.m2:/root/.m2') {
                    // Package without running tests (tests already ran above)
                    sh './mvnw package -DskipTests -q'
                }
            }
        }

        stage('Docker build & push — MR') {
            when { changeRequest() }
            steps {
                script {
                    docker.withRegistry(
                        "http://${env.NEXUS_MR_REGISTRY}",
                        'nexus-credentials'
                    ) {
                        def image = docker.build(
                            "${env.NEXUS_MR_REGISTRY}/mr:${env.SHORT_SHA}",
                            '--file Dockerfile .'
                        )
                        image.push()
                        // Clean up local image to save agent disk space
                        sh "docker rmi ${env.NEXUS_MR_REGISTRY}/mr:${env.SHORT_SHA} || true"
                    }
                }
            }
        }

        // ----------------------------------------------------------------
        // Main branch pipeline
        // Runs only when commits land on the main branch
        // ----------------------------------------------------------------

        stage('Docker build & push — main') {
            when { branch 'main' }
            steps {
                script {
                    docker.withRegistry(
                        "http://${env.NEXUS_MAIN_REGISTRY}",
                        'nexus-credentials'
                    ) {
                        def image = docker.build(
                            "${env.NEXUS_MAIN_REGISTRY}/main:${env.SHORT_SHA}",
                            '--file Dockerfile .'
                        )
                        image.push()
                        // Also tag as latest for convenience
                        image.push('latest')
                        sh "docker rmi ${env.NEXUS_MAIN_REGISTRY}/main:${env.SHORT_SHA} || true"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully. Image tag: ${env.SHORT_SHA}"
        }
        failure {
            echo "Pipeline failed. Check stage logs above."
        }
        cleanup {
            // Always clean the workspace after the build
            cleanWs()
        }
    }
}
