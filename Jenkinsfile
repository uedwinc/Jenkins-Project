def COLOR_MAP = [ //This def is used to define variables for slack notification color codes
    'SUCCESS': 'good',
    'FAILURE': 'danger'
]
pipeline {
    agent any
    environment {
        awsEcrCreds = 'ecr:us-east-2:AWSCreds' //Use your own zone and credentials ID
        awsEcrRegistry =  "052101902987.dkr.ecr.us-east-2.amazonaws.com/jenkins-docker-image" //Enter your registry URI
        devopsacadEcrImgReg = "https://052101902987.dkr.ecr.us-east-2.amazonaws.com" //Endpoint of the URI
        awsRegion = "us-east-2"
    }
    tools {
        maven "MAVEN3"
        jdk "OracleJDK8"
    }
    stages {
        stage ('GetCodeFromGit') {
            steps {
                git branch: 'main', url: 'https://github.com/uedwinc/Jenkins-Project.git'
            }
        }
        stage ('MavenBuild') {
            steps {
                sh 'mvn install -DskipTests' //This tells maven to skip test goal
            }
            post {
                success {
                    echo 'Archiving artifacts' //This will provide a download button for the artifact on the pipeline.
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }
        stage ('UnitTest') {
            steps {
                sh 'mvn test'
            }
        }
        stage ('Checkstyle Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle' //This calls up the maven checkstyle plugin for code style reporting
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }
        stage ('SonarQube Code Analysis') {
            environment {
                ScannerHome = tool 'sonar4.7' //As defined in the "SonarQube Scanner" section under "Manage Jenkins" > "Tools"
            }
            steps {
                withSonarQubeEnv('sonarqube') { //'sonarqube' as defined in the "SonarQube servers" section under "Manage Jenkins" > "System"
                    sh ''' ${ScannerHome}/bin/sonar-scanner -Dsonar.projectKey=MavenBuildScan \
                            -Dsonar.projectName=MavenBuildScan \
                            -Dsonar.projectVersion=1.1 \
                            -Dsonar.sources=src/ \
                            -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                            -Dsonar.junit.reportsPath=target/surefire-reports/ \
                            -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                            -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                } //All of these output are defined in the developer pom.xml
            }
        }
        stage ('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    //Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    //true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage ('Upload Artifact to Nexus') {
            steps {
                nexusArtifactUploader ( //Calling up the plugin installed
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '172.31.44.174:8081',
                    groupId: 'com.devopsacad', //Copy from pom.xml
                    version: "${env.BUILD_ID}-${BUILD_TIMESTAMP}", //Make sure you configure this in global settings
                    repository: 'Maven-From-Jenkins', //Create a maven-hosted repository on nexus. Create view* role and attach it to a user.
                    credentialsId: 'JenkinsMaven', //Created in manage jenkins > credentials with username and password from nexus
                    artifacts: [
                        [artifactId: 'devopsacad', //Found in pom.xml
                            classifier: '',
                            file: 'target/devopsacad-v2.war', //Can just use 'target/devopsacad-v2.war'
                            type: 'war'
                        ]
                    ]
                )
            }
        }
        stage ('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build(awsEcrRegistry + ":$BUILD_NUMBER", "./Docker-files/app/multistage/")
                }
            }
        }
        stage ('Push Image to ECR') {
            steps {
                script {
                    docker.withRegistry (devopsacadEcrImgReg, awsEcrCreds) {
                        dockerImage.push ("$BUILD_NUMBER")
                        dockerImage.push ('latest')
                    }
                }
            }
        }
    }
    post {
        always {
            echo 'Slack Notifications' //https://www.jenkins.io/doc/pipeline/steps/slack/
            slackSend channel: 'jenkins-slack-ci',
                color:  COLOR_MAP [currentBuild.currentResult],
                message: "*${currentBuild.currentResult}: * Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}-${env.BUILD_TIMESTAMP} \n More info at ${env.BUILD_URL}"
        }
    }
}