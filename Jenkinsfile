pipeline {
    agent any
    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        durabilityHint('PERFORMANCE_OPTIMIZED')
        timeout(time: 30, unit: 'MINUTES')
    }
    environment {
        MAVEN_ARGS = '-B -ntp'
        TOMCAT_HOME = '/ec2-user/apache-tomcat-7.0.94'
        TOMCAT_WEBAPPS = "${TOMCAT_HOME}/webapps"
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Build & Test') {
            steps {
                script {
                    sh "mvn ${env.MAVEN_ARGS} --version"
                    sh "mvn ${env.MAVEN_ARGS} clean validate compile -DskipTests"
                    sh "mvn ${env.MAVEN_ARGS} test"
                }
            }
            post {
                always { junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml' }
            }
        }
        stage('Package WAR') {
            steps {
                sh "mvn ${env.MAVEN_ARGS} package -DskipTests"
            }
        }
        stage('Archive Artifact') {
            steps { archiveArtifacts artifacts: 'target/*.war', fingerprint: true }
        }
        stage('Deploy to Tomcat') {
            steps {
                sh """
                    echo 'Stopping Tomcat...'
                    ${TOMCAT_HOME}/bin/shutdown.sh || true
                    echo 'Deploying WAR...'
                    cp target/NumberGuessGame-1.0-SNAPSHOT.war ${TOMCAT_WEBAPPS}/NumberGuessGame.war
                    echo 'Starting Tomcat...'
                    ${TOMCAT_HOME}/bin/startup.sh
                    echo 'Deployment complete!'
                """
            }
        }
    }
    post {
        success {
            echo ':white_check_mark: Build & Deployment successful!'
        }
        failure {
            echo ':x: Build or Deployment failed. Check logs.'
        }
        always {
            cleanWs(deleteDirs: true, notFailBuild: true)
        }
    }
}
