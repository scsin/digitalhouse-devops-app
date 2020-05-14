pipeline {

    agent none

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }
    triggers {
        cron('@daily')
    }

    stages{
        stage("Build, Test and Push Docker Image") {
            agent {  
                node {
                    label 'master'
                }
            }
            stages {

                stage('Clone repository') {
                    steps {
                        sh 'hostname'
                        script {
                            if(env.GIT_BRANCH=='origin/dev'){
                                checkout scm
                            }
                            sh('printenv | sort')
                            echo "My branch is: ${env.GIT_BRANCH}"
                        }
                    }
                }
                stage('Build image'){       
                    steps {
                        script {
                            print "Environment will be : ${env.NODE_ENV}"
                            docker.build("digitalhouse-devops:latest")
                        }
                    }
                }

                stage('Test image') {
                    steps {
                        script {

                            docker.image("digitalhouse-devops:latest").withRun('-p 8030:3000') { c ->
                                sh 'docker ps'
                                sh 'sleep 10'
                                sh 'curl http://127.0.0.1:8030/api/v1/healthcheck'
                                
                            }
                    
                        }
                    }
                }

                stage('Docker push') {
                    steps {
                        echo 'Push latest para AWS ECR'
                        script {
                            docker.withRegistry('https://086217385171.dkr.ecr.us-east-1.amazonaws.com', 'ecr:us-east-1:docker-images-pi') {
                                docker.image('docker-images-pi').push()
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to Homolog') {
            agent {  
                node {
                    label 'Homolog'
                }
            }

            steps { 
                script {
                    if(env.GIT_BRANCH=='origin/dev'){
 
                        docker.withRegistry('https://086217385171.dkr.ecr.us-east-1.amazonaws.com', 'ecr:us-east-1:docker-images-pi') {
                            docker.image('docker-images-pi').pull()
                        }

                        echo 'Deploy para Desenvolvimento'
                        sh "hostname"
                        sh "docker stop app1"
                        sh "docker rm app1"
                        //sh "docker run -d --name app1 -p 8030:3000 086217385171.dkr.ecr.us-east-1.amazonaws.com/docker-images-pi:latest"
                        withCredentials([[$class:'AmazonWebServicesCredentialsBinding' 
                            , credentialsId: 'homologs3']]) {
                        sh "docker run -d --name app1 -p 8030:3000 -e NODE_ENV=homolog -e AWS_ACCESS_KEY=${env.AWS_ACCESS_KEY_ID} -e AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY} -e BUCKET_NAME=${env.BUCKET_NAME} 086217385171.dkr.ecr.us-east-1.amazonaws.com/docker-images-pi:latest"
                        }
                        
                        sh "docker ps"
                        sh 'sleep 10'
                        sh 'curl http://127.0.0.1:8030/api/v1/healthcheck'

                    }
                }
            }

        }

        stage('Deploy to Producao') {
            agent {  
                node {
                    label 'Prod'
                }
            }

            steps { 
                sh 'hostname'
                script {
                    if(env.GIT_BRANCH=='origin/prod'){
 
                        environment {

                            NODE_ENV="production"
                            AWS_ACCESS_KEY=credentials('aws_access_key')
                            AWS_SECRET_ACCESS_KEY=credentials('aws_secret_access_key')
                            AWS_SDK_LOAD_CONFIG="0"
                            BUCKET_NAME="digitalhouse-gitgirls-prod-sara"
                            REGION="us-east-1" 
                            PERMISSION=""
                            ACCEPTED_FILE_FORMATS_ARRAY=""
                        }


                        docker.withRegistry('https://086217385171.dkr.ecr.us-east-1.amazonaws.com', 'ecr:us-east-1:docker-images-pi') {
                            docker.image('docker-images-pi').pull()
                        }

                        echo 'Deploy para Producao'
                        sh "hostname"
                        sh "docker stop app1"
                        sh "docker rm app1"
                        //sh "docker run -d --name app1 -p 8030:3000 086217385171.dkr.ecr.us-east-1.amazonaws.com/docker-images-pi:latest"
                        withCredentials([[$class:'AmazonWebServicesCredentialsBinding' 
                            , credentialsId: 'producaos3']]) {
                          sh "docker run -d --name app1 -p 8030:3000 -e NODE_ENV=producao -e AWS_ACCESS_KEY=$AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY -e BUCKET_NAME=${env.BUCKET_NAME} 086217385171.dkr.ecr.us-east-1.amazonaws.com/docker-images-pi:latest"
                        }
                        sh "docker ps"
                        sh 'sleep 10'
                        sh 'curl http://127.0.0.1:8030/api/v1/healthcheck'

                    }
                }
            }

        }

    }
}
