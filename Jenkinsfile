pipeline {

agent {

docker {

image 'devops-agent:latest'

}

}

environment {

APELLIDO="vallejomerchan"

ACR_NAME="acrglobalcicd"
ACR_LOGIN_SERVER="${ACR_NAME}.azurecr.io"

IMAGE_NAME="my-nodejs-app-${APELLIDO}"

RESOURCE_GROUP="rg-cicd-terraform-app-baraujox"
AKS_NAME="aks-dev-eastus"

}

stages {


// ================= CI =================


stage('[CI] Install dependencies') {

steps {

sh 'npm install'

}

}


stage('[CI] Unit Tests') {

steps {

sh 'npm run test'

}

}


stage('[CI] Integration Tests') {

steps {

sh 'npm run test:integration'

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



stage('[CI] Git SHA') {

steps {

script {

env.IMAGE_TAG=sh(

script:'git rev-parse --short HEAD',

returnStdout:true

).trim()

}

}

}



stage('[CI] Build Push ACR') {

steps {

sh '''

az acr login --name $ACR_NAME

docker build -t $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG .

docker push $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG

'''

}

}



// ================= DEV =================


stage('[CD-DEV] Deploy') {

environment {

ENV="dev"
API_PROVIDER_URL="https://dev.api.com"

}

steps {

sh '''

envsubst < k8s.yml > k8s-dev.yml

az aks command invoke \
--resource-group $RESOURCE_GROUP \
--name $AKS_NAME \
--command "kubectl apply -f k8s-dev.yml" \
--file k8s-dev.yml

'''

}

}



stage('[CD-DEV] Get IP') {

environment { ENV="dev" }

steps {

sh '''

SERVICE_NAME="my-nodejs-service-${APELLIDO}-${ENV}"

echo "Esperando IP DEV..."

LB_IP=""

MAX=30
COUNT=0

while [ -z "$LB_IP" ] && [ $COUNT -lt $MAX ]; do

LB_IP=$(az aks command invoke \
--resource-group $RESOURCE_GROUP \
--name $AKS_NAME \
--command "kubectl get svc $SERVICE_NAME -o jsonpath='{.status.loadBalancer.ingress[0].ip}'" \
--query logs -o tsv | tr -d '\\r')

if [ -z "$LB_IP" ]; then

COUNT=$((COUNT+1))

echo "Intento $COUNT/$MAX..."

sleep 15

fi

done

[ -z "$LB_IP" ] && exit 1

echo "DEV LB IP -> $LB_IP"

'''

}

}



// ================= QA =================


stage('Aprobación QA') {

steps {

input message:'Deploy QA?', ok:'Deploy QA'

}

}



stage('[CD-QA] Deploy') {

environment {

ENV="qa"
API_PROVIDER_URL="https://qa.api.com"

}

steps {

sh '''

envsubst < k8s.yml > k8s-qa.yml

az aks command invoke \
--resource-group $RESOURCE_GROUP \
--name $AKS_NAME \
--command "kubectl apply -f k8s-qa.yml" \
--file k8s-qa.yml

'''

}

}



stage('[CD-QA] Get IP') {

environment { ENV="qa" }

steps {

sh '''

SERVICE_NAME="my-nodejs-service-${APELLIDO}-${ENV}"

echo "Esperando IP QA..."

LB_IP=""

MAX=35
COUNT=0

while [ -z "$LB_IP" ] && [ $COUNT -lt $MAX ]; do

LB_IP=$(az aks command invoke \
--resource-group $RESOURCE_GROUP \
--name $AKS_NAME \
--command "kubectl get svc $SERVICE_NAME -o jsonpath='{.status.loadBalancer.ingress[0].ip}'" \
--query logs -o tsv | tr -d '\\r')

if [ -z "$LB_IP" ]; then

COUNT=$((COUNT+1))

echo "Intento $COUNT/$MAX..."

sleep 15

fi

done

[ -z "$LB_IP" ] && exit 1

echo "QA LB IP -> $LB_IP"

'''

}

}



// ================= PRD =================


stage('Aprobación PRD') {

steps {

input message:'Deploy PRODUCCIÓN?', ok:'Deploy PRD'

}

}



stage('[CD-PRD] Deploy') {

environment {

ENV="prd"
API_PROVIDER_URL="https://api.com"

}

steps {

sh '''

envsubst < k8s.yml > k8s-prd.yml

az aks command invoke \
--resource-group $RESOURCE_GROUP \
--name $AKS_NAME \
--command "kubectl apply -f k8s-prd.yml" \
--file k8s-prd.yml

'''

}

}



stage('[CD-PRD] Get IP') {

environment { ENV="prd" }

steps {

sh '''

SERVICE_NAME="my-nodejs-service-${APELLIDO}-${ENV}"

echo "Esperando IP PRD..."

LB_IP=""

MAX=40
COUNT=0

while [ -z "$LB_IP" ] && [ $COUNT -lt $MAX ]; do

LB_IP=$(az aks command invoke \
--resource-group $RESOURCE_GROUP \
--name $AKS_NAME \
--command "kubectl get svc $SERVICE_NAME -o jsonpath='{.status.loadBalancer.ingress[0].ip}'" \
--query logs -o tsv | tr -d '\\r')

if [ -z "$LB_IP" ]; then

COUNT=$((COUNT+1))

echo "Intento $COUNT/$MAX..."

sleep 20

fi

done

[ -z "$LB_IP" ] && exit 1

echo "PRD LB IP -> $LB_IP"

'''

}

}

}

}