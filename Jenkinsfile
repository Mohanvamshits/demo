pipeline {
    // You can specify a docker agent here if needed (e.g., hashicorp/terraform or google/cloud-sdk), 
    // but typically standard Jenkins nodes have these tools pre-installed.
    agent any

    environment {
        // You will need to add these credentials in the Jenkins Manage Credentials dashboard
        PROJECT_ID = credentials('gke-project-id')            // Secret text
        GOOGLE_APPLICATION_CREDENTIALS = credentials('gcp-credentials-json') // Secret file
        
        GKE_CLUSTER = 'demo-gke-cluster'
        GKE_ZONE = 'us-central1'
        IMAGE_NAME = 'demo-app'
    }

    stages {
        stage('Checkout Repo') {
            steps {
                checkout scm
            }
        }

        stage('Authenticate to GCP') {
            steps {
                sh '''
                echo "Authenticating Google Cloud CLI..."
                gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
                gcloud config set project $PROJECT_ID
                gcloud auth configure-docker gcr.io --quiet
                '''
            }
        }

        stage('Infrastructure via Terraform') {
            steps {
                dir('terraform') {
                    sh '''
                    terraform init
                    terraform apply -auto-approve -var="project_id=$PROJECT_ID"
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                # Use the Jenkins built-in GIT_COMMIT hash for image versioning
                docker build -t gcr.io/$PROJECT_ID/$IMAGE_NAME:$GIT_COMMIT .
                '''
            }
        }

        stage('Push Docker Image to GCR') {
            steps {
                sh '''
                docker push gcr.io/$PROJECT_ID/$IMAGE_NAME:$GIT_COMMIT
                '''
            }
        }

        stage('Deploy Application to GKE') {
            steps {
                sh '''
                echo "Extracting GKE Cluster Credentials..."
                gcloud container clusters get-credentials $GKE_CLUSTER --zone $GKE_ZONE --project $PROJECT_ID
                
                echo "Updating Image tag in Kubernetes Manifest..."
                sed -i "s|TARGET_IMAGE|gcr.io/$PROJECT_ID/$IMAGE_NAME:$GIT_COMMIT|g" k8s/deployment.yaml
                
                echo "Applying Kubernetes configs..."
                kubectl apply -f k8s/deployment.yaml
                kubectl apply -f k8s/service.yaml
                
                echo "Checking Rollout Status..."
                kubectl rollout status deployment/demo-app-deployment
                '''
            }
        }
    }
}
