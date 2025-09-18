#### **ARCH-4: Add Object Storage for Image Uploads**

*   **Problem:** The current architecture only supports text-based data, but the PRD (`FR-2.2.1`) requires users to upload images with their posts. Storing binary files (images) directly in the SQL database is highly inefficient, does not scale, and would dramatically increase storage costs and database load.

*   **Solution:** We will introduce a dedicated, managed **Object Storage** service (AWS S3) to handle image persistence. The `Backend Service` will be responsible for processing image uploads from the client; it will securely upload the file to a private S3 bucket and then store only the resulting object URL or key in the `SQL Database` alongside the post data. When a user views a post, the `Frontend Service` will receive the image URL from the `Backend Service` and render the image by sourcing it directly from S3 (using a pre-signed URL for security).

*   **Trade-offs:**
    *   **Pros:**
        *   **Right Tool for the Job:** S3 is highly durable, scalable, and cost-effective for storing and serving large files, offloading this responsibility from our application server and database.
        *   **Improved Performance:** Decouples image storage from the main application, allowing both to be optimized and scaled independently.
        *   **Enhanced Security:** By handling uploads via the backend, the S3 bucket can remain private and inaccessible to the public internet, with the backend granting temporary access via pre-signed URLs.
    *   **Cons:**
        *   **Increased Complexity:** Introduces a new external cloud service (S3) and a new data flow that must be managed and secured.
        *   **Backend Upload Bottleneck:** The chosen server-side upload flow adds load (CPU, memory, bandwidth) to the backend service. This is a conscious trade-off for better security, but it's a potential bottleneck we must address in later scaling stages.

---

#### **Logical View (C4 Component Diagram)**

The logical view now includes the Object Store. The `Backend Service` acts as the writer, while the `User` (via their browser) is the ultimate reader of the image data.

```mermaid
C4Container
    title Logical Architecture for ClassifyHub - Image Storage

    Person(user, "Authenticated User", "A registered user interacting with the UI.")

    System_Boundary(c1, "ClassifyHub System") {
        Container(frontend_service, "Frontend Service", "Docker Container (React SPA served by Nginx)", "Provides the UI. Handles image uploads from the user and displays images fetched from the Object Store.")
        Container(backend_service, "Backend Service", "Docker Container (e.g., Node.js/Python)", "Handles business logic, auth, and processes image uploads to the Object Store.")
        ContainerDb(sql_database, "SQL Database", "Docker Container (PostgreSQL)", "Stores post metadata and image URLs.")
        ContainerDb(object_store, "Object Store", "AWS S3", "Provides highly durable storage for all user-uploaded images.")
    }

    Rel(user, frontend_service, "Views and interacts with", "HTTPS")
    Rel(frontend_service, backend_service, "Makes API calls and sends image data to", "HTTPS (JWT)")
    Rel(backend_service, sql_database, "Writes/Reads image URLs", "TCP")
    Rel(backend_service, object_store, "Writes image files to", "HTTPS")
    Rel(user, object_store, "Reads/Displays images from", "HTTPS (via Pre-signed URLs)")

```

---

#### **Physical View (AWS Deployment Diagram)**

The physical view is updated to include the AWS S3 service, which exists outside our EC2 instance but within the AWS Cloud.

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
                    db_container["PostgreSQL Container"]
                end
            end
        end
    end

    %% --- Define Relationships ---
    %% 1. User loads the SPA from the Frontend.
    user_client -- "HTTPS (Port 443)" --> frontend_container

    %% 2. The SPA sends API requests (including image uploads) to the Backend.
    user_client -- "HTTPS API Calls" --> backend_container

    %% 3. The Backend uploads the received image file to S3.
    backend_container -- "Uploads images <br/> (AWS SDK)" --> s3_bucket

    %% 4. The Backend writes the image URL/key to the Database.
    backend_container -- "Reads/Writes via TCP <br/>(Docker Private Network)" --> db_container
    
    %% 5. The SPA receives the image URL and the browser renders the image directly from S3.
    user_client -- "Displays images via HTTPS <br/> (Pre-signed URL)" --> s3_bucket

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

We add the new `Object Store` to our mapping table.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Frontend Service** | Docker Container (Nginx) on a single EC2 Instance | No change. It will be updated to handle file upload forms. |
| **Backend Service** | Docker Container on a single EC2 Instance | No change in resource. It will be updated to include the AWS SDK and logic for handling S3 uploads. |
| **SQL Database** | Docker Container (PostgreSQL) on the same EC2 Instance | No change. Its schema will be updated to include a column for image URLs/keys. |
| **Object Store** | AWS S3 Bucket | A highly durable, scalable, and cost-effective managed service ideal for storing and serving large binary files like images, offloading this burden from the application server. |
