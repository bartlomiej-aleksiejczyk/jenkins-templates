pipeline {
    agent any

    environment {
        // Define your image name here
        IMAGE_NAME = "${JOB_NAME}".toLowerCase().replaceAll(/[^a-z0-9._-]/, '-')
        // Define a tag for your image
        IMAGE_TAG = 'latest'
        SPRING_DB_PROD_URL = "${env.SPRING_DB_PROD_URL}"
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }
    

stage('Build Docker Image') {
    steps {
        script {
            echo SPRING_DB_PROD_URL
            sh "docker stop ${env.IMAGE_NAME} || true"
            sh "docker rm ${env.IMAGE_NAME} || true"
            // Use the 'withCredentials' block to obtain database credentials
            withCredentials([usernamePassword(credentialsId: 'database-config', passwordVariable: 'DB_PASSWORD', usernameVariable: 'DB_USERNAME')]) {
            // Ensure the build arguments are correctly quoted
                sh '''
                # Use environment variables directly without Groovy interpolation
                docker build -t $IMAGE_NAME:$IMAGE_TAG --build-arg DB_USERNAME=$DB_USERNAME --build-arg DB_PASSWORD=$DB_PASSWORD --build-arg SPRING_DB_PROD_URL=$SPRING_DB_PROD_URL .
                '''            
            }
        }
    }
}


        stage('Get Host IP') {
            steps {
                script {
                    // Using a shell command to get the host IP address. Adjust the command according to your OS and network configuration.
                    // Note the use of double dollar signs ($$) to escape the dollar sign in the Groovy string.
                    env.HOST_IP = sh(script: "hostname -I | awk '{print \$1}'", returnStdout: true).trim()
                }
            }
        }

        stage('Ensure Traefik is Running') {
            steps {
                script {
                    // Execute a shell command to check if the Traefik container is running and start it if it isn't
                    sh '''
                    RUNNING=$(docker ps --filter "name=^/traefik$" --format "{{.Names}}")
                    if [ "$RUNNING" != "traefik" ]; then
                    echo "Starting Traefik container..."
                    docker rm traefik || true
                    docker run -d --name traefik \
                        --restart=unless-stopped \
                        -p 80:80 \
                        -p 8085:8080 \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        traefik:v2.5 \
                        --api.insecure=true \
                        --providers.docker \
                        --entrypoints.web.address=:80
                    else
                    echo "Traefik container is already running."
                    fi
                    '''
                }
            }
        }


        stage('Deploy') {
            steps {
                script {
                    // Inject database username and password as environment variables
                    withCredentials([
                        usernamePassword(credentialsId: 'database-config', passwordVariable: 'DB_PASSWORD', usernameVariable: 'DB_USERNAME')
                    ]) {
                        // Using single quotes in sh step. Note: Variables inside single quotes won't be interpolated by Groovy.
                        // Environment variables passed to Docker need to be referenced directly as $VARIABLE in the shell,
                        // assuming Jenkins automatically exports them to the shell environment
                        sh '''
                        docker run -d --restart=unless-stopped --name $IMAGE_NAME \
                        -e DB_USERNAME="$DB_USERNAME" -e DB_PASSWORD="$DB_PASSWORD" -e SPRING_DB_PROD_URL="$SPRING_DB_PROD_URL" -e IMAGE_NAME="$IMAGE_NAME"\
                        -l traefik.enable=true \
                        -l "traefik.http.routers.$IMAGE_NAME.rule=Host(\\`$HOST_IP\\`) && PathPrefix(\\`/$IMAGE_NAME\\`)" \
                        -l traefik.http.services.$IMAGE_NAME.loadbalancer.server.port=8080 \
                        $IMAGE_NAME:$IMAGE_TAG
                        '''
                    }
                }
            }
        }
    }

    
}
