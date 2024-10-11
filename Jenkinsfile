pipeline {
    environment {
        // Déclaration des variables d'environnement
        DOCKER_ID = "mira797"
        DOCKER_IMAGE_CAST = "cast-service"
        DOCKER_IMAGE_MOVIE = "movie-service"
        // DOCKER_IMAGE_NGINX = "nginx"
        DOCKER_TAG_CAST = "cast-v.${BUILD_ID}.0" // Tag spécifique pour cast-service
        DOCKER_TAG_MOVIE = "movie-v.${BUILD_ID}.0" // Tag spécifique pour movie-service
        // DOCKER_TAG_NGINX = "nginx-v.${BUILD_ID}.0"  // Tag pour nginx
    }

    agent any

    stages {
        stage('Docker Build of all images') { // docker build image stage
            steps {
                script {
                    sh '''
                    # Supprimer tous les conteneurs s'ils existent
                    if [ "$(docker ps -aq)" ]; then
                        docker rm -f $(docker ps -aq)
                    fi

                    # Build de l'image cast-service
                    docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG_CAST ./cast-service || exit 1

                    # Build de l'image movie-service
                    docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG_MOVIE ./movie-service || exit 1

                    # Pause de 6 secondes (si nécessaire)
                    sleep 6
                    '''
                }
            }
        }

        stage('Docker run') { // run container from our built image
            steps {
                script {
                    sh '''
                    # Exécuter cast-service sur le port 8081
                    docker run -d -p 8081:8080 --name cast-service $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG_CAST

                    # Exécuter movie-service sur le port 8082 (pas 8081 pour éviter le conflit)
                    docker run -d -p 8082:8080 --name movie-service $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG_MOVIE

                    # docker run -d -p 8083:8080 --name nginx $DOCKER_ID/$DOCKER_IMAGE_NGINX:$DOCKER_TAG_NGINX
                    sleep 10
                    '''
                }
            }
        }

        stage('Test Acceptance') { // We launch the curl command to validate that the container responds to the request
            steps {
                script {
                    sh '''
                    sleep 10

                    # Tester si cast-service répond sur le port 8081
                    curl localhost:8081 || exit 1

                    # Tester si movie-service répond sur le port 8082
                    curl localhost:8082 || exit 1
                    '''
                }
            }
        }

        stage('Docker Push') { // We pass the built image to our docker hub account
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // We retrieve docker password from secret text saved on Jenkins
            }

            steps {
                script {
                    sh '''
                    docker login -u $DOCKER_ID -p $DOCKER_PASS
                    docker push $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG_CAST
                    docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG_MOVIE
                    '''
                }
            }
        }

        stage('Deployment in dev') {
            environment {
                KUBECONFIG = credentials("config") // We retrieve kubeconfig from secret file saved on Jenkins
            }

            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config
                    sed -i "s+tag.*+tag: ${DOCKER_TAG_CAST}+g" ./cast_service/values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG_MOVIE}+g" ./movie_service/values.yml
                    helm upgrade --install app-cast ./cast_service --values=./cast_service/values.yml --namespace dev
                    helm upgrade --install app-movie ./movie_service --values=./movie_service/values.yml --namespace dev
                    helm upgrade --install app-nginx ./nginx --values=./nginx/values.yml --namespace dev
                    '''
                }
            }
        }

        stage('Deployment in staging') {
            environment {
                KUBECONFIG = credentials("config") // We retrieve kubeconfig from secret file saved on Jenkins
            }

            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config
                    sed -i "s+tag.*+tag: ${DOCKER_TAG_CAST}+g" ./cast_service/values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG_MOVIE}+g" ./movie_service/values.yml
                    helm upgrade --install app-cast ./cast_service --values=./cast_service/values.yml --namespace staging
                    helm upgrade --install app-movie ./movie_service --values=./movie_service/values.yml --namespace staging
                    helm upgrade --install app-nginx ./nginx --values=./nginx/values.yml --namespace staging
                    '''
                }
            }
        }

        stage('Deployment in prod') {
            environment {
                KUBECONFIG = credentials("config") // We retrieve kubeconfig from secret file saved on Jenkins
            }

            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production ?', ok: 'Yes'
                }

                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config
                    sed -i "s+tag.*+tag: ${DOCKER_TAG_CAST}+g" ./cast_service/values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG_MOVIE}+g" ./movie_service/values.yml
                    helm upgrade --install app-cast ./cast_service --values=./cast_service/values.yml --namespace prod
                    helm upgrade --install app-movie ./movie_service --values=./movie_service/values.yml --namespace prod
                    helm upgrade --install app-nginx ./nginx --values=./nginx/values.yml --namespace prod
                    '''
                }
            }
        }
    }
}
