#### **ARCH-6: Add a Distributed Caching Layer**

*   **Problem:** The system currently relies on the `SQL Database` for all non-search data retrieval. As traffic increases, especially for popular posts, this will lead to high read load, increased latency, and will prevent us from meeting the P99 latency requirement of under 1 second (`NFR-1.1`). The database will become the primary system bottleneck.

*   **Solution:** Introduce a new, dedicated **Cache Service** using a distributed in-memory datastore like Redis. The `Backend Service` will implement a "cache-aside" pattern. When a request for a post is received, the backend will first check the cache.
    *   **Cache Hit:** If the post is in the cache, it is returned directly to the user for a very fast response.
    *   **Cache Miss:** If the post is not in the cache, the backend will query the `SQL Database`, store the result in the cache with a Time-To-Live (TTL), and then return the data. Subsequent requests for that same post will now be cache hits.

*   **Trade-offs:**
    *   **Pros:**
        *   **Massively Improved Latency:** Serving data from memory (Redis) is significantly faster than from a disk-based database, directly addressing our P99 goal.
        *   **Reduced Database Load:** The cache will absorb a large portion of the read traffic, protecting the database and improving its performance on essential write operations.
        *   **Increased Scalability:** Provides a major boost to the read scalability of the entire system.
    *   **Cons:**
        *   **Data Staleness:** The data in the cache can be out-of-date if the source data in the SQL database changes. Using a short TTL (e.g., 5-10 minutes) is a simple initial strategy to mitigate this, but it's not perfect. A more complex cache invalidation strategy (e.g., on write) would be needed for real-time consistency.
        *   **Increased Complexity:** Introduces another stateful service into our architecture that requires deployment, monitoring, and maintenance.

---

#### **Logical View (C4 Component Diagram)**

The logical view now includes the `Cache Service`, showing how it sits between the `Backend Service` and the `SQL Database` for read operations.

```mermaid
C4Container
    title Logical Architecture for ClassifyHub - Caching Layer

    Person(user, "Authenticated User", "A user browsing the application.")

    System_Boundary(c1, "ClassifyHub System") {
        Container(frontend_service, "Frontend Service", "Docker Container (React SPA served by Nginx)", "Provides the UI.")
        Container(backend_service, "Backend Service", "Docker Container (e.g., Node.js/Python)", "Handles business logic. Reads from cache first, then database.")
        Container(cache_service, "Cache Service", "Docker Container (Redis)", "Stores frequently accessed post data in-memory for fast retrieval.")
        Container(search_service, "Search Service", "Docker Container (Elasticsearch)", "Provides the full-text search API.")
        ContainerDb(sql_database, "SQL Database", "Docker Container (PostgreSQL)", "The single source of truth for all data.")
        ContainerDb(object_store, "Object Store", "AWS S3", "Stores all user-uploaded images.")
    }
    
    Rel(user, frontend_service, "Interacts with", "HTTPS")
    Rel(frontend_service, backend_service, "Makes API calls for posts", "HTTPS (JWT)")
    Rel(frontend_service, search_service, "Sends search queries to", "HTTPS")

    Rel(backend_service, cache_service, "Reads/Writes post data", "TCP")
    Rel(backend_service, sql_database, "Reads/Writes data (on cache miss)", "TCP")
    Rel(backend_service, search_service, "Writes/Updates search index", "HTTPS")
    Rel(backend_service, object_store, "Writes image files", "HTTPS")

    Rel(user, object_store, "Reads images", "HTTPS")
```

---

#### **Physical View (AWS Deployment Diagram)**

We add the new `Cache Service` container to our single EC2 instance.

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
                    
                    frontend_container["Frontend Service Container"]
                    backend_container["Backend Service Container"]
                    cache_container["Cache Service Container (Redis)"]
                    search_container["Search Service Container"]
                    db_container["PostgreSQL Container"]
                end
            end
        end
    end

    %% --- Define Relationships ---
    %% User Interactions
    user_client -- "HTTPS" --> frontend_container
    user_client -- "API Calls" --> backend_container
    user_client -- "Search Queries" --> search_container
    user_client -- "Displays Images" --> s3_bucket

    %% Backend Interactions
    backend_container -- "1.Check Cache" --> cache_container
    backend_container -- "2.Read/Write DB <br/> (on cache miss)" --> db_container
    backend_container -- "3.Write to Cache" --> cache_container
    backend_container -- "Updates Index" --> search_container
    backend_container -- "Uploads to S3" --> s3_bucket
    
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

We add the new `Cache Service` to our mapping table.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Frontend Service** | Docker Container (Nginx) on a single EC2 Instance | No change. |
| **Backend Service** | Docker Container on a single EC2 Instance | No change in resource. It will be updated to include a Redis client and the cache-aside logic. |
| **Cache Service** | Docker Container (Redis) on a single EC2 Instance | Redis is an industry-standard, high-performance in-memory cache. Running it as a container on the same host is the simplest deployment for this stage. |
| **Search Service** | Docker Container (Elasticsearch) on a single EC2 Instance | No change. |
| **SQL Database** | Docker Container (PostgreSQL) on the same EC2 Instance | No change. |
| **Object Store** | AWS S3 Bucket | No change. |
