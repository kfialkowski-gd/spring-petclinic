pipeline {
    agent none

    environment {
        NEXUS_MR_REGISTRY   = 'localhost:5000'
        NEXUS_MAIN_REGISTRY = 'localhost:5001'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Init') {
            agent { label 'docker-agent' }
            steps {
                script {
                    env.SHORT_SHA = sh(
                        script: 'git rev-parse --short=8 HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Checkstyle') {
            when { changeRequest() }
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-21'
                    label 'docker-agent'
                    args '-v $HOME/.m2:/root/.m2'
                    reuseNode true
                }
            }
            steps {
                sh './mvnw checkstyle:checkstyle -q'
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
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-21'
                    label 'docker-agent'
                    args '-v $HOME/.m2:/root/.m2'
                    reuseNode true
                }
            }
            steps {
                sh './mvnw test -q'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Build') {
            when { changeRequest() }
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-21'
                    label 'docker-agent'
                    args '-v $HOME/.m2:/root/.m2'
                    reuseNode true
                }
            }
            steps {
                sh './mvnw package -DskipTests -q'
            }
        }

        stage('Docker build & push — MR') {
            when { changeRequest() }
            agent { label 'docker-agent' }
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
                        sh "docker rmi ${env.NEXUS_MR_REGISTRY}/mr:${env.SHORT_SHA} || true"
                    }
                }
            }
        }

        stage('Docker build & push — main') {
            when { branch 'main' }
            agent { label 'docker-agent' }
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
            cleanWs()
        }
    }
}
