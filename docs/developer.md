# **Developer Getting Started Guide**

This guide provides a step-by-step tutorial to configure and run the entire secure messaging system on a local development machine.

## **1\. Prerequisites**

Before you begin, ensure you have the following tools installed and configured on your system.

* **Node.js**: v20.x or later
* **Angular CLI**: v17.x or later
* **Go**: v1.21 or later
* **Google Cloud SDK (gcloud)**: Authenticated to your GCP project (gcloud auth application-default login).
* **Docker**: For running a Firestore emulator (optional but recommended).

## **2\. Configuration**

Each service requires its own configuration file for environment variables and secrets. It is critical to set these up correctly.

### **Angular Frontend (ng-action-intention)**

1. Navigate to ng-action-intention/src/app/config/.
2. Edit environment.ts to ensure the service URLs point to the correct local ports.  
   // environment.ts  
   export const environment \= {  
   production: false,  
   identityServiceUrl: 'http://localhost:3000',  
   messagingServiceUrl: 'http://localhost:3001',  
   keyServiceUrl: 'http://localhost:8081',  
   routingServiceUrl: 'http://localhost:8082',  
   };

### **Node.js Identity Service (node-identity-service)**

1. In the service's root directory, copy the example environment file: cp .env.example .env.
2. Edit the .env file and fill in all required values.
    * ISSUER: The public URL of this service (e.g., http://localhost:3000).
    * GCP\_PROJECT\_ID: Your Google Cloud Project ID.
    * GOOGLE\_CLIENT\_ID & GOOGLE\_CLIENT\_SECRET: Your OAuth 2.0 credentials from the Google Cloud Console.
    * JWT\_PRIVATE\_KEY: A valid RSA private key in PEM format. **Must be wrapped in double quotes to preserve newlines.**
    * Generate secure random strings for SESSION\_SECRET and INTERNAL\_API\_KEY.

### **Node.js Messaging Service (node-messaging-service)**

1. In the service's root directory, copy the example file: cp .env.example .env.
2. Edit the .env file.
    * GCP\_PROJECT\_ID: Your Google Cloud Project ID.
    * IDENTITY\_SERVICE\_URL: The full URL of the running node-identity-service (e.g., http://localhost:3000).
    * INTERNAL\_API\_KEY: Must be the **exact same value** as in the node-identity-service .env file.

### **Go Services (go-key-service, go-routing-service, go-notification-service)**

The Go services load configuration from both a .yaml file and environment variables.

1. **Set Environment Variables**: These are required for all Go services.  
   export GCP\_PROJECT\_ID="your-gcp-project-id"  
   export IDENTITY\_SERVICE\_URL="http://localhost:3000"

2. **Review local.yaml / config.yaml**: Check the cmd/.../ directory in each Go service. The local configuration files should be pre-configured for local development, but you can review them to see which ports and Pub/Sub topics are being used.

## **3\. Startup Sequence**

To ensure a stable launch, services must be started in the correct order due to dependencies.

1. **Start the Firestore Emulator (Optional)**: If you are using the emulator, start it first.  
   gcloud emulators firestore start \--host-port=8088

2. **Start the Identity Service**: This service must be running before any others can authenticate.  
   cd node-identity-service/  
   npm run dev

3. **Start the Other Backend Services**: The remaining backend services can be started in any order. Open a new terminal for each one.  
   \# Terminal 2: Messaging Service  
   cd node-messaging-service/  
   npm run dev

   \# Terminal 3: Key Service  
   cd go-key-service/  
   go run ./cmd/keyservice/runscalablekeyservice.go

   \# Terminal 4: Routing Service  
   cd go-routing-service/  
   go run ./cmd/runroutingservice/runroutingservice.go

   \# Terminal 5: Notification Service  
   cd go-notification-service/  
   go run ./cmd/runnotificationservice/runnotificationservice.go

4. **Start the Angular Frontend**: Once all backend services are running, start the client application.  
   cd ng-action-intention/  
   ng serve

## **4\. Smoke Test**

After all services are running, perform the following steps to verify that the entire system is working correctly.

1. Open your browser and navigate to http://localhost:4200.
2. You should be redirected to the login page. Click "Login with Google" and complete the authentication flow.
3. Upon successful login, you should be redirected to the main messaging interface.
4. Navigate to the **Settings** page. Click **"Generate and Store Keys"**. The status should update to show that a key has been found.
5. Navigate back to the **Messaging** page. In the "Add New Contact" section, enter the email address of another authorized user and click "Add Contact". The status log should confirm the contact was added and your address book should update.
6. Select the new contact from the recipient dropdown, type a message, and click "Send Message". The status log should confirm the message was sent successfully.

If all these steps complete without error, your local development environment is fully configured and operational.