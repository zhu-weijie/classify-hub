#### **ARCH-5: Implement Dedicated Search Service**

*   **Problem:** The PRD requires users to perform keyword searches on posts (`FR-3.2.1`). Relying on the SQL database for full-text search (e.g., using `LIKE '%query%'` clauses) is inefficient, does not scale, and provides a poor user experience. This approach would quickly become a major performance bottleneck.

*   **Solution:** Introduce a dedicated **Search Service** using a specialized search engine like Elasticsearch. This service will maintain an optimized index of post data (title, description, etc.). When a post is created or updated, the `Backend Service` will perform a "dual write," sending the data to both the `SQL Database` for persistent storage and to the `Search Service` for indexing. All search queries from the `Frontend Service` will be sent directly to the `Search Service`'s API, completely isolating the search workload from the primary database.

*   **Trade-offs:**
    *   **Pros:**
        *   **High Performance:** Elasticsearch is purpose-built for fast, complex full-text search, providing a vastly superior experience.
        *   **Decoupled Workloads:** Separates the read-intensive search traffic from the transactional workload of the main database, allowing each to be scaled and optimized independently.
        *   **Advanced Features:** Enables features like relevance scoring, faceting, and suggestions which are difficult to implement with SQL.
    *   **Cons:**
        *   **Data Consistency Challenge:** Using a "dual write" strategy introduces the risk of data becoming inconsistent if one write succeeds and the other fails. For now, we accept this risk for simplicity, but a more robust system might require a Change Data Capture (CDC) pipeline.
        *   **Increased Operational Complexity:** Adds another complex, stateful service to our architecture that requires deployment, monitoring, and maintenance.

---

#### **Logical View (C4 Component Diagram)**

The logical view now incorporates the `Search Service`, showing how it integrates into both the read (search) and write (indexing) paths.

```mermaid
C4Container
    title Logical Architecture for ClassifyHub - Dedicated Search

    Person(user, "Authenticated User", "A registered user searching for and creating posts.")

    System_Boundary(c1, "ClassifyHub System") {
        Container(frontend_service, "Frontend Service", "Docker Container (React SPA served by Nginx)", "Provides the UI. Sends search queries to the Search Service.")
        Container(backend_service, "Backend Service", "Docker Container (e.g., Node.js/Python)", "Handles business logic. Writes data to SQL and the Search Service.")
        Container(search_service, "Search Service", "Docker Container (Elasticsearch)", "Provides a fast, full-text search API over all post data.")
        ContainerDb(sql_database, "SQL Database", "Docker Container (PostgreSQL)", "The single source of truth for all post and user data.")
        ContainerDb(object_store, "Object Store", "AWS S3", "Stores all user-uploaded images.")
    }

    Rel(user, frontend_service, "Views and interacts with", "HTTPS")
    Rel(frontend_service, backend_service, "Makes API calls for creating/viewing posts", "HTTPS (JWT)")
    Rel(frontend_service, search_service, "Sends search queries to", "HTTPS")

    Rel(backend_service, sql_database, "Writes/Reads post data", "TCP")
    Rel(backend_service, search_service, "Writes/Updates search index", "HTTPS")
    Rel(backend_service, object_store, "Writes image files to", "HTTPS")
    
    Rel(user, object_store, "Reads images from", "HTTPS (via Pre-signed URLs)")
```

---

#### **Physical View (AWS Deployment Diagram)**

We add the new `Search Service` container to our single EC2 instance, keeping the deployment simple.

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
                    
                    frontend_container["Frontend Service Container (Nginx)"]
                    backend_container["Backend Service Container"]
                    search_container["Search Service Container (Elasticsearch)"]
                    db_container["PostgreSQL Container"]
                end
            end
        end
    end

    %% --- Define Relationships ---
    %% 1. User interacts with Frontend.
    user_client -- "HTTPS" --> frontend_container

    %% 2. SPA makes calls to Backend for posts and Search for queries.
    user_client -- "API Calls" --> backend_container
    user_client -- "Search Queries" --> search_container

    %% 3. Backend performs dual writes.
    backend_container -- "Writes to DB" --> db_container
    backend_container -- "Updates Index" --> search_container

    %% 4. Backend handles image uploads.
    backend_container -- "Uploads to S3" --> s3_bucket
    
    %% 5. User's browser renders images from S3.
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

We add the new `Search Service` to our mapping table.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Frontend Service** | Docker Container (Nginx) on a single EC2 Instance | No change. It will be updated to include a search UI and API calls to the new service. |
| **Backend Service** | Docker Container on a single EC2 Instance | No change in resource. It will be updated to perform dual writes to the database and search service. |
| **Search Service** | Docker Container (Elasticsearch) on a single EC2 Instance | Elasticsearch is a powerful, industry-standard search engine. Running it as a container on the same host maintains simplicity and is cost-effective for this stage. |
| **SQL Database** | Docker Container (PostgreSQL) on the same EC2 Instance | No change. Remains the primary source of truth. |
| **Object Store** | AWS S3 Bucket | No change. |
