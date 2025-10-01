# **Deployment Guide**

This guide provides a handbook for DevOps and SREs on deploying the secure messaging system to a production environment on Google Cloud Platform (GCP).

## **1\. Prerequisites**

Before deploying, ensure you have the following:

* A Google Cloud Project with billing enabled.
* The gcloud CLI installed and authenticated.
* Docker installed locally for building images.
* An Artifact Registry repository in your GCP project to host the container images.

## **2\. Infrastructure Setup**

The following GCP services must be enabled and configured.

### **a. Identity and Access Management (IAM)**

Create a dedicated service account for the microservices. This service account will need the following roles:

* **Firestore User**: For database access.
* **Pub/Sub Publisher**: To publish messages to topics.
* **Pub/Sub Subscriber**: To consume messages from subscriptions.

### **b. Firestore**

Enable the Firestore API in your GCP project and create a Firestore database in Native mode.

### **c. Pub/Sub**

Create the following Pub/Sub topics. The services are designed to create their own subscriptions idempotently on startup, but the topics must exist.

* message-ingress
* message-ingress-dlq (Dead-Letter Queue)
* realtime-delivery
* push-notifications
* push-notifications-dlq (Dead-Letter Queue)

### **d. Secret Manager**

Store all production secrets in Google Secret Manager. The services will require access to secrets for:

* GOOGLE\_CLIENT\_ID
* GOOGLE\_CLIENT\_SECRET
* SESSION\_SECRET
* JWT\_PRIVATE\_KEY
* INTERNAL\_API\_KEY

## **3\. Building Production Images**

Each microservice has a Dockerfile for building a production-ready container image.

From the root directory of each service (e.g., node-identity-service/), run the following commands:

\# Example for the identity service  
export PROJECT\_ID="your-gcp-project-id"  
export REGISTRY\_REPO="your-artifact-registry-repo"  
export SERVICE\_NAME="node-identity-service"

\# Build the Docker image  
docker build \-t ${SERVICE\_NAME}:latest .

\# Tag the image for Artifact Registry  
docker tag ${SERVICE\_NAME}:latest us-central1-docker.pkg.dev/${PROJECT\_ID}/${REGISTRY\_REPO}/${SERVICE\_NAME}:latest

\# Push the image to the registry  
docker push us-central1-docker.pkg.dev/${PROJECT\_ID}/${REGISTRY\_REPO}/${SERVICE\_NAME}:latest

Repeat this process for all five backend microservices.

## **4\. Deployment to Cloud Run**

We will deploy each microservice as a separate, fully-managed Cloud Run service.

### **Environment Variables**

When deploying each service, you must configure its environment variables. Some will be plain text, while others should be securely mounted from Secret Manager.

**Example: node-identity-service**

* PORT: 8080 (Cloud Run's default)
* NODE\_ENV: production
* GCP\_PROJECT\_ID: (Plain text) your-gcp-project-id
* ISSUER: (Plain text) The public URL provided by the Cloud Run service itself.
* GOOGLE\_CLIENT\_ID: (From Secret Manager)
* GOOGLE\_CLIENT\_SECRET: (From Secret Manager)
* JWT\_PRIVATE\_KEY: (From Secret Manager)
* SESSION\_SECRET: (From Secret Manager)
* INTERNAL\_API\_KEY: (From Secret Manager)

### **Example gcloud Deploy Command**

This command deploys the node-identity-service, setting environment variables and mounting secrets.

gcloud run deploy node-identity-service \\  
\--image=us-central1-docker.pkg.dev/your-gcp-project-id/your-repo/node-identity-service:latest \\  
\--platform=managed \\  
\--region=us-central1 \\  
\--allow-unauthenticated \\  
\--set-env-vars="GCP\_PROJECT\_ID=your-gcp-project-id,NODE\_ENV=production" \\  
\--set-secrets="GOOGLE\_CLIENT\_ID=google-client-id:latest,GOOGLE\_CLIENT\_SECRET=google-client-secret:latest,JWT\_PRIVATE\_KEY=jwt-private-key:latest,SESSION\_SECRET=session-secret:latest,INTERNAL\_API\_KEY=internal-api-key:latest"

You will need to adapt this command for each of the five microservices, ensuring they are configured with the correct environment variables (like the IDENTITY\_SERVICE\_URL for the other services).