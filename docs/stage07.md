#### **ARCH-7: Introduce an API Gateway**

*   **Problem:** Our current architecture exposes multiple services (`Frontend`, `Backend`, `Search`) directly to the internet. This forces the client (the user's browser) to know the addresses of multiple services, which is brittle and complex. It also creates a large attack surface and lacks a centralized location for handling cross-cutting concerns like SSL termination, rate limiting, and request logging.

*   **Solution:** Introduce an **API Gateway** as the single entry point for all incoming traffic. This gateway will act as a reverse proxy. The user's browser will only ever communicate with the API Gateway. The gateway will then be responsible for:
    1.  Serving the static assets of the **Frontend Service**.
    2.  Routing API requests (e.g., `/api/posts/*`) to the **Backend Service**.
    3.  Routing search requests (e.g., `/api/search/*`) to the **Search Service**.
    This simplifies the client logic, reduces the attack surface by hiding internal services, and provides a single control point for future enhancements.

*   **Trade-offs:**
    *   **Pros:**
        *   **Single Entry Point:** Simplifies client configuration and network security rules.
        *   **Centralized Concerns:** Provides a single place to manage SSL, authentication checks, rate limiting, and logging.
        *   **Decoupling:** Hides the internal service architecture from the client, allowing us to refactor or move services without impacting the user.
    *   **Cons:**
        *   **New Component to Manage:** Adds another piece of infrastructure that needs to be configured, deployed, and monitored.
        *   **Potential Bottleneck / Single Point of Failure:** All traffic now flows through the gateway. If it is not configured for high availability and performance (which we will address in later issues), it can become a critical failure point.

---

#### **Logical View (C4 Component Diagram)**

The logical view now includes the `API Gateway`, which sits at the edge of the system and mediates all communication.

```mermaid
C4Container
    title Logical Architecture for ClassifyHub - API Gateway

    Person(user, "Authenticated User", "A user browsing the application.")

    System_Boundary(c1, "ClassifyHub System") {
        Container(api_gateway, "API Gateway", "e.g., Nginx, Kong", "The single entry point for all traffic. Routes requests to the appropriate internal service.")
        Container(frontend_service, "Frontend Service", "Docker Container (React SPA served by Nginx)", "Provides the user interface (served via the Gateway).")
        Container(backend_service, "Backend Service", "Internal Docker Container", "Handles business logic (requests are routed from the Gateway).")
        Container(search_service, "Search Service", "Internal Docker Container", "Provides the full-text search API (requests are routed from the Gateway).")
        Container(cache_service, "Cache Service", "Docker Container (Redis)", "Stores frequently accessed post data.")
        ContainerDb(sql_database, "SQL Database", "Docker Container (PostgreSQL)", "The single source of truth for all data.")
        ContainerDb(object_store, "Object Store", "AWS S3", "Stores all user-uploaded images.")
    }
    
    Rel(user, api_gateway, "Makes all requests to", "HTTPS")
    Rel(api_gateway, frontend_service, "Serves static UI assets from", "HTTPS")
    Rel(api_gateway, backend_service, "Routes API requests to", "HTTPS (JWT)")
    Rel(api_gateway, search_service, "Routes search queries to", "HTTPS")

    Rel(backend_service, cache_service, "Reads/Writes to cache", "TCP")
    Rel(backend_service, sql_database, "Reads/Writes to database", "TCP")
    Rel(backend_service, search_service, "Updates search index", "HTTPS")
    Rel(backend_service, object_store, "Writes image files", "HTTPS")
    
    Rel(user, object_store, "Reads images", "HTTPS")
```

---

#### **Physical View (AWS Deployment Diagram)**

The physical view is significantly cleaned up. All traffic from the user's browser now targets the `API Gateway Container`.

```mermaid
graph TD
    %% --- Define Actors ---
    subgraph "Internet"
        user_client["User's Browser"]
    end

    %% --- Define Cloud Infrastructure ---
    subgraph "AWS Cloud"
        s3_bucket["S3 Bucket <br/> (Object Store)"]

        subgraph "VPC"
            subgraph "Public Subnet"
                subgraph ec2_instance ["EC2 Instance (t3.micro)"]
                    docker["Docker Engine"]
                    
                    api_gateway_container["API Gateway Container"]
                    frontend_container["Frontend Service Container"]
                    backend_container["Backend Service Container"]
                    cache_container["Cache Service Container"]
                    search_container["Search Service Container"]
                    db_container["PostgreSQL Container"]
                end
            end
        end
    end

    %% --- Define Relationships ---
    %% 1. ALL user traffic now goes through the API Gateway
    user_client -- "HTTPS (Port 443)" --> api_gateway_container
    
    %% 2. Gateway routes traffic to internal services
    api_gateway_container -- "Serves UI" --> frontend_container
    api_gateway_container -- "Routes API Calls" --> backend_container
    api_gateway_container -- "Routes Search Queries" --> search_container

    %% 3. Backend still performs its functions
    backend_container -- "Check Cache" --> cache_container
    backend_container -- "R/W DB" --> db_container
    backend_container -- "Update Index" --> search_container
    backend_container -- "Uploads to S3" --> s3_bucket
    
    %% 4. User still displays images directly from S3
    user_client -- "Displays Images" --> s3_bucket

    %% --- Styling ---
    classDef aws_cloud fill:#FF9900,stroke:#333,stroke-width:2px;
    classDef vpc fill:#f2f2f2,stroke:#333;
    classDef ec2 fill:#D2F2D2,stroke:#333;
    classDef s3 fill:#5A97DB,stroke:#333,stroke-width:2px;

    class AWS_Cloud aws_cloud
    class VPC vpc
    class ec2_instance ec2
    class s3_bucket s3
```

---

#### **Component-to-Resource Mapping Table**

We add the new `API Gateway` to our mapping table.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **API Gateway** | Docker Container (e.g., Nginx, Kong) on a single EC2 Instance | Centralizes ingress traffic, providing a single point of control for routing, security, and other cross-cutting concerns. Co-locating it maintains simplicity for this stage. |
| **Frontend Service** | Docker Container (Nginx) on a single EC2 Instance | No change. Now an internal service whose assets are served via the gateway. |
| **Backend Service** | Docker Container on a single EC2 Instance | No change. Now an internal service, reducing its direct exposure to the internet. |
| **Cache Service** | Docker Container (Redis) on a single EC2 Instance | No change. |
| **Search Service** | Docker Container (Elasticsearch) on a single EC2 Instance | No change. Now an internal service, reducing its direct exposure. |
| **SQL Database** | Docker Container (PostgreSQL) on the same EC2 Instance | No change. |
| **Object Store** | AWS S3 Bucket | No change. |
