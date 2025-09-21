#### **ARCH-11: Add Rate Limiting at the API Gateway**

*   **Problem:** The API Gateway, as our single entry point, currently accepts an unlimited number of requests from any given client. This leaves our system vulnerable to various forms of abuse, including Denial-of-Service (DoS) attacks and runaway client scripts, which could overwhelm downstream services and degrade performance for all users (`NFR-4.4`).

*   **Solution:** We will enhance the existing **API Gateway** by adding rate-limiting logic. To do this efficiently, the gateway will use our existing **Cache Service** (Redis) as a high-speed counter. For each incoming API request, the gateway will:
    1.  Identify the client (e.g., by IP address).
    2.  Perform a quick check/increment operation in Redis to track the client's request count within a sliding time window.
    3.  If the count exceeds a predefined limit (e.g., 100 requests/minute), the gateway will immediately reject the request with a `429 Too Many Requests` HTTP status code, preventing it from ever reaching our backend services.

*   **Trade-offs:**
    *   **Pros:**
        *   **Enhanced Security & Stability:** Provides critical protection against simple DoS attacks and abusive clients, preserving system resources for legitimate users.
        *   **Resource Efficiency:** Leverages the existing Redis instance, avoiding the overhead of deploying and managing a separate service just for rate limiting.
        *   **Centralized Control:** The rate-limiting logic is managed in one central place (the gateway).
    *   **Cons:**
        *   **Shared Resource Contention:** We are now using the Redis instance for two distinct purposes (caching and rate limiting). Extremely high traffic could theoretically cause contention between these two functions.
        *   **Minor Latency Increase:** Adds a sub-millisecond network hop from the gateway to Redis for every request, which is a negligible but real performance cost.
        *   **Configuration Overhead:** Requires careful planning and tuning of the rate limits to avoid impacting legitimate, high-traffic users.

---

#### **Logical View (C4 Component Diagram)**

The logical view is updated to show a new critical dependency: the `API Gateway` now communicates with the `Cache Service` to enforce rate limits.

```mermaid
C4Container
    title Logical Architecture for ClassifyHub - Rate Limiting

    Person(user, "Authenticated User", "A user of the application.")

    System_Boundary(c1, "ClassifyHub System") {
        Container(api_gateway, "API Gateway", "e.g., Nginx, Kong", "Single entry point. Enforces rate limits by checking counters in the Cache Service.")
        Container(backend_service, "Backend Service", "Handles business logic.")
        Container(cache_service, "Cache Service", "Docker Container (Redis)", "1. Stores frequently accessed post data. \n2. Stores request counters for rate limiting.")
        
        %% --- Other Components for context ---
        Container(frontend_service, "Frontend Service", "Provides the UI.")
        ContainerDb(sql_database, "SQL Database Cluster", "The single source of truth.")
        ContainerDb(object_store, "Object Store", "Stores images.")
        Container(search_service, "Search Service", "Provides search API.")
        Container(cdn, "CDN", "Serves static assets.")
    }

    Rel(user, cdn, "Loads static content")
    Rel(user, api_gateway, "Makes API requests")

    Rel(api_gateway, cache_service, "Checks/Increments Rate Limit Counters", "TCP")
    Rel(api_gateway, backend_service, "Routes API requests (if not rate limited)")
    Rel(api_gateway, search_service, "Routes search queries (if not rate limited)")

    Rel(backend_service, sql_database, "Reads/Writes database")
    Rel(backend_service, cache_service, "Reads/Writes post data from cache")
```

---

#### **Physical View (AWS Deployment Diagram)**

The physical view adds a new connection path from the `API Gateway Container` to the `Cache Service Container`.

```mermaid
graph TD
    %% --- Define Actors ---
    subgraph "Internet"
        user_client["User's Browser"]
    end

    %% --- Define Cloud Infrastructure ---
    subgraph AWS_Cloud ["AWS Cloud"]
        cloudfront["CloudFront (CDN)"]
        s3_bucket["S3 Bucket"]
        sqs_queue["SQS Queue"]

        subgraph VPC ["VPC"]
            subgraph Public_Subnet ["Public Subnet"]
                subgraph ec2_instance ["EC2 Instance (t3.micro)"]
                    docker["Docker Engine"]
                    
                    api_gateway_container["API Gateway"]
                    backend_container["Backend Service"]
                    cache_container["Cache Service (Redis)"]
                    
                    %% Other containers for context
                    db_primary_container["Primary DB"]
                    db_replica_container["Replica DB"]
                    frontend_container["Frontend Service"]
                    notification_worker_container["Notification Worker"]
                    search_container["Search Service"]
                end
            end
        end
    end

    %% --- Define Relationships ---
    %% 1. Edge Traffic (unchanged)
    user_client -- "Loads Assets" --> cloudfront
    user_client -- "API Calls" --> api_gateway_container

    %% 2. NEW: API Gateway Rate Limiting Flow
    api_gateway_container -- "1.Check/Increment Counter" --> cache_container
    api_gateway_container -- "2.Route Request <br/>(if allowed)" --> backend_container & search_container
    
    %% 3. Backend DB & Cache interactions
    backend_container -- "WRITES" --> db_primary_container
    backend_container -- "READS" --> db_replica_container
    backend_container -- "R/W Post Cache" --> cache_container

    %% --- Styling ---
    classDef aws_cloud fill:#FF9900,stroke:#333,stroke-width:2px;
    classDef managed_service fill:#5A97DB,stroke:#333,stroke-width:2px;
    classDef cdn_style fill:#4CAF50,stroke:#333,stroke-width:2px;
    classDef vpc_style fill:#f2f2f2,stroke:#333;
    classDef ec2 fill:#D2F2D2,stroke:#333;
    
    class AWS_Cloud aws_cloud
    class VPC,Public_Subnet vpc_style
    class ec2_instance ec2
    class s3_bucket,sqs_queue managed_service
    class cloudfront cdn_style
```

---

#### **Component-to-Resource Mapping Table**

We update the roles of the `API Gateway` and `Cache Service`.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **API Gateway** | Docker Container on a single EC2 Instance | Role updated to include rate-limiting logic, making it a more robust and secure entry point. |
| **Cache Service** | Docker Container (Redis) on a single EC2 Instance | Role updated to be dual-purpose: application-level caching for the backend and high-speed counting for the API gateway. This is a cost-effective use of an existing resource. |
| **... (Other components)** | ... (No change) | ... |
