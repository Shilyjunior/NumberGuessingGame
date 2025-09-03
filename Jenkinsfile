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
        WAR_NAME = 'NumberGuessGame'
    }
    stages {
        stage('Checkout') {
            steps { 
                checkout scm 
            }
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
                always { 
                    script {
                        // Check if test results exist before trying to publish them
                        def testResults = sh(script: "find . -name 'surefire-reports' -type d", returnStdout: true).trim()
                        if (testResults) {
                            junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml'
                        } else {
                            echo 'No test reports found - skipping junit publisher'
                        }
                    }
                }
            }
        }
        stage('Package WAR') {
            steps {
                sh "mvn ${env.MAVEN_ARGS} package -DskipTests"
            }
        }
        stage('Archive Artifact') {
            steps { 
                archiveArtifacts artifacts: 'target/*.war', fingerprint: true, allowEmptyArchive: false
            }
        }
        stage('Pre-deployment Check') {
            steps {
                script {
                    // Verify Tomcat installation exists
                    sh """
                        echo 'Checking Tomcat installation...'
                        if [ ! -d "${TOMCAT_HOME}" ]; then
                            echo "ERROR: Tomcat directory not found at ${TOMCAT_HOME}"
                            echo "Please ensure Tomcat is installed at the specified location"
                            exit 1
                        fi
                        
                        if [ ! -f "${TOMCAT_HOME}/bin/startup.sh" ]; then
                            echo "ERROR: Tomcat startup script not found"
                            exit 1
                        fi
                        
                        if [ ! -f "${TOMCAT_HOME}/bin/shutdown.sh" ]; then
                            echo "ERROR: Tomcat shutdown script not found"
                            exit 1
                        fi
                        
                        if [ ! -d "${TOMCAT_WEBAPPS}" ]; then
                            echo "ERROR: Tomcat webapps directory not found at ${TOMCAT_WEBAPPS}"
                            exit 1
                        fi
                        
                        # Check if scripts are executable
                        if [ ! -x "${TOMCAT_HOME}/bin/startup.sh" ]; then
                            echo "Making Tomcat scripts executable..."
                            chmod +x ${TOMCAT_HOME}/bin/*.sh
                        fi
                        
                        echo '✅ Tomcat installation verified'
                        ls -la ${TOMCAT_HOME}/bin/
                        ls -la ${TOMCAT_WEBAPPS}/
                    """
                }
            }
        }
        stage('Deploy to Tomcat') {
            steps {
                script {
                    sh """
                        echo 'Starting deployment process...'
                        
                        # Stop Tomcat gracefully
                        echo 'Stopping Tomcat...'
                        if pgrep -f "tomcat" > /dev/null; then
                            ${TOMCAT_HOME}/bin/shutdown.sh || true
                            echo 'Waiting for Tomcat to stop...'
                            sleep 10
                            
                            # Force kill if still running
                            if pgrep -f "tomcat" > /dev/null; then
                                echo 'Force stopping Tomcat processes...'
                                pkill -f tomcat || true
                                sleep 5
                            fi
                        else
                            echo 'Tomcat is not running'
                        fi
                        
                        # Clean up old deployment
                        echo 'Cleaning up old deployment...'
                        rm -rf ${TOMCAT_WEBAPPS}/${WAR_NAME}*
                        
                        # Verify WAR file exists before deployment
                        if [ ! -f "target/${WAR_NAME}-1.0-SNAPSHOT.war" ]; then
                            echo "ERROR: WAR file not found in target directory"
                            ls -la target/
                            exit 1
                        fi
                        
                        # Deploy new WAR
                        echo 'Deploying new WAR file...'
                        cp target/${WAR_NAME}-1.0-SNAPSHOT.war ${TOMCAT_WEBAPPS}/${WAR_NAME}.war
                        
                        # Verify deployment file was copied
                        if [ ! -f "${TOMCAT_WEBAPPS}/${WAR_NAME}.war" ]; then
                            echo "ERROR: Failed to copy WAR file to webapps directory"
                            exit 1
                        fi
                        
                        echo "WAR file deployed successfully:"
                        ls -la ${TOMCAT_WEBAPPS}/${WAR_NAME}.war
                        
                        # Start Tomcat
                        echo 'Starting Tomcat...'
                        ${TOMCAT_HOME}/bin/startup.sh
                        
                        # Wait for Tomcat to start
                        echo 'Waiting for Tomcat to start...'
                        sleep 15
                        
                        # Check if Tomcat is running
                        if pgrep -f "tomcat" > /dev/null; then
                            echo '✅ Tomcat started successfully'
                        else
                            echo '❌ Tomcat failed to start'
                            echo 'Checking Tomcat logs...'
                            tail -20 ${TOMCAT_HOME}/logs/catalina.out || true
                            exit 1
                        fi
                        
                        # Wait for application deployment
                        echo 'Waiting for application deployment...'
                        sleep 10
                        
                        # Verify deployment
                        if [ -d "${TOMCAT_WEBAPPS}/${WAR_NAME}" ]; then
                            echo '✅ Application deployed successfully!'
                            echo "Application should be available at: http://localhost:8080/${WAR_NAME}"
                        else
                            echo '⚠️  Application directory not found - deployment may still be in progress'
                            echo 'Please check Tomcat logs for any deployment issues'
                        fi
                        
                        echo 'Deployment process completed!'
                    """
                }
            }
        }
    }
    post {
        success {
            script {
                echo '✅ Build & Deployment successful!'
                echo "Application URL: http://localhost:8080/${WAR_NAME}"
            }
        }
        failure {
            script {
                echo '❌ Build or Deployment failed. Check logs above for details.'
                // Try to show Tomcat logs if deployment stage failed
                sh """
                    if [ -f "${TOMCAT_HOME}/logs/catalina.out" ]; then
                        echo "=== Recent Tomcat Logs ==="
                        tail -50 ${TOMCAT_HOME}/logs/catalina.out || true
                    fi
                """ 
            }
        }
        always {
            script {
                // Only clean workspace if the plugin is available
                try {
                    cleanWs(deleteDirs: true, notFailBuild: true)
                } catch (Exception e) {
                    echo "Warning: cleanWs not available - workspace cleanup skipped"
                    echo "Consider installing the 'Workspace Cleanup Plugin'"
                }
            }
        }
    }
}
