pipeline {
environment { // Declaration of environment variables
DOCKER_ID = "kbnhvn" // replace this with your docker-id
DOCKER_IMAGE_CAST = "cast"
DOCKER_IMAGE_MOVIE = "movie"
DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
}
agent any // Jenkins will be able to select all available agents
stages {
    stage(' Docker Build'){ // docker build images stage
        parallel {
            stage(' Build Cast Image') {
                steps {
                    script {
                        sh '''
                        docker rm -f $DOCKER_IMAGE_CAST
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG ./BaseProject/cast-service
                        sleep 6
                        '''
                    }
                }
            }
            stage(' Build Movie Image') {
                steps {
                    script {
                        sh '''
                        docker rm -f $DOCKER_IMAGE_MOVIE
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG ./BaseProject/movie-service
                        sleep 6
                        '''
                    }
                }
            }

        }
    }
    stage(' Test environment deployment'){ // docker build DB images
        steps {
            script {
                sh '''
                docker network create my_network || true
                docker run -d \
                --name cast_db_dev_container --network my_network\
                -v postgres_data_cast:/var/lib/postgresql/data/ \
                -e POSTGRES_USER=cast_db_username \
                -e POSTGRES_PASSWORD=cast_db_password \
                -e POSTGRES_DB=cast_db_dev \
                -p 5433:5432 \
                postgres:12.1-alpine
                docker run -d \
                --name movie_db_dev_container --network my_network\
                -v postgres_data_movie:/var/lib/postgresql/data/ \
                -e POSTGRES_USER=movie_db_username \
                -e POSTGRES_PASSWORD=movie_db_password \
                -e POSTGRES_DB=movie_db_dev \
                -p 5434:5432 \
                postgres:12.1-alpine
                '''
                }
            }
    }
    stage('Docker run'){ // run containers from our builded images
        parallel {
            stage(' Run Cast Container') {
                steps {
                    script {
                        sh '''
                        docker run -d -p 8002:8000 --name $DOCKER_IMAGE_CAST --network my_network\
                        -e DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast_db_dev_container:5433/cast_db_dev \
                        $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG \
                        uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
                        sleep 10
                        '''
                    }
                }
            }
            stage(' Run Movie Container') {
                steps {
                    script {
                        sh '''
                        docker run -d -p 8001:8000 --name $DOCKER_IMAGE_MOVIE --network my_network\
                        -e DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie_db_dev_container:5434/movie_db_dev \
                        -e CAST_SERVICE_HOST_URL=http://cast_service:8000/api/v1/casts/ \
                        $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG \
                        uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
                        sleep 10
                        '''
                    }
                }
            }

        }
    }
    stage('Test Acceptance'){ // we launch the curl commands to validate that the containers responds to the request
        parallel {
            stage(' Test Cast Container') {
                steps {
                    script {
                        sh '''
                        curl localhost:8002
                        '''
                    }
                }
            }
            stage(' Test Movie Container') {
                steps {
                    script {
                        sh '''
                        curl localhost:8001
                        '''
                    }
                }
            }

        }

    }
    stage('Docker Push'){ //we pass the built images to our docker hub account
        environment
        {
            DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
        }
        parallel {
            stage(' Push Cast Image') {
                steps {
                    script {
                        sh '''
                        docker login -u $DOCKER_ID -p $DOCKER_PASS
                        docker push $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
                        '''
                    }
                }
            }
            stage(' Push Movie Image') {
                steps {
                    script {
                        sh '''
                        docker login -u $DOCKER_ID -p $DOCKER_PASS
                        docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
                        '''
                    }
                }
            }

        }

    }

stage('Deploiement en dev'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        NAMESPACE = "dev"
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+namespace.*+namespace: ${NAMESPACE}+g" values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app fastapi --values=values.yml --namespace dev
                '''
                }
            }

        }
stage('Deploiement en QA'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        NAMESPACE = "qa"
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+namespace.*+namespace: ${NAMESPACE}+g" values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app fastapi --values=values.yml --namespace qa
                '''
                }
            }

        }
stage('Deploiement en staging'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        NAMESPACE = "staging"
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+namespace.*+namespace: ${NAMESPACE}+g" values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app fastapi --values=values.yml --namespace staging
                '''
                }
            }

        }
  stage('Deploiement en prod'){
        when {
            branch 'master' // Cette condition s'assure que le stage ne s'exécute que sur la branche master
        }
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        NAMESPACE = "prod"
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
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+namespace.*+namespace: ${NAMESPACE}+g" values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app fastapi --values=values.yml --namespace prod
                '''
                }
            }

        }

}
}
