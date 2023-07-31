pipeline {

    environment { // Declaration of environment variables
    DOCKER_ID = "ardhmd" // replace this with your docker-id
    DOCKER_IMAGE = "cast-service"
    DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
    }

    agent any // Jenkins will be able to select all available agents

    stages {
        stage(' Docker Build -> Cast Service'){ // docker build image stage
            steps {
                script {
                sh '''
                docker rm -f cast_db
                docker rm -f cast_service
                cd cast-service/
                docker build -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG .
                cd - >/dev/null
                sleep 6
                '''
                }
            }
        }

        stage(' Docker run -> Cast DB'){ // run container from our builded image
            environment
            {
                CAST_DB_PASSWORD = credentials("CAST_DB_PASSWORD") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
                CAST_DB_NAME = credentials("CAST_DB_NAME") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
            }
            steps {
                script {
                sh '''
                docker run -d --name cast_db --network monReseau -e POSTGRES_USER=$CAST_DB_NAME -e POSTGRES_PASSWORD=$CAST_DB_PASSWORD -e POSTGRES_DB=cast_db_dev postgres:12.1-alpine
                sleep 10
                '''
                }
            }
        }

        stage(' Docker run -> Cast Service'){ // run container from our builded image
            environment
            {
                CAST_DB_PASSWORD = credentials("CAST_DB_PASSWORD") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
                CAST_DB_NAME = credentials("CAST_DB_NAME") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
            }
            steps {
                script {
                sh '''
                docker run -d -p 8085:8000 -e DATABASE_URI=postgresql://$CAST_DB_NAME:$CAST_DB_PASSWORD@cast_db/cast_db_dev --name cast_service --network monReseau $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                sleep 10
                '''
                }
            }
        }

        stage('Test Acceptance -> Cast Service'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
                    sh '''
                    curl -I -s localhost:8085/api/v1/casts/docs | grep HTTP/1.1 | awk '{print $2}'
                    '''
                    }
            }

        }

        stage('Docker Push -> Cast Service'){ //we pass the built image to our docker hub account
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
            }

            steps {

                script {
                sh '''
                docker login -u $DOCKER_ID -p $DOCKER_PASS
                docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                '''
                }
            }

        }

        stage('Deploiement en dev -> Cast Service'){
            environment
            {
            KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
            }
                steps {
                    script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    cp cast-chart/values.yaml values.yaml
                    cat values.yaml
                    sed -i '0,/tag.*/s//tag: ${DOCKER_TAG}/' values.yaml
                    helm upgrade --install cast-chart-release cast-chart --values=values.yaml --set service.port=8001 --set db.name=cast_db_dev --namespace dev
                    '''
                    }
                }

        }

        stage('Deploiement en staging -> Cast Service'){
            environment
            {
            KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
            }
                steps {
                    script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    cp cast-chart/values.yaml values.yaml
                    cat values.yaml
                    sed -i '0,/tag.*/s//tag: ${DOCKER_TAG}/' values.yaml
                    helm upgrade --install cast-chart-release cast-chart --values=values.yaml --set service.port=8003 --set db.name=cast_db_staging --namespace staging
                    '''
                    }
                }

        }

        stage('Deploiement en prod -> Cast Service'){
            environment
            {
            KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
            }
                steps {
                // Create an Approval Button with a timeout of 15minutes.
                // this require a manuel validation in order to deploy on production environment
                        timeout(time: 15, unit: "MINUTES") {
                            input message: 'Do you want to deploy in production ?', ok: 'Yes'
                        }

                    script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    cp cast-chart/values.yaml values.yaml
                    cat values.yaml
                    sed -i '0,/tag.*/s//tag: ${DOCKER_TAG}/' values.yaml
                    helm upgrade --install cast-chart-release cast-chart --values=values.yaml --set service.port=8005 --set db.name=cast_db_prod --namespace prod
                    '''
                    }
                }
        }
    }
}
