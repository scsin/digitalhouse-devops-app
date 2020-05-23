pipeline {

    agent none

    environment {

        NODE_ENV="dev"
        
        APP_PREFIX = "docker-images-zumims20"
        APP_IMAGE = "${APP_PREFIX}:${BUILD_NUMBER}"
        APP_CONTAINER = "${APP_PREFIX}-${BUILD_NUMBER}"
        PORT_IMAGE='3000'
        PORT_CONTAINER="8030"
        REGISTRY_ADDRESS = "086217385171.dkr.ecr.us-east-1.amazonaws.com"

        CREDENTIALID="aws_credential"
        CREDENTIAL_ECR="ecr:us-east-1:${CREDENTIALID}"
        BUCK_NAME="zumims20-dev"
        CREDENTIALID_S3="s3_dev"
        
        REGION="us-east-1" 
        IS_BUILD_VERSION="YES"
        IS_NEW_VERSION="NO"
    }


    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }
    triggers {
        cron('@daily')
    }

    stages{

        
        stage("Build, Test and Push Docker Image") {
             
            agent {
                label 'master'
            }
            when {
                environment name: "IS_BUILD_VERSION", value: "YES"
            }
            
            stages {

                stage('Clone repository') {
                    steps {
                        checkout scm
                        
                        script {
                            SHORTCOMMIT = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
                            APP_IMAGE = "${APP_PREFIX}:${SHORTCOMMIT}"
                        }
                        sh('printenv | sort')
                    }
                }
                stage('Build image'){       
                    steps {
                        sh('printenv | sort')
                        script {
                            print "Environment will be : ${NODE_ENV}/${BUILD_ID}"
                            docker.build("${APP_IMAGE}") 
                        }
                    }
                }

                stage('Test image') {
                    steps {
                        script {
                            withCredentials([[$class:'AmazonWebServicesCredentialsBinding' 
                                , credentialsId: "${CREDENTIALID_S3}"]]) {

                                docker.image("${APP_IMAGE}").withRun("-p ${PORT_CONTAINER}:${PORT_IMAGE} -e NODE_ENV=${NODE_ENV} -e AWS_ACCESS_KEY=${env.AWS_ACCESS_KEY_ID} -e AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY} -e BUCKET_NAME=${BUCK_NAME}") { c ->
                                    sh 'docker ps'
                                    sh 'sleep 10'
                                    sh "curl http://127.0.0.1:${PORT_CONTAINER}/api/v1/healthcheck"
                                    
                                }
                            }
                        }
                    }
                }

                stage('Docker push') {
                    steps {
                        echo 'Push latest para AWS ECR'
                        script {
                            docker.withRegistry("https://${REGISTRY_ADDRESS}", "${CREDENTIAL_ECR}") {
                                docker.image("${APP_IMAGE}").push("${SHORTCOMMIT}")
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            agent {  
                label 'dev'
            }
            when {
                environment name: "IS_BUILD_VERSION", value: "YES"
            }
            steps { 
                script {
                    echo 'Deploy to Develop'

                    docker.withRegistry("https://${REGISTRY_ADDRESS}", "${CREDENTIAL_ECR}") {
                        docker.image("${APP_IMAGE}").pull()
                    }

                    withCredentials([[$class:'AmazonWebServicesCredentialsBinding' 
                        , credentialsId: "${CREDENTIALID_S3}"]]) {
                        try {
                            sh "docker run -d --rm --name ${APP_PREFIX} -p ${PORT_CONTAINER}:${PORT_IMAGE} -e NODE_ENV=${NODE_ENV} -e AWS_ACCESS_KEY=${env.AWS_ACCESS_KEY_ID} -e AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY} -e BUCKET_NAME=${BUCK_NAME} ${REGISTRY_ADDRESS}/${APP_IMAGE}"
                        } 
                        catch (Exception err) {
                            sh "docker stop ${APP_PREFIX}"
                            sh 'sleep 10'
                            sh "docker run -d --rm --name ${APP_PREFIX} -p ${PORT_CONTAINER}:${PORT_IMAGE} -e NODE_ENV=${NODE_ENV} -e AWS_ACCESS_KEY=${env.AWS_ACCESS_KEY_ID} -e AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY} -e BUCKET_NAME=${BUCK_NAME} ${REGISTRY_ADDRESS}/${APP_IMAGE}"
                        }

                    }
                    
                    sh "docker ps"
                    sh 'sleep 10'
                    sh "curl http://127.0.0.1:${PORT_CONTAINER}/api/v1/healthcheck"
                }
            }
            post{
                failure {
                    sh "docker stop ${APP_PREFIX}"
                }
            }
        }
        stage('Docker push latest') {
            agent {
                label 'master'
            }
            when {
                environment name: "IS_BUILD_VERSION", value: "YES"
            }
            steps {
                echo 'Push latest para AWS ECR'
                script {

                    sh "docker tag ${APP_IMAGE} ${APP_PREFIX}:latest"

                    docker.withRegistry("https://${REGISTRY_ADDRESS}", "${CREDENTIAL_ECR}") {
                        docker.image("${APP_PREFIX}").push("latest")
                    }
                }
            }
        }
    }
}