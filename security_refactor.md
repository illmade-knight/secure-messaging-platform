# **Security Refactor: Migrating to Asymmetric JWTs (RS256)**

## **1\. Executive Summary**

This document outlines the strategic plan to harden the security of our microservices architecture by migrating from our current symmetric JWT signing mechanism (HS256) to a more robust asymmetric system (RS256).

The current architecture relies on a single JWT\_SECRET shared across all services. If this secret is compromised in *any* service, an attacker can forge valid identity tokens for any user, compromising the entire system.

The target architecture establishes the node-identity-service as the sole guardian of a **private signing key**. All other services will only need a corresponding **public verification key**, which is not a secret. This change fundamentally improves our security posture by eliminating the shared secret liability and aligns our system with industry best practices for distributed identity.

## **2\. The Role of Google Cloud Secret Manager**

A critical question is whether using Google Cloud Secret Manager for the JWT\_SECRET negates the need for this upgrade. The answer is **no, the upgrade is still essential.**

* **What Secret Manager Does**: Google Cloud Secret Manager is the industry-standard best practice for **storing and controlling access to secrets at rest**. It ensures that only authorized services and principals can *retrieve* the secret value. This is a critical piece of a secure infrastructure.
* **The Remaining Risk**: However, Secret Manager does not protect a secret once it is loaded into the memory of a running application. In our current design, multiple services (Node.js and Go) load the same shared JWT\_SECRET into memory. If an attacker finds a remote code execution vulnerability in *any* of these consumer services, they can extract the secret from memory and use it to forge tokens.
* **Conclusion**: Using Secret Manager is a necessary but not sufficient condition for security. The architectural flaw is the **sharing of the secret itself**. The migration to asymmetric keys (RS256) is the definitive solution because it ensures that even if a consumer service is fully compromised, the attacker gains no ability to create new, valid tokens.

## **3\. Target Architecture**

The new authentication flow will be as follows:

1. The node-identity-service will be the only service with access to a **private RSA key**, stored securely in Google Cloud Secret Manager.
2. It will sign all JWTs using this private key with the RS256 algorithm.
3. It will expose a new, public, unauthenticated endpoint at /.well-known/jwks.json which allows other services to discover its **public key**.
4. All other consumer services (node-messaging-service, go-key-service, etc.) will validate incoming JWTs by:  
   a. Fetching the public key from the JWKS endpoint.  
   b. Caching the key for a short duration to minimize latency.  
   c. Verifying the JWT signature against that public key.

## **4\. Zero-Downtime Migration Plan**

This plan is designed as a phased rollout that can be executed without taking the system offline.

### **Phase 0: Preparation**

1. **Generate Key Pair**: Generate a secure 2048-bit RSA key pair.  
   openssl genpkey \-algorithm RSA \-out private\_key.pem \-pkeyopt rsa\_keygen\_bits:2048  
   openssl rsa \-pubout \-in private\_key.pem \-out public\_key.pem

2. **Store Private Key**: Store the contents of private\_key.pem as a new secret in **Google Cloud Secret Manager**. Grant the service account for the node-identity-service permission to access this new secret.

### **Phase 1: Upgrade node-identity-service (The Issuer)**

1. **Update Configuration**: Modify the service's configuration logic to securely load the new JWT\_PRIVATE\_KEY from Google Cloud Secret Manager at startup. The old JWT\_SECRET should remain for now.
2. **Modify JWT Signing**: Update the generateToken function in jwt.service.ts to sign tokens using the RS256 algorithm and the loaded private key.
3. **Create JWKS Endpoint**: Add a new, unauthenticated route in main.ts at /.well-known/jwks.json. This endpoint will serve a JSON object containing the public key in the standard [JSON Web Key Set (JWKS)](https://auth0.com/docs/secure/tokens/json-web-tokens/json-web-key-sets) format. Libraries like jwk-to-pem can help with this conversion.

At this point, the identity service will be issuing new, more secure tokens, but all consumer services will still be validating them using the old, shared secret (which will fail). The next phase resolves this. *Correction: Standard JWT libraries will validate either HS256 or RS256 if configured to do so. We will update consumers to handle both during the transition.*

### **Phase 2: Upgrade Consumer Services (The Verifiers)**

This can be performed one service at a time.

1. **Update JWT Middleware**: Refactor the JWT validation middleware in each consumer service (auth.middleware.ts and jwt.go).
2. Implement JWKS Logic: The middleware must be updated to:  
   a. Attempt validation using the new RS256 method first.  
   b. To do this, it will fetch the public key from the identity service's /.well-known/jwks.json endpoint.  
   c. It is critical to cache this key in memory for a short period (e.g., 5-15 minutes) to avoid high latency.  
   d. Use the public key to verify the token's signature.
3. **Provide Fallback (Optional but Recommended)**: For a seamless transition, the middleware can be configured to first try RS256 validation, and if that fails, fall back to trying HS256 validation with the old JWT\_SECRET. This allows you to deploy the identity service changes before the consumer changes are fully rolled out.
4. **Recommended Libraries**:
    * **Node.js**: Use the jwks-rsa library. It handles the complexity of fetching, caching, and using the public key automatically.
    * **Go**: A library like github.com/MicahParks/keyfunc can be used to manage the JWKS fetching and caching.

### **Phase 3: Final Cleanup**

1. **Confirm Migration**: Once all consumer services have been successfully updated and deployed, and you have confirmed they are validating with the new RS256 method, the final cleanup can begin.
2. **Remove Fallback Logic**: Remove the HS256 fallback logic from all consumer middleware.
3. **Deprecate the Secret**: The JWT\_SECRET environment variable can now be safely removed from the configuration of **all services**.
4. **Delete the Secret**: The old shared secret should be deleted from Google Cloud Secret Manager to complete the migration.