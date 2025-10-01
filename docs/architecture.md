# **System Architecture Overview**

This document provides a detailed overview of the secure messaging system's architecture, its components, and the core data flows for authentication and message processing.

## **System Components Diagram**

The architecture is composed of an Angular frontend and five distinct backend microservices that communicate via HTTP APIs and an asynchronous Pub/Sub message bus. External dependencies include Google Cloud services like Firestore and Pub/Sub, as well as Google's OAuth 2.0 for identity verification.


````mermaid
graph TD
    subgraph User Interaction
        User[End User] --> Angular[Angular Frontend]
    end

    subgraph Authentication
        Angular -- 1. Login Redirect --> IdentityService[Node Identity Service]
        IdentityService -- 2. OAuth Flow --> Google[Google OAuth]
        Google -- 3. Callback --> IdentityService
        IdentityService -- 4. Authorize User --> Firestore[(Firestore)]
        IdentityService -- 5. Redirect w/ Session --> Angular
        Angular -- 6. Get JWT (REST API) --> IdentityService
        IdentityService -- 7. Issue JWT --> Angular
    end

    subgraph Backend Services
        Angular -- REST API + JWT --> MessagingService[Node Messaging Service]
        MessagingService -- CRUD --> Firestore

        Angular -- REST API + JWT --> KeyService[Go Key Service]
        KeyService -- CRUD --> Firestore

        Angular -- REST API + JWT (Send/Get Offline) --> RoutingAPI[Go Routing Service - API]
        Angular -- WebSocket --> RoutingRealtime[Go Routing Service - Real-time]

        RoutingAPI -- Publishes to --> IngressTopic([Pub/Sub: Ingress Topic])
        RoutingAPI -- Stores/Retrieves --> Firestore

        IngressTopic -- Consumed by --> RoutingProcessor{Routing Processor}
        RoutingProcessor -- Checks --> Firestore
        
        RoutingProcessor -- Publishes to --> DeliveryBus([Pub/Sub: Delivery Bus])
        RoutingProcessor -- Publishes to --> PushTopic([Pub/Sub: Push Topic])
        
        DeliveryBus -- Consumed by --> RoutingRealtime
        PushTopic -- Consumed by --> NotificationService[Go Notification Service]
    end

    style User fill:#d4edda,stroke:#155724
    style Angular fill:#cce5ff,stroke:#004085
    style IdentityService fill:#f8d7da,stroke:#721c24
    style MessagingService fill:#f8d7da,stroke:#721c24
    style KeyService fill:#fff3cd,stroke:#856404
    style RoutingAPI fill:#fff3cd,stroke:#856404
    style RoutingRealtime fill:#fff3cd,stroke:#856404
    style NotificationService fill:#fff3cd,stroke:#856404
    style Google fill:#e2e3e5,stroke:#383d41
    style Firestore fill:#e2e3e5,stroke:#383d41
    style IngressTopic fill:#d1ecf1,stroke:#0c5460
    style DeliveryBus fill:#d1ecf1,stroke:#0c5460
    style PushTopic fill:#d1ecf1,stroke:#0c5460
````

## **Component Responsibilities**

* **Angular Frontend**: A standalone web application responsible for the entire user experience. It performs all client-side cryptographic operations (key generation, encryption, decryption), ensuring that unencrypted message content never leaves the user's device.
* **Node.js Identity Service**: The central authentication authority. Built with Node.js, Express, and Passport.js, its sole responsibility is to verify a user's identity against Google, check if they are authorized to use the application, and issue short-lived, asymmetrically signed (RS256) JSON Web Tokens (JWTs). It is the single source of truth for user identity.
* **Node.js Messaging Service**: A simple resource server that provides supporting business logic, primarily managing the user's address book. It performs server-to-server communication with the Identity Service to look up user profiles when a contact is added.
* **Go Key Service**: A high-performance Go microservice for storing and retrieving public keys. It provides a secure, URN-based API for clients to upload their public keys and discover the public keys of other users, which is essential for establishing encrypted communication.
* **Go Routing Service**: The core message bus of the system. This event-driven Go service ingests all messages and intelligently routes them. It uses a presence cache to determine if a recipient is online or offline. Online messages are sent to a real-time delivery topic, while offline messages are persisted to Firestore and a notification event is dispatched.
* **Go Notification Service**: A headless, single-responsibility Go microservice that acts as a backend consumer. It listens to a Pub/Sub topic for notification events, transforms them into the appropriate format for native mobile platforms, and dispatches them to services like Apple Push Notification Service (APNS) or Firebase Cloud Messaging (FCM).

## **Core Data Flows**

### **Authentication & Authorization**

The system uses a standard OAuth2 flow with a custom authorization step and JWT issuance for internal authentication.

1. **Login Request**: The Angular client redirects the user to the node-identity-service, which in turn redirects them to Google's OAuth consent screen.
2. **Google Callback**: After successful authentication, Google redirects the user back to the Identity Service with an authorization code.
3. **Authorization Check**: The Identity Service exchanges the code for a Google ID token and verifies its authenticity. It then extracts the user's email and queries a authorized\_users collection in Firestore. If the user is not in this collection, the login flow is terminated.
4. **Session & JWT Issuance**: If the user is authorized, the Identity Service creates a session for itself and generates a new, internal JWT. This JWT is signed with a private RS256 key. The token contains the user's unique ID, alias, and other claims.
5. **Token to Client**: The client, in a subsequent request, receives this JWT and stores it securely.
6. **Authenticated API Calls**: For all subsequent requests to other microservices (Key Service, Routing Service, etc.), the client includes the JWT in the Authorization: Bearer \<token\> header.
7. **JWT Validation**: The resource server (e.g., go-key-service) receives the request. On startup, it will have already fetched the public key set from the Identity Service's public /.well-known/jwks.json endpoint. It uses the appropriate public key to cryptographically verify the JWT's signature. If valid, the request is trusted and processed.

### **Message Processing Pipeline**

The system uses an event-driven pipeline to ensure messages are delivered reliably and scalably.

1. **Ingestion**: A client sends an encrypted message to the go-routing-service's POST /send endpoint. The service validates the user's JWT, wraps the message in a new event envelope, and immediately publishes it to a "message-ingress" Pub/Sub topic.
2. **Processing**: A background worker in the go-routing-service consumes the message from the ingress topic. It checks a presence cache (e.g., Firestore) to determine if the recipient is online.
3. **Online Routing**: If the user is online, the message is published to a "realtime-delivery" Pub/Sub topic. The specific WebSocket server instance connected to that user consumes the message and delivers it over the WebSocket.
4. **Offline Routing**: If the user is offline, two actions occur:
    * The message is written to a messages collection in Firestore, keyed by the recipient's ID.
    * A new, smaller NotificationRequest event is published to a "push-notifications" Pub/Sub topic.
5. **Push Notification**: The go-notification-service consumes the event from the "push-notifications" topic, determines the user's device platform (iOS/Android), and dispatches a native push notification via the appropriate service (APNS/FCM).
6. **Offline Retrieval**: When the user comes back online, their client makes a GET /messages request to the go-routing-service, which retrieves all persisted messages from Firestore and delivers them.
