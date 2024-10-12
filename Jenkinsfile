pipeline {
    environment {
        // Déclaration des variables d'environnement
        DOCKER_ID = "mira797"
        DOCKER_IMAGE_CAST = "cast-service"
        DOCKER_IMAGE_MOVIE = "movie-service"
        // DOCKER_IMAGE_NGINX = "nginx"
        DOCKER_TAG_CAST = "cast-v.${BUILD_ID}.0" // Tag spécifique pour cast-service
        DOCKER_TAG_MOVIE = "movie-v.${BUILD_ID}.0" // Tag spécifique pour movie-service
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
                    # Exécuter cast-service sur le port 8090
                    docker run -d -p 8090:8000 --name cast-service $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG_CAST

                    # Exécuter movie-service sur le port 8001 (pour éviter le conflit de ports)
                    docker run -d -p 8001:8000 --name movie-service $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG_MOVIE
                    sleep 10
                    '''
                }
            }
        }

        stage('Docker Push') { // We pass the built image to our Docker Hub account
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // We retrieve the Docker password from Jenkins credentials
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

        stage('Configure Kubernetes Access') {
            environment {
                KUBECONFIG = credentials('config') // Retrieve the kubeconfig file from Jenkins credentials
            }
            steps {
                script {
                    sh '''
                    # Supprimer le répertoire .kube s'il existe
                    rm -Rf ~/.kube

                    # Créer le répertoire .kube et y ajouter le fichier kubeconfig
                    mkdir -p ~/.kube
                    echo "$KUBECONFIG" > ~/.kube/config

                    # Assurez-vous que les permissions sont correctes
                    chmod 600 ~/.kube/config
                    '''
                }
            }
        }

        stage('Prepare Kubernetes Secrets') {
            steps {
                // Supprimer les secrets dans plusieurs namespaces
                sh '''
                kubectl delete secret postgres-secret --namespace dev || echo "Secret not found in dev, skipping deletion"
                kubectl delete secret postgres-secret --namespace staging || echo "Secret not found in staging, skipping deletion"
                kubectl delete secret postgres-secret --namespace prod || echo "Secret not found in prod, skipping deletion"
                '''
            }
        }

        stage('Deployment in dev') {
            environment {
                KUBECONFIG = credentials('config') // Retrieve the kubeconfig file from Jenkins credentials
            }

            steps {
                script {
                    sh '''
                    # Supprimer le répertoire .kube s'il existe
                    rm -Rf ~/.kube

                    # Créer le répertoire .kube et y ajouter le fichier kubeconfig
                    mkdir -p ~/.kube
                    echo "$KUBECONFIG" > ~/.kube/config

                    # Assurez-vous que les permissions sont correctes
                    chmod 600 ~/.kube/config

                    # Mise à jour des fichiers YAML avec les tags Docker
                    sed -i "s+tag.*+tag: ${DOCKER_TAG_CAST}+g" ./cast_service/values.yaml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG_MOVIE}+g" ./movie_service/values.yaml

                    # Déploiement Helm dans dev
                    helm upgrade --install app-cast ./cast_service --values=./cast_service/values.yaml --namespace dev
                    helm upgrade --install app-movie ./movie_service --values=./movie_service/values.yaml --namespace dev
                    helm upgrade --install app-nginx ./nginx --values=./nginx/values.yaml --namespace dev
                    '''
                }
            }
        }

        stage('Deployment in staging') {
            environment {
                KUBECONFIG = credentials('config') // Retrieve the kubeconfig file from Jenkins credentials
            }

            steps {
                script {
                    sh '''
                    # Supprimer le répertoire .kube s'il existe
                    rm -Rf ~/.kube
                    mkdir -p ~/.kube
                    echo "$KUBECONFIG" > ~/.kube/config
                    chmod 600 ~/.kube/config

                    # Mise à jour des namespaces dans les fichiers de valeurs
                    sed -i '/namespace:/s/dev/staging/' ./cast_service/values.yaml
                    sed -i '/namespace:/s/dev/staging/' ./movie_service/values.yaml
                    cat ./cast_service/values.yaml

                    # Annotation des PV pour le namespace staging
                    #kubectl annotate pv movie-db-st meta.helm.sh/release-namespace=staging --overwrite
                    #kubectl annotate svc nginx -n dev "meta.helm.sh/release-namespace=staging" --overwrite
                    #kubectl annotate deployment nginx -n dev "meta.helm.sh/release-name-" --overwrite
                    #kubectl annotate deployment nginx -n dev "meta.helm.sh/release-namespace-" --overwrite

                    # Déploiement via Helm dans staging
                    helm upgrade --install app-cast_staging ./cast_service --values=./cast_service/values.yaml --namespace staging
                    helm upgrade --install app-movie_staging ./movie_service --values=./movie_service/values.yaml --namespace staging
                    helm upgrade --install app-nginx_staging ./nginx --values=./nginx/values.yaml --namespace staging
                    '''
                }
            }
        }

        stage('Deployment in prod') {
            environment {
                KUBECONFIG = credentials('config') // Retrieve the kubeconfig file from Jenkins credentials
            }

            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production?', ok: 'Yes'
                }

                script {
                    sh '''
                    # Supprimer le répertoire .kube s'il existe
                    rm -Rf ~/.kube
                    mkdir -p ~/.kube
                    echo "$KUBECONFIG" > ~/.kube/config
                    chmod 600 ~/.kube/config

                    # Mise à jour des namespaces dans les fichiers de valeurs
                    sed -i '/namespace:/s/staging/prod/' ./cast_service/values.yaml
                    sed -i '/namespace:/s/staging/prod/' ./movie_service/values.yaml
                    cat ./cast_service/values.yaml

                    # Déploiement via Helm dans prod
                    helm upgrade --install app-cast_prod ./cast_service --values=./cast_service/values.yaml --namespace prod
                    helm upgrade --install app-movie_prod ./movie_service --values=./movie_service/values.yaml --namespace prod
                    helm upgrade --install app-nginx_prod ./nginx --values=./nginx/values.yaml --namespace prod
                    '''
                }
            }
        }
    }
}
