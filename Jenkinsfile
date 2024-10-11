pipeline {
    environment { 
        // Déclaration des variables d'environnement
        DOCKER_ID = "mira797"
        DOCKER_IMAGE_CAST = "cast-service"
        DOCKER_IMAGE_MOVIE = "movie-service"
        #DOCKER_IMAGE_NGINX = "nginx"
        DOCKER_TAG_CAST = "cast-v.${BUILD_ID}.0" // Tag spécifique pour cast-service
        DOCKER_TAG_MOVIE = "movie-v.${BUILD_ID}.0" // Tag spécifique pour movie-service
        #DOCKER_TAG_NGINX = "nginx-v.${BUILD_ID}.0"  // Tag pour nginx 
    }

agent any // Jenkins will be able to select all available agents
stages {
        stage('Docker Build of all images '){ // docker build image stage
            steps {                script {
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
                    #docker build -t $DOCKER_ID/$DOCKER_IMAGE_NGINX:$DOCKER_TAG_NGINX ./nginx || exit 1
                    sleep 6
                    '''
                }
            }
        }
        stage(' Docker run'){ // run container from our built image
                steps {
                    script {
                    sh '''
                    # Exécuter cast-service sur le port 8080
		    docker run -d -p 8081:8080 --name cast-service $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG_CAST

	            # Exécuter movie-service sur le port 8081
		    docker run -d -p 8082:8080 --name movie-service $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG_MOVIE

                    #docker run -d -p 8083:8080 --name nginx $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG_MOVIE
                    sleep 10
                    '''
                    }
                }
            }

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
	            sleep 10

	            # Tester si cast-service répond sur le port 8081
	            curl localhost:8081 || exit 1

	            # Tester si movie-service répond sur le port 8082
	            curl localhost:8082 || exit 1
	            '''
                    }
            }

        }
        stage('Docker Push'){ //we pass the built image to our docker hub account
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve docker password from secret text called docker_hub_pass saved on jenkins
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

stage('Deployment in dev'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cat ./movie_service/values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG_CAST}+g" ./cast_service/values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG_MOVIE}+g" ./movie_service/values.yml
                helm upgrade --install app-cast ./cast_service --values=./cast_service/values.yml --namespace dev
                helm upgrade --install app-movie ./movie_service --values=./movie_service/values.yml --namespace dev
                helm upgrade --install app-nginx ./nginx --values=./nginx/values.yml --namespace dev

                '''
                }
            }

        }
stage('Deploiement en staging'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cat ./movie_service/values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG_CAST}+g" ./cast_service/values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG_MOVIE}+g" ./movie_service/values.yml
                helm upgrade --install app-cast ./cast_service --values=./cast_service/values.yml --namespace staging
                helm upgrade --install app-movie ./movie_service --values=./movie_service/values.yml --namespace staging
                helm upgrade --install app-nginx ./nginx --values=./nginx/values.yml --namespace staging
 
                '''
                }
            }

        }
  stage('Deploiement en prod'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve kubeconfig from secret file called config saved on jenkins
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
                sh '''
                rm -Rf .kube
                mkdir .kube
                cat ./movie_service/values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG_CAST}+g" ./cast_service/values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG_MOVIE}+g" ./movie_service/values.yml
                helm upgrade --install app-cast ./cast_service --values=./cast_service/values.yml --namespace prod
                helm upgrade --install app-movie ./movie_service --values=./movie_service/values.yml --namespace prod
                helm upgrade --install app-nginx ./nginx --values=./nginx/values.yml --namespace production

                '''
                }
            }

        }

}
}
