#### **ARCH-8: Implement Asynchronous Notification System**

*   **Problem:** The system needs to send emails for actions like post renewal reminders (`FR-2.1.5`) and poster contact forms (`FR-3.3.1`). Performing these actions synchronously within an API request would make the API slow and unreliable. If a third-party email service is down or slow, the user's request will hang or fail, leading to a poor user experience.

*   **Solution:** We will implement an asynchronous notification system. The `Backend Service` will no longer attempt to send emails directly. Instead, it will publish a message containing the notification details (e.g., recipient, type, content) to a **Message Queue** (like AWS SQS). We will introduce a new, independent **Notification Worker Service**. This service's only job is to poll the queue for new messages, process them, and make the actual API call to a third-party **Email Service** (like AWS SES). This decouples the core API from the unreliable task of sending emails.

*   **Trade-offs:**
    *   **Pros:**
        *   **Improved API Performance:** User-facing API responses are now immediate, as the backend only needs to perform a quick write to the queue.
        *   **Increased Reliability:** If the email service is down, messages are safely stored in the queue and can be processed when the service recovers. This prevents lost notifications.
        *   **Decoupling & Scalability:** The notification logic is now a separate service that can be maintained and scaled independently of the main backend.
    *   **Cons:**
        *   **Eventual Consistency:** The user receives a success response before the email is actually sent. This is perfectly acceptable for notifications.
        *   **Increased Complexity:** Introduces two new architectural components: the message queue and the worker service, which add to the operational overhead.

---

#### **Logical View (C4 Component Diagram)**

The logical view now includes the `Message Queue` and the `Notification Worker`, as well as the external `Email Service`.

```mermaid
C4Container
    title Logical Architecture for ClassifyHub - Async Notifications

    Person(user, "Authenticated User", "A user of the application.")
    System(email_service, "Email Service", "e.g., AWS SES, SendGrid", "External service that delivers emails.")

    System_Boundary(c1, "ClassifyHub System") {
        Container(api_gateway, "API Gateway", "Single entry point for all traffic.")
        Container(frontend_service, "Frontend Service", "Provides the user interface.")
        Container(backend_service, "Backend Service", "Handles business logic. Publishes notification jobs to the queue.")
        Container(notification_worker, "Notification Worker", "Docker Container", "Consumes jobs from the queue and sends emails via the Email Service.")
        Container(search_service, "Search Service", "Provides the full-text search API.")
        Container(cache_service, "Cache Service", "Stores frequently accessed data.")
        ContainerDb(message_queue, "Message Queue", "AWS SQS", "Buffers notification jobs for reliable, asynchronous processing.")
        ContainerDb(sql_database, "SQL Database", "The single source of truth.")
        ContainerDb(object_store, "Object Store", "Stores user-uploaded images.")
    }
    
    Rel(user, api_gateway, "Makes all requests to", "HTTPS")
    Rel(api_gateway, frontend_service, "Serves UI")
    Rel(api_gateway, backend_service, "Routes API requests")
    Rel(api_gateway, search_service, "Routes search queries")

    Rel(backend_service, message_queue, "Publishes notification job to", "HTTPS")
    Rel(notification_worker, message_queue, "Consumes notification job from", "HTTPS")
    Rel(notification_worker, email_service, "Sends email via", "HTTPS")

    Rel_Back(sql_database, backend_service, "Reads/Writes data")
    Rel_Back(cache_service, backend_service, "Reads/Writes data")
    Rel_Back(search_service, backend_service, "Updates index")
    Rel_Back(object_store, backend_service, "Writes files")

    Rel(user, object_store, "Reads images", "HTTPS")
```

---

#### **Physical View (AWS Deployment Diagram)**

The physical view adds the new `Notification Worker` container to the EC2 instance and the `Message Queue` (SQS) as a managed AWS service.

```mermaid
graph TD
    %% --- Define Actors & External Systems ---
    subgraph "Internet"
        user_client["User's Browser"]
    end
    
    email_service["External Email Service <br/> (e.g., AWS SES)"]

    %% --- Define Cloud Infrastructure ---
    subgraph AWS_Cloud ["AWS Cloud"]
        s3_bucket["S3 Bucket"]
        sqs_queue["SQS Queue <br/> (Message Queue)"]

        subgraph VPC ["VPC"]
            subgraph Public_Subnet ["Public Subnet"]
                subgraph ec2_instance ["EC2 Instance (t3.micro)"]
                    docker["Docker Engine"]
                    
                    api_gateway_container["API Gateway"]
                    frontend_container["Frontend Service"]
                    backend_container["Backend Service"]
                    notification_worker_container["Notification Worker"]
                    cache_container["Cache Service"]
                    search_container["Search Service"]
                    db_container["PostgreSQL"]
                end
            end
        end
    end

    %% --- Define Relationships ---
    %% 1. User -> Gateway -> Services
    user_client -- "HTTPS" --> api_gateway_container
    api_gateway_container -- "Routes to" --> frontend_container & backend_container & search_container
    
    %% 2. Async Notification Flow
    backend_container -- "1.Publish Job" --> sqs_queue
    notification_worker_container -- "2.Consume Job" --> sqs_queue
    notification_worker_container -- "3.Send Email" --> email_service

    %% 3. Backend Service internal interactions (defined directly)
    backend_container -- "R/W Cache" --> cache_container
    backend_container -- "R/W DB" --> db_container
    backend_container -- "Update Index" --> search_container
    backend_container -- "Uploads to S3" --> s3_bucket
    
    %% 4. User reads images directly from S3
    user_client -- "Displays Images" --> s3_bucket

    %% --- Styling ---
    classDef aws_cloud fill:#FF9900,stroke:#333,stroke-width:2px;
    classDef managed_service fill:#5A97DB,stroke:#333,stroke-width:2px;
    classDef vpc_style fill:#f2f2f2,stroke:#333;
    classDef ec2 fill:#D2F2D2,stroke:#333;
    
    class AWS_Cloud aws_cloud
    class VPC,Public_Subnet vpc_style
    class ec2_instance ec2
    class s3_bucket,sqs_queue managed_service
```

---

#### **Component-to-Resource Mapping Table**

We add the new components to our mapping table.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **API Gateway** | Docker Container on a single EC2 Instance | No change. |
| **Frontend Service** | Docker Container on a single EC2 Instance | No change. |
| **Backend Service** | Docker Container on a single EC2 Instance | No change in resource. Updated to publish messages instead of sending emails. |
| **Notification Worker Service** | Docker Container on a single EC2 Instance | A new, lightweight service. Co-locating it is simple and cost-effective at this stage. It can be scaled separately later if needed. |
| **Message Queue** | AWS SQS (Simple Queue Service) | A fully managed, highly available, and scalable message queue service that perfectly suits our need to decouple the notification workload reliably. |
| **Cache Service** | Docker Container (Redis) on a single EC2 Instance | No change. |
| **Search Service** | Docker Container (Elasticsearch) on a single EC2 Instance | No change. |
| **SQL Database** | Docker Container (PostgreSQL) on the same EC2 Instance | No change. |
| **Object Store** | AWS S3 Bucket | No change. |
