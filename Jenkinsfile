pipeline {
    agent any 
    
    stages {
        
        stage("Initialize"){
            steps {
                script {
                    def dockerHome = tool "docker"
                    env.PATH = "${dockerHome}/bin:${env.PATH}"
                }
            }
        }

       
        stage("Git"){
            steps{
               script {
                    git branch: 'main', credentialsId: 'github', url: 'https://github.com/afkademy/wordsmith-web.git'
                }
            }
        }

        stage ("Sonar Scan") {
            tools {
                maven "maven-3.9"
            }
            steps {
                withSonarQubeEnv("sonar"){
                    sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.7.0.1746:sonar -Dsonar.projectKey=worthsmith-web'
                }
            }
        }

        stage("Quantity Gates") {
            steps {
                timeout(time: 4, unit: "MINUTES"){
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage ("Build Docker Image") {
            steps {
                script {
                    def tag = getDockerTag()
                    sh "docker build -t 345331916214.dkr.ecr.us-east-2.amazonaws.com/worthsmith-web:${tag} ."
                }
            }
        }

        stage("Push to ECR"){
            steps {
                script{
                    def tag = getDockerTag()
                    withAWS([credentials: 'aws-creds', region: 'us-east-2']) {
                        sh "aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 345331916214.dkr.ecr.us-east-2.amazonaws.com"
                        sh "docker push 345331916214.dkr.ecr.us-east-2.amazonaws.com/worthsmith-web:${tag}"
                    }
                }
            }
        }
    }

    post{
        always {
            script {
                def msg = "See ${env.BUILD_URL}console"
                def subject = "Jenkins: ${env.JOB_NAME}: Build status is ${currentBuild.currentResult}"
                withAWS([credentials: 'aws-creds', region: 'us-east-2']){
                    sh "aws sns publish --topic-arn arn:aws:sns:us-east-2:345331916214:jenkins-notification --message '${msg}' --subject '${subject}'"
                }
            }
        }
    }
}


def getDockerTag() {
    // develop=> 1.1.0.230-rc    | main => 1.1.0.200 | feature => 1.1.0.240-feature-something
    def version = "1.1.0"
    def branch = "${env.BRANCH_NAME}"
    def build_number = "${env.BUILD_NUMBER}"

    def tag = "" 

    if (branch == 'main') {
        tag = "${version}.${build_number}"
    } else if(branch == "develop") {
        tag = "${version}.${build_number}-rc"
    } else {
        branch = branch.replace("/", "-").replace("\\", "-")
        tag = "${version}.${build_number}-${branch}"
    }

    return tag 
}
