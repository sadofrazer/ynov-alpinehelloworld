pipeline{
    environment{
        IMAGE_NAME = "sadofrazer/alpinehelloworld"
        IMAGE_TAG = "${BUILD_TAG}"
        CONTAINER_NAME = "alpinehelloworld"
        STAGING = "ynov-frazer-staging"
        PRODUCTION = "ynov-frazer-production"
        PRODUCTION_HOST = "3.89.75.76"
    }
    agent none

    stages{

        stage ('Build Stage'){
            agent any
            steps{
                script{
                    sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
                }
            }
        }

        stage ('Run container based on builded image'){
            agent any
            steps{
                script{
                    sh '''
                       docker run -d --name ${CONTAINER_NAME} -e PORT=5000 -p 5000:5000 ${IMAGE_NAME}:${IMAGE_TAG}
                       sleep 5
                    '''
                }
            }
        }

        stage ('Test image'){
            agent any
            steps{
                script{
                    sh '''
                       curl http://localhost:5000 | grep -q "Hello world!"
                    '''
                }
            }
        }

        stage ('Delete Container'){
            agent any
            steps{
                script{
                    sh '''
                       docker stop ${CONTAINER_NAME}
                       docker rm ${CONTAINER_NAME}
                    '''
                }
            }
        }

        stage ('Push image on dockerhub an delete it'){
            agent any
            steps{
                script{
                    sh '''
                       docker login -u sadofrazer -p ${PASSWORD}
                       docker push ${IMAGE_NAME}:${IMAGE_TAG}
                       docker rmi ${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage ('Push image and deploy it in staging env'){
            when{
                expression{ GIT_BRANCH == 'origin/master'}
            }
            agent any
            environment{
                HEROKU_API_KEY = credentials('heroku_api_key')
            }
            steps{
                script{
                    sh'''
                      heroku container:login
                      heroku create ${STAGING} || echo "This env already exist"
                      heroku container:push -a ${STAGING} web
                      heroku container:release -a ${STAGING} web
                    '''
                }
            }
        }

        stage ('Push image and deploy it in production env'){
            when{
                expression{ GIT_BRANCH == 'origin/master'}
            }
            agent any
            environment{
                HEROKU_API_KEY = credentials('heroku_api_key')
            }
            steps{
                script{
                    sh'''
                      heroku container:login
                      heroku create ${PRODUCTION} || echo "This env already exist"
                      heroku container:push -a ${PRODUCTION} web
                      heroku container:release -a ${PRODUCTION} web
                    '''
                }
            }
        }

        stage ('Run container on prod host'){
            agent { label 'prod'}
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                steps{
                    script{
                    sh '''
                       docker rm -f ${CONTAINER_NAME}
                       docker run -d --name ${CONTAINER_NAME} -e PORT=5000 -p 80:5000 ${IMAGE_NAME}:${IMAGE_TAG}
                       sleep 5
                       curl http://localhost:80 | grep -q "Hello world!"
                       curl http://3.89.75.76:80 | grep -q "Hello world!"
                      '''
                    }
                }
            }
        } 

    }
    post {
        success{
            slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
        failure {
            slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
    }
}