pipeline {

    agent {
        docker {
            image 'devops-agent:latest'
        }
    }

    environment {
        APELLIDO = "vallejo"
        ACR_NAME = "acrglobalcicd"
        ACR_LOGIN_SERVER = "${ACR_NAME}.azurecr.io"
        IMAGE_NAME = "my-nodejs-app-${APELLIDO}"
        RESOURCE_GROUP = "rg-cicd-terraform-app-baraujox"
        AKS_NAME = "aks-dev-eastus"
        ENV = "dev"
        API_PROVIDER_URL = "https://dev.api.com"
    }

    stages {

        stage('[CI] install dependencies') {
            steps {
                sh '''
                echo ">>> Instalando dependencias..."
                npm install
                '''
            }
        }

        stage('[CI] Unit tests') {
            steps {
                sh '''
                echo ">>> Ejecutando tests..."
                npm run test
                '''
            }
        }

        stage('[CI] Integration tests') {
            steps {
                sh '''
                echo ">>> Ejecutando pruebas integración..."
                npm run test:integration
                '''
            }
        }

        stage('Hello world') {
            steps {

                script {
                    env.VARIABLE = "demo123"
                }

                sh '''
                echo "Hello world"
                echo $VARIABLE
                echo $APELLIDO
                '''

                sh '''
                node -v
                npm -v
                docker --version
                az version
                '''
            }
        }

        stage('Azure Login') {

            steps {

                withCredentials([

                    string(credentialsId:'azure-clientId',variable:'AZ_CLIENT_ID'),
                    string(credentialsId:'azure-clientSecret',variable:'AZ_CLIENT_SECRET'),
                    string(credentialsId:'azure-tenantId',variable:'AZ_TENANT_ID'),
                    string(credentialsId:'azure-subscriptionId',variable:'AZ_SUBSCRIPTION_ID')

                ]) {

                    sh '''

                    az login --service-principal \
                     --username "$AZ_CLIENT_ID" \
                     --password "$AZ_CLIENT_SECRET" \
                     --tenant "$AZ_TENANT_ID"

                    az account set --subscription $AZ_SUBSCRIPTION_ID

                    '''

                }
            }
        }

        stage('AKS Credentials') {

            steps {

                sh '''

                az aks get-credentials \
                --resource-group $RESOURCE_GROUP \
                --name $AKS_NAME \
                --overwrite-existing

                '''

            }
        }

        stage('[CI] Get Git Commit Short SHA') {

            steps {

                script {

                    env.IMAGE_TAG = sh(
                    script:'git rev-parse --short HEAD',
                    returnStdout:true
                    ).trim()

                    echo "IMAGE_TAG ${env.IMAGE_TAG}"

                }

            }

        }

        stage('[CI] Build & Push to ACR') {

            steps {

                sh '''

                az acr login --name $ACR_NAME

                docker build \
                 -t $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG .

                docker push \
                 $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG

                '''

            }

        }

        stage('[CD-DEV] Render k8s.yml') {

            steps {

                sh '''

                envsubst < k8s.yml > k8s-dev.yml

                cat k8s-dev.yml

                '''

            }

        }

        stage('[CD-DEV] Deploy to AKS') {

            steps {

                sh '''

                az aks command invoke \
                --resource-group $RESOURCE_GROUP \
                --name $AKS_NAME \
                --command "kubectl apply -f k8s-dev.yml" \
                --file k8s-dev.yml

                '''

            }

        }

        stage('[CD-DEV] Get LoadBalancer IP') {

            steps {

                sh '''

                SERVICE_NAME="my-nodejs-service-${APELLIDO}-${ENV}"

                LB_IP=""
                MAX_RETRIES=5
                RETRY_COUNT=0

                while [ -z "$LB_IP" ] && [ $RETRY_COUNT -lt $MAX_RETRIES ]; do

                  LB_IP=$(kubectl get svc $SERVICE_NAME -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

                  if [ -z "$LB_IP" ]; then
                    RETRY_COUNT=$((RETRY_COUNT+1))
                    echo "Esperando IP..."
                    sleep 5
                  fi

                done

                [ -z "$LB_IP" ] && exit 1

                echo "LB IP -> $LB_IP"

                '''

            }

        }

    }

}