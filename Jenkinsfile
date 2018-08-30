pipeline {
    agent {
        label 'jenkins-docker-slave'
    }
    tools {
        nodejs "node-6.3.1"
    }
    environment{
        AWS_ID = credentials("servana-aws-creds")
        AWS_ACCESS_KEY_ID = "${env.AWS_ID_USR}"
        AWS_SECRET_ACCESS_KEY = "${env.AWS_ID_PSW}"
        AWS_DEFAULT_REGION = 'eu-west-1'
        AWS_DEFAULT_OUTPUT = 'json'
        DEVELOPERS_EMAIL="dhruva@servana.net"
        ENVIRONMENT='dev'
        DOCKER_REPOSITORY='482016542819.dkr.ecr.eu-west-1.amazonaws.com'
        APP_NAME='nodejs-app'
    }
    options{
        timeout(time:48,unit:'HOURS')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timestamps()
    }
    stages {
        stage('Checkout') {
            steps {
            script {
            env.SOURCE_HASH = sh ( script: 'git rev-parse HEAD | cut -c1-6',returnStdout: true).trim()
            }   
            }
        }
        stage('QA/QC'){
        parallel {
            stage("CodeQualityCheck"){
                steps {
                    script{
                    codeQualityCheck()
                    }
                    }
            }
            stage("ImageBuild") {
                steps {
                    imageBuild()
                      }
            }
            }
        }
        stage('ImageScan') {
                steps {
                    imageScan()
                }
        }
        stage('UnitTest'){
            steps{
                runTest()
            }
        }
        stage("ImagePush"){
            steps{
                uploadToRepository();
            }
        }
        stage("Approve"){
            agent none
            steps{
                script{
                    env.DEPLOY_TO= input message: 'Approval is required',
                            parameters: [
                                choice(name: "Do you want to deploy to ${ENVIRONMENT}?", choices: 'no\nyes', 
                                description: 'Choose "yes" if you want to deploy the DEV server')
                            ]
                }
            }
        }
        stage('Deploy'){
            when{
                environment name:'DEPLOY_TO', value:'yes'
                expression { env.Environment == 'dev' }
            }
            steps{
                deployToECS("DEV")
            }
        }
        stage('Archive'){
            steps{
                archiveTheBuild()
            }
        }
    }
    post {
        always {
            finalizeWorkflow()
        }
        success {
            echo "Sucess"
            notifyBuild("SUCCESS")
        }
        failure {
            echo "Success"
            notifyBuild("FAILED")
        }
        unstable {
            echo "Success"
            notifyBuild("UNSTABLE")
        }
        aborted {
            echo "Success"
            notifyBuild("ABORTED")
        }
    }
}

//define groovy functions here
def codeQualityCheck(){
    echo 'Executing Code Quality Check....'
    def scannerHome = tool 'sonar-scanner';
    withSonarQubeEnv('sonarqube') {
        sh "${scannerHome}/bin/sonar-scanner"
    }
}

def imageBuild(){
    echo 'Build docker image'
    sh "docker build . -t ${DOCKER_REPOSITORY}/${APP_NAME}:${SOURCE_HASH}"
}

def imageScan(){
    echo 'Scan docker image'
    aquaMicroscanner imageName: "${DOCKER_REPOSITORY}/${APP_NAME}:${SOURCE_HASH}", notCompliesCmd: '', onDisallowed: 'ignore', outputFormat: 'html'
}

def runTest(){
    echo "Run npm unit tests here"
}
def uploadToRepository(){
    echo "We are about to publish artifacts to remote repository"
    sh """
    `aws ecr get-login --no-include-email --region eu-west-1`
    docker push $DOCKER_REPOSITORY/$APP_NAME:$SOURCE_HASH
    """
}

def deploytoECS(){
    echo 'Deploy to ECS'
}


def archiveTheBuild(){
    echo "We are archiveing the files to S3"
    //archive '*/target/**/*'
    //junit '*/target/surefire-reports/*.xml'
    //aws s3 sync...
}

def deployToECS(deployTo){
    echo "We are deploying to : ${deployTo}"
    //Implement ECS Deploy leveraging https://github.com/silinternational/ecs-deploy
}

def finalizeWorkflow(){
    echo "Cleaning up the workspace"
}

def notifyBuild(bStatus) {
    def subject = "${bStatus}: Job ${env.JOB_NAME} [${env.BUILD_NUMBER}]"
    def details = """Check console output at: ${env.BUILD_URL}"""
    switch (bStatus) {
    case "SUCCESS":
      colorhash = "good"
      break
    case "FAILED":
      colorhash = "danger"
      break
    case "UNSTABLE":
      colorhash = "warning"
      break
    case "ABORTED":
      colorhash = "warning"
      break
    }    
    slackSend channel:'#cistack',
    color: "${colorhash}",
    message: "${subject} \n ${details}",
    failOnError: true
    //mail to:"${env.DEVELOPERS_EMAIL}",subject:"${subject}",body:"${details}"
    //build status to slack
}