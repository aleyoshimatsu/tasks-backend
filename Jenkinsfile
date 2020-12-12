pipeline {
    agent any

    stages {
        stage ('Build Backend') {
            steps {
                withEnv(["JAVA_HOME=${ tool 'JDK 8' }", "PATH+MAVEN=${tool 'MVN 3.6.3'}/bin:${env.JAVA_HOME}/bin"]) {
                    sh 'mvn clean package -DskipTests=true'
                }
            }
        }
        stage ('Unit Tests') {
            steps {
                withEnv(["JAVA_HOME=${ tool 'JDK 8' }", "PATH+MAVEN=${tool 'MVN 3.6.3'}/bin:${env.JAVA_HOME}/bin"]) {
                    sh 'mvn test'
                }
            }
        }
        stage ('Sonar Analysis') {
            environment {
                scannerHome = tool 'SONAR_SCANNER'
            }
            steps {
                withSonarQubeEnv('Sonar') {
                    sh "${scannerHome}/bin/sonar-scanner -e -Dsonar.projectKey=DeployBack -Dsonar.host.url=http://sonarqube:9000 -Dsonar.login=cf6826d57f1e453e08ecbd6cf862496472061f66 -Dsonar.java.binaries=target -Dsonar.coverage.exclusions=**/.mvn/**,**/src/test/**,**/model/**,**Application.java"
                }
            }
        }
        stage ('Quality Gate') {
            steps {
                sleep(5)
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage ('Deploy Backend') {
            steps {
                deploy adapters: [tomcat8(credentialsId: 'admin', path: '', url: 'http://tomcat:8080/')], contextPath: 'tasks-backend', war: 'target/tasks-backend.war'
            }
        }
        stage ('API Test') {
            steps {
                withEnv(["JAVA_HOME=${ tool 'JDK 8' }", "PATH+MAVEN=${tool 'MVN 3.6.3'}/bin:${env.JAVA_HOME}/bin"]) {
                    dir('api-test') {
                        git credentialsId: 'SSH', url: 'https://github.com/aleyoshimatsu/tasks-api-test'
                        sh 'mvn test'
                    }
                }
            }
        }
        stage ('Deploy Frontend') {
            steps {
                withEnv(["JAVA_HOME=${ tool 'JDK 8' }", "PATH+MAVEN=${tool 'MVN 3.6.3'}/bin:${env.JAVA_HOME}/bin"]) {
                    dir('frontend') {
                        git credentialsId: 'SSH', url: 'https://github.com/aleyoshimatsu/tasks-frontend'
                        sh 'mvn clean package'
                        deploy adapters: [tomcat8(credentialsId: 'admin', path: '', url: 'http://tomcat:8080/')], contextPath: 'tasks', war: 'target/tasks.war'
                    }
                }
            }
        }
        stage ('Functional Test') {
            steps {
                withEnv(["JAVA_HOME=${ tool 'JDK 8' }", "PATH+MAVEN=${tool 'MVN 3.6.3'}/bin:${env.JAVA_HOME}/bin"]) {
                    dir('functional-test') {
                        git credentialsId: 'SSH', url: 'https://github.com/aleyoshimatsu/tasks-functional-tests'
                        sh 'mvn test'
                    }
                }
            }
        }
        stage('Deploy Prod') {
            steps {
                sh 'docker-compose build'
                sh 'docker-compose up -d'
            }
        }
        stage ('Health Check') {
            steps {
                withEnv(["JAVA_HOME=${ tool 'JDK 8' }", "PATH+MAVEN=${tool 'MVN 3.6.3'}/bin:${env.JAVA_HOME}/bin"]) {
                    sleep(5)
                    dir('functional-test') {
                        sh 'mvn verify -Dskip.surefire.tests'
                    }
                }
            }
        }
    }
    post {
        always {
            junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml, api-test/target/surefire-reports/*.xml, functional-test/target/surefire-reports/*.xml, functional-test/target/failsafe-reports/*.xml'
            archiveArtifacts artifacts: 'target/tasks-backend.war, frontend/target/tasks.war', onlyIfSuccessful: true
        }
        unsuccessful {
            emailext attachLog: true, body: 'See the attached log below', subject: 'Build $BUILD_NUMBER has failed', to: 'aleyoshimatsu+jenkins@gmail.com'
        }
        fixed {
            emailext attachLog: true, body: 'See the attached log below', subject: 'Build is fine!!!', to: 'aleyoshimatsu+jenkins@gmail.com'
        }
    }

}
