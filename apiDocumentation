# Api Documentation


&nbsp;  
&nbsp; 

## 1. Abstract

This thesis presents the design, implementation, and evaluation of a scalable and resilient backend system for an e-commerce platform, architected using microservices principles. The system decomposes core e-commerce functionalities—authentication, product catalog management, image handling, and search—into independently deployable services. Key technologies employed include Node.js (Express.js), PostgreSQL with Prisma ORM, Apache Kafka for asynchronous event streaming, Elasticsearch for optimized product searching, Consul for service discovery, and Docker for containerization. The architecture emphasizes loose coupling, fault isolation, and technology diversity, enabling independent scaling and development cycles for each service. This presentation details the system architecture, technology stack, service interactions, data flow, deployment strategy, and discusses the achieved benefits and potential challenges of the microservices approach in this context.

---

## 2. Introduction

*   **Problem Domain:** Traditional monolithic e-commerce backends often face challenges related to scalability, maintainability, fault tolerance, and technology lock-in as complexity grows.
*   **Proposed Solution:** A microservices architecture addresses these challenges by breaking down the application into smaller, independent, and specialized services.
*   **Project Goals:**
    *   Design and implement a robust backend for a typical e-commerce platform.
    *   Leverage microservices principles for modularity and scalability.
    *   Implement core functionalities: Authentication, Product Catalog, Image Management, Search.
    *   Ensure service discovery and resilience.
    *   Utilize asynchronous communication for decoupling critical processes (e.g., search indexing).
    *   Containerize services for consistent deployment.
*   **Scope:** Focus on the backend infrastructure, core service implementation, inter-service communication, and deployment aspects. Frontend and advanced features (e.g., complex order management, payment gateways) are outside the primary scope.

---

## 3. Background & Related Work

*   **Microservices Architecture:**
    *   Definition: An architectural style structuring an application as a collection of small, autonomous services modeled around a business domain.
    *   Benefits: Scalability, Resilience, Technology Diversity, Independent Deployment, Faster Development Cycles.
    *   Challenges: Distributed System Complexity, Network Latency, Data Consistency, Monitoring Overhead, Testing Complexity.
*   **Key Enabling Technologies:**
    *   **Containerization (Docker):** Packaging applications and dependencies, ensuring consistency across environments.
    *   **Service Discovery (Consul):** Dynamically locating service instances in a distributed environment.
    *   **API Gateways:** Single entry point, request routing, cross-cutting concerns (authentication, rate limiting).
    *   **Asynchronous Messaging (Kafka):** Decoupling services, enabling event-driven workflows, improving resilience.
    *   **Search Engines (Elasticsearch):** Providing efficient full-text search capabilities beyond traditional database queries.
    *   **ORMs (Prisma):** Simplifying database interactions and migrations.

---

## 4. System Architecture

### 4.1. High-Level Overview

The system comprises several independent microservices communicating via synchronous (REST API) and asynchronous (Kafka) methods, coordinated through an API Gateway and service discovery.

