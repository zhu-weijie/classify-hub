#### **ARCH-3: Decouple Frontend Application**

*   **Problem:** Currently, there is no defined user interface. The `Backend Service` is a standalone API, which is insufficient for a user to interact with the application as per the PRD (`FR-3.1.1`). We need a dedicated frontend to provide the user experience.

*   **Solution:** Introduce a new container: a **Frontend Service**. This will be a Single-Page Application (SPA), for example, built with React and served by a lightweight web server like Nginx. This service will be responsible for all UI rendering. It will run as a separate container but, for now, will be co-located on the same EC2 instance as the other services. The `User` will now interact with the `Frontend Service`, which in turn makes API calls to the `Backend Service`.

*   **Trade-offs:**
    *   **Pros:**
        *   **Separation of Concerns:** Frontend and backend development can happen independently, allowing for specialized teams and faster iteration.
        *   **Improved User Experience:** An SPA provides a modern, responsive user experience compared to server-side rendered pages.
        *   **Clear API Contract:** Forces a well-defined API between the client and server.
    *   **Cons:**
        *   **Increased Complexity:** We now have two services to build, deploy, and manage instead of one.
        *   **Co-located Deployment:** While logically separate, they still share the same underlying hardware in this design, meaning they are not physically isolated and a resource contention issue could affect both.

---

#### **Logical View (C4 Component Diagram)**

This diagram introduces the new `Frontend Service` as the primary point of interaction for the user.

```mermaid
C4Container
    title Logical Architecture for ClassifyHub - Decoupled Frontend

    Person(user, "Authenticated User", "A registered user interacting with the UI.")

    System_Boundary(c1, "ClassifyHub System") {
        Container(frontend_service, "Frontend Service", "Docker Container (React SPA served by Nginx)", "Provides the user interface for the ClassifyHub application. All user interactions happen here.")
        Container(backend_service, "Backend Service", "Docker Container (e.g., Node.js/Python)", "Handles business logic and user authentication. Exposes the core REST API.")
        ContainerDb(sql_database, "SQL Database", "Docker Container (PostgreSQL)", "Stores all user and post data.")
    }

    Rel(user, frontend_service, "Views and interacts with", "HTTPS")
    Rel(frontend_service, backend_service, "Makes API calls to", "HTTPS (JWT)")
    Rel(backend_service, sql_database, "Reads/Writes data to", "TCP")
```

---

#### **Physical View (AWS Deployment Diagram)**

The physical view evolves to include the new `Frontend Service` container running on the same EC2 instance. The user's browser now connects to the frontend container, which then communicates with the backend container.

```mermaid
graph TD
    %% --- Define Actors ---
    subgraph "Internet"
        user_client["User's Browser"]
    end

    %% --- Define Cloud Infrastructure ---
    subgraph "AWS Cloud"
        subgraph "VPC"
            subgraph "Public Subnet"
                subgraph ec2_instance ["EC2 Instance (t3.micro)"]
                    docker["Docker Engine"]
                    
                    %% All three containers are co-located on the same host
                    frontend_container["Frontend Service Container (Nginx)"]
                    backend_container["Backend Service Container"]
                    db_container["PostgreSQL Container"]
                end
            end
        end
    end

    %% --- Define Relationships ---
    %% 1. User's browser loads the SPA from the Frontend container.
    user_client -- "HTTPS (Port 443)" --> frontend_container

    %% 2. The SPA (in the browser) makes API calls to the Backend container.
    %% Note: This implies the EC2 instance forwards traffic from another port (e.g., 8080) to the backend.
    user_client -- "HTTPS API Calls (e.g., Port 8080)" --> backend_container

    %% 3. The Backend communicates with the Database.
    backend_container -- "Reads/Writes via TCP <br/>(Docker Private Network)" --> db_container

    %% --- Styling ---
    classDef aws_cloud fill:#FF9900,stroke:#333,stroke-width:2px;
    classDef vpc fill:#f2f2f2,stroke:#333;
    classDef ec2 fill:#D2F2D2,stroke:#333;

    class AWS_Cloud aws_cloud
    class VPC vpc
    class ec2_instance ec2
```

---

#### **Component-to-Resource Mapping Table**

We add the new `Frontend Service` to our mapping table.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Frontend Service** | Docker Container (Nginx) on a single EC2 Instance | A lightweight Nginx container is efficient for serving static SPA assets. Co-locating it on the same EC2 instance is cost-effective and maintains simplicity for this stage. |
| **Backend Service** | Docker Container on a single EC2 Instance | No change in resource. It now serves an internal API role for the frontend. |
| **SQL Database** | Docker Container (PostgreSQL) on the same EC2 Instance | No change. It remains the single source of truth for data. |
