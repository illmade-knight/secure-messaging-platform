# **API Reference**

This document provides a reference for all public HTTP endpoints exposed by the secure messaging system. All endpoints that require authentication expect a JSON Web Token (JWT) to be provided in the Authorization header with the Bearer scheme.

We also support an OpenAPI [page](index.html) generated from [openapi](openapi.yaml)

## **Identity Service (node-identity-service)**

**Base URL**: http://localhost:3000

### **Authentication**

GET /auth/google

* **Description**: Initiates the Google OAuth 2.0 login flow. This is intended to be used via direct browser redirection, not a programmatic API call.
* **Authentication**: None.

GET /auth/google/callback

* **Description**: The callback URL that Google redirects to after a user grants consent. Handled by the service's session management.
* **Authentication**: None.

POST /api/auth/logout

* **Description**: Logs the user out by destroying their session cookie.
* **Authentication**: Cookie-based session.
* **Success Response**: 200 OK
* **Error Response**: 500 Internal Server Error

### **API**

GET /api/auth/status

* **Description**: Checks the user's current session status. If the user has a valid session cookie, this endpoint returns their profile data along with a short-lived JWT for use with other microservices.
* **Authentication**: Cookie-based session.
* **Success Response (200 OK)**:  
  {  
  "authenticated": true,  
  "user": {  
  "id": "firestore-doc-id",  
  "email": "user@example.com",  
  "alias": "UserAlias",  
  "token": "a.jwt.string"  
  }  
  }

GET /api/users/by-email/:email

* **Description**: (Internal) Finds a user's public profile by their email. This endpoint is for server-to-server communication only.
* **Authentication**: Requires an X-Internal-API-Key header.
* **Success Response (200 OK)**:  
  {  
  "id": "firestore-doc-id",  
  "email": "user@example.com",  
  "alias": "UserAlias"  
  }

* **Error Response**: 404 Not Found

## **Messaging Service (node-messaging-service)**

**Base URL**: http://localhost:3001

GET /api/address-book

* **Description**: Retrieves the authenticated user's address book.
* **Authentication**: Bearer \<token\>
* **Success Response (200 OK)**:  
  \[  
  {  
  "id": "contact-user-id",  
  "email": "contact@example.com",  
  "alias": "ContactAlias"  
  }  
  \]

POST /api/address-book/contacts

* **Description**: Adds a new contact to the user's address book by email.
* **Authentication**: Bearer \<token\>
* **Request Body**:  
  {  
  "email": "newcontact@example.com"  
  }

* **Success Response (201 Created)**: Returns the full profile of the added contact.

## **Key Service (go-key-service)**

**Base URL**: http://localhost:8081

GET /keys/{entityURN}

* **Description**: Retrieves the public key for a given entity (e.g., urn:sm:user:user-123).
* **Authentication**: None.
* **Success Response (200 OK)**: Returns the raw public key bytes in an application/octet-stream.

POST /keys/{entityURN}

* **Description**: Stores or overwrites the public key for an entity.
* **Authentication**: Bearer \<token\>. The user ID in the token (sub claim) must match the entity ID in the URN.
* **Request Body**: The raw public key bytes (application/octet-stream).
* **Success Response**: 201 Created
* **Error Responses**: 403 Forbidden if the token user does not match the URN.

## **Routing Service (go-routing-service)**

**Base URL**: http://localhost:8082

POST /send

* **Description**: Ingests a secure message envelope for processing and routing.
* **Authentication**: Bearer \<token\>.
* **Request Body**: A Protobuf-serialized SecureEnvelope message, sent as application/json.
* **Success Response**: 202 Accepted. This indicates the message has been accepted for asynchronous processing.

GET /messages

* **Description**: Retrieves any messages that were stored for the authenticated user while they were offline.
* **Authentication**: Bearer \<token\>.
* **Success Response (200 OK)**: A Protobuf-serialized SecureEnvelopeList message, sent as application/json.
* **Success Response (204 No Content)**: If there are no messages.