```mermaid
graph TD
    subgraph User Interaction
        User[End User / Client Application]
    end

    subgraph Infrastructure
        Consul[(Consul Agent<br/>Service Discovery)]
        Kafka[(Kafka Broker<br/>Event Stream)]
        Elasticsearch[(Elasticsearch<br/>Search Index)]
        AuthDB[(PostgreSQL<br/>Auth Database)]
        ProductDB[(PostgreSQL<br/>Product Database)]
        ImageStore[(Volume/FS<br/>Image Storage)]
    end

    subgraph Microservices Network
        APIGW(API Gateway<br/>api-gateway:3000)
        AuthSvc(Auth Service<br/>auth-service:3001)
        ProductSvc(Product Service<br/>product-service:3003)
        ImageSvc(Image Service<br/>image-service:3004)
        SearchSvc(Search Service<br/>search-service:3005)
    end

    User -- HTTPS --> APIGW

    APIGW -- REST --> AuthSvc
    APIGW -- REST --> ProductSvc
    APIGW -- REST --> ImageSvc
    APIGW -- REST --> SearchSvc

    AuthSvc -- CRUD --> AuthDB
    ProductSvc -- CRUD --> ProductDB
    ProductSvc -- Publishes Events --> Kafka
    ImageSvc -- Writes/Reads --> ImageStore

    SearchSvc -- Consumes Events --> Kafka
    SearchSvc -- Writes/Reads --> Elasticsearch
    SearchSvc -- REST (Search Query) --> Elasticsearch

    %% Service Discovery
    APIGW -.-> Consul
    AuthSvc -.-> Consul
    ProductSvc -.-> Consul
    ImageSvc -.-> Consul
    SearchSvc -.-> Consul

    Consul -- Provides Service Locations --> APIGW
    ProductSvc -- Query Auth Svc Location --> Consul

    classDef service fill:#f9f,stroke:#333,stroke-width:2px;
    classDef infra fill:#ccf,stroke:#333,stroke-width:2px;
    class APIGW,AuthSvc,ProductSvc,ImageSvc,SearchSvc service;
    class Consul,Kafka,Elasticsearch,AuthDB,ProductDB,ImageStore infra;

4.2. Components

API Gateway (api-gateway): The single entry point for all client requests. Responsible for routing requests to appropriate backend services, service discovery lookup, and potentially handling cross-cutting concerns (though primary auth validation is delegated).

Authentication Service (auth-service): Manages user registration, login, JWT generation, and token validation. Stores user credentials securely.

Product Service (product-service): Manages the product catalog, including products, categories, variants, stock levels, and product images metadata. Publishes events (e.g., PRODUCT_CREATED, PRODUCT_UPDATED) to Kafka.

Image Service (image-service): Handles image uploads, storage, and retrieval. Provides URLs for accessing images.

Search Service (search-service): Consumes product events from Kafka, indexes product data into Elasticsearch, and exposes a search API.

Consul: Service registry and discovery mechanism. Services register themselves and query Consul to find other services.

Kafka: Distributed event streaming platform used for asynchronous communication between product-service and search-service.

Elasticsearch: Search engine used by search-service for indexing and querying product data.

PostgreSQL Databases: Relational databases used by auth-service and product-service for persistent data storage.

Docker: Containerization platform for packaging and running services.

Docker Compose: Tool for defining and running the multi-container application in development.

Jenkins (Jenkinsfile): Indicates the presence of a CI/CD pipeline for automating build, test, and deployment processes.

5. Technology Stack

Backend Framework: Node.js with Express.js

Database: PostgreSQL

ORM: Prisma Client

Messaging Queue: Apache Kafka (via kafkajs)

Search Engine: Elasticsearch (via @elastic/elasticsearch)

Service Discovery: HashiCorp Consul (via consul)

API Gateway Proxy: http-proxy-middleware

Containerization: Docker, Docker Compose

CI/CD: Jenkins (based on Jenkinsfile)

Authentication: JWT (JSON Web Tokens), bcrypt (for password hashing)

File Uploads: Multer (image-service)

6. Service Details & Design
6.1. API Gateway (api-gateway)

Purpose: Centralized entry point, routing, service discovery integration.

Responsibilities:

Receive incoming client requests.

Discover downstream service instances via Consul (findService).

Route requests to the appropriate service based on path (/auth, /products, /images, /search).

Proxy requests using http-proxy-middleware.

Provide a health check endpoint (/health).

Key Components: index.js (startup), config/app.js (Express setup, proxy middleware), config/consul.js (discovery, registration).

Diagram: Request Routing Flow

sequenceDiagram
    participant Client
    participant APIGW as API Gateway
    participant Consul
    participant TargetSvc as Target Service (e.g., Product)

    Client->>+APIGW: Request (e.g., GET /products/123)
    APIGW->>+Consul: findService('product-service')
    Consul-->>-APIGW: Return healthy instance URL (e.g., http://product-svc-xyz:3003)
    APIGW->>+TargetSvc: Proxy Request (GET /id/123)
    TargetSvc-->>-APIGW: Response (Product Data)
    APIGW-->>-Client: Forward Response
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Mermaid
IGNORE_WHEN_COPYING_END
6.2. Authentication Service (auth-service)

Purpose: Handle user identity and access control.

Responsibilities:

User registration (hashing passwords with bcrypt).

User login (comparing hashed passwords, generating JWT).

Provide user profile information (/me).

Validate JWTs for internal service communication (/validate).

Register with Consul.

Key Components: index.js, config/app.js, config/prisma.js, config/consul.js, modules/auth (Controller, Routes), modules/user (Controller, Routes), middlewares/auth.js (JWT verification for /me), prisma/schema.prisma.

Database Schema (prisma/schema.prisma):

classDiagram
    class User {
        +String id (PK)
        +String name
        +String email (Unique)
        +String password
        +String role
        +DateTime createdAt
        +DateTime updatedAt
    }
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Mermaid
IGNORE_WHEN_COPYING_END
6.3. Product Service (product-service)

Purpose: Manage the complete product catalog.

Responsibilities:

CRUD operations for Products, Categories, Variants, Images (metadata).

Stock management (adjustments via StockMovement, updating Variant.stockQuantity).

Associate products with categories and images.

Publish product change events (PRODUCT_CREATED, PRODUCT_UPDATED, PRODUCT_DELETED) to Kafka.

Register with Consul.

(Potentially) Validate user tokens via auth-service for protected routes.

Key Components: index.js, config/app.js, config/prisma.js, config/consul.js, kafka/producer.js, modules/* (Controllers, Routes for Product, Category, Variant, Stock), middlewares/auth.js (token validation), prisma/schema.prisma.

Database Schema (prisma/schema.prisma):

classDiagram
    direction LR
    class Category {
        +String id (PK)
        +String name
        +String slug (Unique)
        +String? parentId (FK)
        +Boolean isLeaf
        +String? imageUrl
        +DateTime createdAt
        +DateTime updatedAt
        +Category? parent
        +Category[] children
        +ProductCategory[] products
    }
    class Product {
        +String id (PK)
        +String sku (Unique)
        +String name
        +String? description
        +Boolean isActive
        +DateTime createdAt
        +DateTime updatedAt
        +Variant[] variants
        +ProductCategory[] categories
        +ProductImage[] images
    }
    class ProductImage {
        +String id (PK)
        +String productId (FK)
        +String imageUrl
        +String? altText
        +Boolean isPrimary
        +Int? order
        +DateTime createdAt
        +Product product
    }
    class Variant {
        +String id (PK)
        +String productId (FK)
        +Json attributes
        +Decimal price
        +Decimal? costPrice
        +Int stockQuantity
        +Int? lowStockThreshold
        +DateTime createdAt
        +DateTime updatedAt
        +Product product
        +StockMovement[] stockMovements
    }
    class ProductCategory {
        +String productId (PK, FK)
        +String categoryId (PK, FK)
        +Product product
        +Category category
    }
    class StockMovement {
        +String id (PK)
        +String variantId (FK)
        +Int changeQuantity
        +StockMovementType type
        +DateTime timestamp
        +String? reason
        +String? relatedOrderId
        +Variant variant
    }
    enum StockMovementType {
        INITIAL_STOCK
        ADMIN_UPDATE
        ORDER
        ADJUSTMENT
        ORDER_CANCELLED
    }

    Category "1" --o "0..*" Category : children / parent
    Product "1" --* "*" Variant : has
    Product "1" --* "*" ProductImage : has
    Product "1" --* "*" ProductCategory : maps to
    Category "1" --* "*" ProductCategory : maps to
    Variant "1" --* "*" StockMovement : logs
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Mermaid
IGNORE_WHEN_COPYING_END
6.4. Image Service (image-service)

Purpose: Handle storage and delivery of product images.

Responsibilities:

Accept image file uploads (using Multer).

Validate file types and size.

Store images in a designated location (Docker volume image-uploads-data).

Generate unique filenames (UUID).

Serve images statically via a URL (/images/:filename).

Register with Consul.

Key Components: index.js, config/app.js (Express setup, Multer config, static serving), config/consul.js, middlewares/errorHandler.js, uploads/ directory (mapped to volume).

Storage: Uses a Docker volume (image-uploads-data) mounted to /app/uploads inside the container.

Diagram: Image Upload Flow

sequenceDiagram
    participant Client
    participant APIGW as API Gateway
    participant ImageSvc as Image Service
    participant Volume as Docker Volume

    Client->>+APIGW: POST /images/upload (multipart/form-data with imageFile)
    APIGW->>+ImageSvc: Proxy Request
    ImageSvc->>ImageSvc: Process upload via Multer
    ImageSvc->>+Volume: Store image file (e.g., uuid.jpg)
    Volume-->>-ImageSvc: File stored
    ImageSvc-->>-APIGW: Response (201 Created, { filename, imageUrl })
    APIGW-->>-Client: Forward Response
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Mermaid
IGNORE_WHEN_COPYING_END
6.5. Search Service (search-service)

Purpose: Provide efficient full-text search over the product catalog.

Responsibilities:

Consume product events (PRODUCT_CREATED, PRODUCT_UPDATED, PRODUCT_DELETED) from Kafka topic (product_events).

Index relevant product data into Elasticsearch (products index).

Handle index creation and mapping on startup (elasticsearch.js).

Expose a search API (/search/products) to query Elasticsearch based on text, filters (category, price, etc.), pagination.

Register with Consul.

Key Components: index.js, config/app.js, config/consul.js, config/elasticsearch.js (client, index mapping), kafka/consumer.js (message processing), modules/search (Controller, Routes), middlewares/errorHandler.js.

Elasticsearch Index (products): Defined in config/elasticsearch.js with mappings for fields like name, description, sku, categories, variants (nested), etc., enabling efficient searching and filtering.

7. Key Workflows & Interactions
7.1. User Registration & Login Flow
sequenceDiagram
    participant User
    participant APIGW as API Gateway
    participant AuthSvc as Auth Service
    participant AuthDB as Auth Database

    User->>+APIGW: POST /auth/register (name, email, pass)
    APIGW->>+AuthSvc: Proxy Request (/register)
    AuthSvc->>AuthSvc: Hash Password (bcrypt)
    AuthSvc->>+AuthDB: Create User Record
    AuthDB-->>-AuthSvc: User Created (ID)
    AuthSvc-->>-APIGW: Response (201 Created)
    APIGW-->>-User: Forward Response

    User->>+APIGW: POST /auth/login (email, pass)
    APIGW->>+AuthSvc: Proxy Request (/login)
    AuthSvc->>+AuthDB: Find User by Email
    AuthDB-->>-AuthSvc: Return User Record (incl. hash)
    alt User Found
        AuthSvc->>AuthSvc: Compare Password (bcrypt.compare)
        alt Password Match
            AuthSvc->>AuthSvc: Generate JWT (payload: id, email, role)
            AuthSvc-->>-APIGW: Response (200 OK, { token })
            APIGW-->>-User: Forward Response (JWT)
        else Password Mismatch
            AuthSvc-->>-APIGW: Response (401 Unauthorized)
            APIGW-->>-User: Forward Response
        end
    else User Not Found
        AuthSvc-->>-APIGW: Response (401 Unauthorized)
        APIGW-->>-User: Forward Response
    end
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Mermaid
IGNORE_WHEN_COPYING_END
7.2. Product Creation & Search Indexing Flow
sequenceDiagram
    participant User
    participant APIGW as API Gateway
    participant ProductSvc as Product Service
    participant ProductDB as Product Database
    participant Kafka
    participant SearchSvc as Search Service
    participant Elasticsearch

    User->>+APIGW: POST /products (Product Data)
    APIGW->>+ProductSvc: Proxy Request (POST /)
    ProductSvc->>+ProductDB: Create Product, Variants, etc. (Transaction)
    ProductDB-->>-ProductSvc: Records Created (Product ID)
    ProductSvc->>ProductSvc: Fetch Full Product Data for Kafka
    ProductSvc->>+Kafka: Publish 'PRODUCT_CREATED' event (Payload: Formatted Product Data, Key: Product ID)
    Kafka-->>-ProductSvc: Ack (Event Published)
    ProductSvc-->>-APIGW: Response (201 Created, Full Product Data)
    APIGW-->>-User: Forward Response

    %% Asynchronous Part
    Kafka->>+SearchSvc: Deliver 'PRODUCT_CREATED' event
    SearchSvc->>SearchSvc: Parse Event Payload
    SearchSvc->>+Elasticsearch: Index Document (index: 'products', id: Product ID, body: Payload)
    Elasticsearch-->>-SearchSvc: Indexing Confirmation
    SearchSvc-->>-Kafka: Commit Offset
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Mermaid
IGNORE_WHEN_COPYING_END
7.3. Service Discovery Flow (Example: API Gateway finding Auth Service)
sequenceDiagram
    participant APIGW as API Gateway (Startup/Request)
    participant ConsulAgent as Consul Agent
    participant AuthSvc as Auth Service (Startup)

    %% Service Registration
    AuthSvc->>+ConsulAgent: Register Service ('auth-service', ID, Address, Port, Health Check)
    ConsulAgent-->>-AuthSvc: Registration OK

    %% Service Discovery
    APIGW->>+ConsulAgent: Query Healthy Services ('auth-service')
    ConsulAgent-->>-APIGW: List of Healthy Instances (e.g., [ { Address: 'auth-svc-xyz', Port: 3001 } ])
    APIGW->>APIGW: Select Instance (e.g., random)
    APIGW->>AuthSvc: Route Request to discovered instance
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Mermaid
IGNORE_WHEN_COPYING_END
8. Infrastructure & Deployment
8.1. Containerization (Docker)

Each microservice has a Dockerfile defining its image (Base Node image, copy code, install dependencies, expose port, run command).

.dockerignore used to exclude unnecessary files (node_modules, .git, .env) from the build context.

Benefits: Consistent environments, dependency isolation, simplified deployment.

8.2. Development Environment (Docker Compose)

docker-compose.yml defines the multi-container setup for local development.

Specifies services (api-gateway, auth-service, etc.), databases (auth-db, product-db), infrastructure (consul, kafka, elasticsearch), networks, volumes, ports, and environment variables.

Uses build: . context for services to build images from local Dockerfiles.

Includes depends_on for startup order and healthchecks for service readiness.

Maps volumes for data persistence (auth-db-data, product-db-data, es-data-dev, image-uploads-data).

Uses develop: section with watch: for hot-reloading/rebuilding during development.

Diagram: Docker Compose Services & Dependencies

graph TD
    subgraph Docker Network (microservices-network)
        APIGW(api-gateway)
        AuthSvc(auth-service)
        ProductSvc(product-service)
        ImageSvc(image-service)
        SearchSvc(search-service)
        AuthDB(auth-db<br/>Postgres)
        ProductDB(product-db<br/>Postgres)
        Consul(consul)
        Kafka(kafka)
        Zookeeper(zookeeper)
        ES(elasticsearch)
    end

    subgraph Volumes
        AuthData(auth-db-data)
        ProductData(product-db-data)
        ImageData(image-uploads-data)
        ESData(es-data-dev)
    end

    APIGW --> Consul
    APIGW --> AuthSvc
    APIGW --> ProductSvc
    APIGW --> ImageSvc
    APIGW --> SearchSvc

    AuthSvc --> Consul
    AuthSvc --> AuthDB

    ProductSvc --> Consul
    ProductSvc --> ProductDB
    ProductSvc --> Kafka
    ProductSvc --> AuthSvc # For Auth Middleware

    ImageSvc --> Consul

    SearchSvc --> Consul
    SearchSvc --> Kafka
    SearchSvc --> ES

    Kafka --> Zookeeper

    AuthSvc -- Mounts --> AuthData
    ProductSvc -- Mounts --> ProductData
    ImageSvc -- Mounts --> ImageData
    ES -- Mounts --> ESData

    classDef service fill:#lightblue,stroke:#333;
    classDef infra fill:#lightgrey,stroke:#333;
    classDef db fill:#moccasin,stroke:#333;
    classDef volume fill:#honeydew,stroke:#333;

    class APIGW,AuthSvc,ProductSvc,ImageSvc,SearchSvc service;
    class Consul,Kafka,Zookeeper,ES infra;
    class AuthDB,ProductDB db;
    class AuthData,ProductData,ImageData,ESData volume;
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Mermaid
IGNORE_WHEN_COPYING_END
8.3. CI/CD (Jenkins)

Jenkinsfile indicates an automated pipeline is defined.

Likely stages (inferred):

Checkout code from version control (e.g., Git).

Build Docker images for each service.

Run tests (Unit, Integration - tests not provided but implied for a robust pipeline).

Push images to a Docker registry.

Deploy containers (e.g., to a staging or production environment using Docker Compose, Kubernetes, etc. - deployment target not specified).

9. Discussion

Benefits Achieved:

Modularity: Clear separation of concerns between services.

Scalability: Individual services can be scaled independently based on load.

Resilience: Failure in one non-critical service (e.g., search indexing delay) doesn't necessarily bring down the entire system.

Technology Flexibility: Each service uses appropriate tools (Postgres for structured data, ES for search, Kafka for events).

Independent Development & Deployment: Teams could theoretically work on and deploy services independently (facilitated by CI/CD).

Challenges & Considerations:

Complexity: Managing multiple services, databases, and infrastructure components is more complex than a monolith.

Distributed Transactions: Ensuring atomicity across service boundaries (e.g., creating an order that updates stock) requires patterns like Sagas (not implemented here).

Data Consistency: Maintaining consistency between service databases (e.g., product data in Product DB vs. Search Index) relies on the event-driven approach (eventual consistency).

Network Latency & Reliability: Inter-service communication adds network overhead and potential failure points.

Testing: End-to-end testing requires running multiple services. Integration testing is crucial.

Monitoring & Logging: Requires centralized logging and distributed tracing solutions for effective debugging.

Future Work:

Implement Order Service and Payment Integration.

Implement distributed tracing (e.g., Jaeger, OpenTelemetry).

Implement centralized logging (e.g., ELK stack, Grafana Loki).

Enhance security (input validation, rate limiting, finer-grained authorization).

Implement Saga pattern for distributed transactions.

Add comprehensive integration and end-to-end tests.

Refine CI/CD pipeline for automated deployment to a production-like environment (e.g., Kubernetes).

10. Conclusion

This project successfully demonstrated the implementation of a microservices-based backend for an e-commerce platform. By decomposing the system into specialized services (api-gateway, auth-service, product-service, image-service, search-service) and leveraging technologies like Docker, Consul, Kafka, Elasticsearch, and Prisma, the architecture achieves key microservice benefits such as modularity, independent scalability, and technological flexibility. The use of asynchronous communication via Kafka effectively decouples the product catalog updates from the search indexing process, contributing to system resilience. While introducing complexities inherent in distributed systems, the chosen architecture provides a solid foundation for a scalable and maintainable e-commerce backend.

11. References

[List any academic papers, books, or significant online resources consulted]

Newman, S. (2015). Building Microservices. O'Reilly Media.

Docker Documentation: https://docs.docker.com/

Consul Documentation: https://www.consul.io/docs

Kafka Documentation: https://kafka.apache.org/documentation/

Elasticsearch Documentation: https://www.elastic.co/guide/index.html

Prisma Documentation: https://www.prisma.io/docs/

12. Q & A

Thank You
