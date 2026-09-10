pipeline {
    agent none

    options {
        disableConcurrentBuilds()
        timestamps()
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

            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        /opt/maven/bin/mvn -B \
                          clean verify \
                          org.sonarsource.scanner.maven:sonar-maven-plugin:sonar
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
