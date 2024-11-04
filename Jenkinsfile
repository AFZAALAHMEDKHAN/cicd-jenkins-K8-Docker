pipeline {
    agent any

    environment {
        registry = "afzaalahmedkhan/vproapp"
        registryCredential = "dockerhub"
        NEXUSIP = "172.31.20.92"
        NEXUSPORT = "8081"
        RELEASE_REPO = 'vprofile-release'
        NEXUS_LOGIN = "nexuslogin"
    }
    
    stages {
        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success{
                    echo "Now Archiving...."
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('CHECKSTYLE ANALYSIS'){
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
                scannerHome = tool 'Sonar6.2'
            }

            steps {
                withSonarQubeEnv('sonarserver') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Upload the artifact to Nexus') {
            steps {
                nexusArtifactUploader(
                nexusVersion: 'nexus3',
                protocol: 'http',
                nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                groupId: 'QA',
                version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                repository: "${RELEASE_REPO}",
                credentialsId: "${NEXUS_LOGIN}",
                artifacts: [
                    [artifactId: 'vproapp',
                     classifier: '',
                     file: 'target/vprofile-v2.war',
                     type: 'war']
                    ]
                )
            }
        }


        stage('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build registry + ":V$BUILD_NUMBER"
                }
            }
        }
        stage('Upload Image') {
            steps {
                script {
                    docker.withRegistry('',registryCredential) {
                        // '' is for the registry url, leaving empty = default dockerhub
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage('Remove Unused docker images') {
            steps{
                sh "docker rmi $registry:V$BUILD_NUMBER"
            }
        }

        stage("Deploying on K8s") {
            agent {label 'KOPS'}
                steps{
                    sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appImage=${registry}:V${BUILD_NUMBER} --namespace production"
                }
        }
    }
}