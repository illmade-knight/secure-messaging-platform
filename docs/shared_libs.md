# **Shared Libraries Guide**

This document provides an overview of the critical shared libraries that are used across multiple microservices in this project. Adhering to the principles of these libraries is essential for maintaining consistency, stability, and maintainability.

## **1\. Protobuf Schemas (action-intention-protos)**

This repository is the **single source of truth** for all data transfer objects (DTOs) that are serialized and sent between services or between the client and services.

* **Purpose**: To provide a language-agnostic, efficient, and strongly-typed way to define the structure of our data. It eliminates entire classes of bugs related to data serialization and deserialization.
* **Key Messages**:
    * SecureEnvelope: The core wrapper for all end-to-end encrypted messages.
    * NotificationRequest: The data structure for a push notification job, consumed by the go-notification-service.
* Consumption Workflow:  
  The proto definitions are automatically compiled and published to language-specific packages by a GitHub Action. Developers should never consume the action-intention-protos repository directly. Instead, they should use the appropriate native library for their service:
    * **Go**: Import github.com/illmade-knight/go-action-intention-protos.
    * **TypeScript/Node.js**: Install and import @illmade-knight/action-intention-protos from npm.

## **2\. Go Microservice Base (go-microservice-base)**

This library provides a lightweight, foundational toolkit for building all Go microservices in the system. Its primary goal is to enforce consistency and solve common boilerplate problems.

* **Guiding Principle**: The library must remain lean and focused. A new feature should only be added if at least 80% of the microservices will need that exact functionality.
* **Core Features**:
    * **Standard Server Lifecycle**: The microservice.BaseServer provides Start() and Shutdown() methods for consistent, graceful server management.
    * **Observability**: Automatically provides standard observability endpoints:
        * GET /healthz: A liveness probe to indicate the service is running.
        * GET /readyz: A readiness probe that can be programmatically controlled to signal when the service is ready to accept traffic.
    * **JWT Authentication**: Provides a robust, production-ready JWT middleware (middleware.NewJWKSAuthMiddleware) that uses the jose library to perform asymmetric (RS256) token validation against a remote JWKS endpoint.
    * **Dynamic Discovery**: Includes the middleware.DiscoverAndValidateJWTConfig function, a critical startup check that allows a service to dynamically discover and validate its configuration against the node-identity-service.
    * **Standardized Responses**: Provides helper functions (response.WriteJSONError) for sending consistent JSON error payloads.

## **3\. Go Dataflow (go-dataflow)**

This library provides a powerful, modular toolkit for building production-grade, asynchronous data processing pipelines, primarily using Google Cloud Pub/Sub. It underpins the go-routing-service and go-notification-service.

* **Core Concept**: The library provides a set of generic building blocks (**Consumers**, **Producers**, **Transformers**, and **Processors**) that can be composed into a complete, concurrent, and resilient StreamingService or BatchingService.
* **Key Components**:
    * **messagepipeline**: This is the core package.
        * MessageConsumer: An interface for message sources (e.g., GooglePubsubConsumer).
        * MessageProducer: An interface for message sinks (e.g., GooglePubsubProducer).
        * MessageTransformer: A function type for transforming raw message bytes into structured Go types.
        * StreamProcessor / BatchProcessor: Function types that define the core business logic to be executed on a single message or a batch of messages.
        * StreamingService: A fully-managed service that wires all the components together into a concurrent pipeline, complete with graceful shutdowns.
    * **cache**: Provides interfaces and implementations for distributed caching, such as the Firestore-backed PresenceCache used by the routing service.

## **4\. Go Secure Messaging (go-secure-messaging)**

This library contains shared domain models and transport-layer logic related to the secure messaging protocol itself.

* **Purpose**: To provide a canonical, reusable implementation of core messaging concepts, ensuring that all services "speak" the same language.
* **Core Components**:
    * **urn Package**: Provides a URN type for representing Uniform Resource Names (e.g., urn:sm:user:user-123). This is the standard way to identify any entity (user, device, etc.) within the system. Using a dedicated type instead of a plain string prevents common errors.
    * **transport Package**: Provides Go-native wrapper types for the Protobuf messages. These wrappers include helper functions like ToProto() and FromProto(), which handle the conversion between the Go-native types (using urn.URN) and the Protobuf-generated types (using plain strings). This isolates the conversion logic in one place.