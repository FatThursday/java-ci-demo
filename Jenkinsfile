pipeline {
    agent none

    options {
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            agent {
                label 'maven'
            }

            steps {
                checkout scm
            }
        }

        stage('Build Test Sonar') {
            agent {
                label 'maven'
            }

            environment {
                ARTIFACTORY_CREDS = credentials('artifactory-deployer')
            }

            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        set +x

                        test -n "$ARTIFACTORY_CREDS_USR" || {
                            echo "ERROR: Artifactory username not injected"
                            exit 1
                        }

                        test -n "$ARTIFACTORY_CREDS_PSW" || {
                            echo "ERROR: Artifactory password not injected"
                            exit 1
                        }

                        export ARTIFACTORY_USERNAME="$ARTIFACTORY_CREDS_USR"
                        export ARTIFACTORY_PASSWORD="$ARTIFACTORY_CREDS_PSW"

                        echo "Testing authenticated Artifactory access..."

                        curl -fsS \
                          -u "$ARTIFACTORY_USERNAME:$ARTIFACTORY_PASSWORD" \
                          "$ARTIFACTORY_URL/libs/org/sonarsource/scanner/maven/sonar-maven-plugin/5.8.0.7211/sonar-maven-plugin-5.8.0.7211.pom" \
                          >/dev/null

                        echo "Artifactory authentication OK"

                        /opt/maven/bin/mvn -B \
                          clean verify \
                          org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar
                    '''
                }
            }

            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml',
                          allowEmptyResults: true
                }
            }
        }

        stage('Quality Gate') {
            agent none

            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Deploy') {
            agent {
                label 'maven'
            }

            environment {
                ARTIFACTORY_CREDS = credentials('artifactory-deployer')
            }

            steps {
                sh '''
                    set +x

                    export ARTIFACTORY_USERNAME="$ARTIFACTORY_CREDS_USR"
                    export ARTIFACTORY_PASSWORD="$ARTIFACTORY_CREDS_PSW"

                    /opt/maven/bin/mvn -B \
                      -DskipTests \
                      -Dsonar.skip=true \
                      deploy
                '''
            }
        }
    }

    post {
        always {
            echo "Build result: ${currentBuild.currentResult}"
        }
    }
}