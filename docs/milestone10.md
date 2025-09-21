#### **ARCH-10: Scale the Database with Read Replicas**

*   **Problem:** Our single `SQL Database` container is a critical single point of failure and a performance bottleneck. It cannot handle the high volume of read requests required (`NFR-2.2`) while simultaneously processing writes without performance degradation and increased latency.

*   **Solution:** We will evolve the database from a single instance into a **Primary-Replica (Leader-Follower) Cluster**.
    *   A **Primary DB** instance will handle all write operations (INSERT, UPDATE, DELETE).
    *   A new **Replica DB** instance will be configured to asynchronously replicate data from the primary and will handle the majority of the application's read traffic (SELECT queries).
    *   The `Backend Service` will be updated to implement read/write splitting, directing all write queries to the primary and all read queries to the replica. This separates the workloads, allowing us to scale reads independently.

*   **Trade-offs:**
    *   **Pros:**
        *   **Read Scalability:** Dramatically increases our capacity for handling read requests. We can add more replicas in the future as read traffic grows.
        *   **Improved Performance:** Offloading the read workload allows the primary database to dedicate its resources to writes, reducing lock contention and improving overall throughput.
        *   **Increased Availability:** The replica serves as a warm standby. If the primary fails, a replica can be promoted to take its place (manual failover at this stage), reducing downtime.
    *   **Cons:**
        *   **Replication Lag:** Data written to the primary is not instantly available on the replica. This introduces a small window of eventual consistency, which is an acceptable trade-off for this application (e.g., a new post taking a few hundred milliseconds to appear in listings).
        *   **Increased Complexity:** We are now managing a distributed data system. The application's data access logic also becomes more complex to handle the read/write splitting.

---

#### **Logical View (C4 Component Diagram)**

The single `SQL Database` container is now replaced by a `Database Cluster` containing a `Primary` and a `Replica` instance, showing the separated read/write paths.

```mermaid
C4Container
    title Logical Architecture for ClassifyHub - DB Read Replicas

    Person(user, "Authenticated User", "A user of the application.")

    System_Boundary(c1, "ClassifyHub System") {
        Container(api_gateway, "API Gateway", "Single entry point for API traffic.")
        Container(backend_service, "Backend Service", "Handles business logic. Implements Read/Write Splitting for the database.")

        System_Boundary(db_cluster, "SQL Database Cluster") {
            ContainerDb(primary_db, "Primary DB", "PostgreSQL", "Handles all write operations (INSERT, UPDATE, DELETE).")
            ContainerDb(replica_db, "Replica DB", "PostgreSQL", "Handles all read operations (SELECT).")
        }
        
        %% --- Other Components for context ---
        Container(frontend_service, "Frontend Service", "Provides the UI.")
        Container(cdn, "CDN", "Serves static assets.")
        ContainerDb(object_store, "Object Store", "Stores images.")
        Container(search_service, "Search Service", "Provides search API.")
        Container(cache_service, "Cache Service", "Internal data cache.")
    }

    Rel(user, cdn, "Loads static content")
    Rel(user, api_gateway, "Makes API requests")
    Rel(api_gateway, backend_service, "Routes API requests")
    Rel(api_gateway, search_service, "Routes search queries")

    Rel(backend_service, primary_db, "Writes to", "TCP")
    Rel(backend_service, replica_db, "Reads from", "TCP")
    Rel(primary_db, replica_db, "Replicates data to", "Async")
    
    Rel(backend_service, cache_service, "Reads/Writes cache")
```

---

#### **Physical View (AWS Deployment Diagram)**

The physical view now shows two database containers inside the EC2 instance, with the backend container connecting to both.

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
                    db_primary_container["Primary DB Container"]
                    db_replica_container["Replica DB Container"]
                    
                    %% Other containers for context
                    frontend_container["Frontend Service"]
                    notification_worker_container["Notification Worker"]
                    cache_container["Cache Service"]
                    search_container["Search Service"]
                end
            end
        end
    end

    %% --- Define Relationships ---
    %% 1. Edge Traffic (unchanged)
    user_client -- "Loads Assets" --> cloudfront
    user_client -- "API Calls" --> api_gateway_container

    %% 2. Gateway Routing (unchanged)
    api_gateway_container -- "Routes to" --> backend_container & search_container

    %% 3. NEW: Backend DB Read/Write Splitting
    backend_container -- "WRITES" --> db_primary_container
    backend_container -- "READS" --> db_replica_container
    
    %% 4. NEW: DB Replication
    db_primary_container -- "Async Replication" --> db_replica_container
    
    %% Other backend interactions
    backend_container -- "R/W Cache" --> cache_container
    backend_container -- "Update Index" --> search_container
    backend_container -- "Publish Job" --> sqs_queue

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

We update the entry for the `SQL Database` to reflect its new cluster configuration.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **SQL Database Cluster** | Two Docker Containers (PostgreSQL) on the same EC2 Instance | Evolving to a Primary-Replica model is a standard pattern for scaling database reads. For now, co-locating both containers on the same host is a simplification. In a truly HA setup (future issue), these would be on separate hosts. |
| **... (Other components)** | ... (No change) | ... |
