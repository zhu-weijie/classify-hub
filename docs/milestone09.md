#### **ARCH-9: Integrate a Content Delivery Network (CDN)**

*   **Problem:** Our application's static assets (JavaScript, CSS from the `Frontend Service`) and user-uploaded images (from the `Object Store`) are served from a single geographic region. This will result in high latency and a poor user experience for users located far from our server's location.

*   **Solution:** We will introduce a **Content Delivery Network (CDN)**, specifically AWS CloudFront. The CDN will cache our static content at edge locations around the world, physically closer to our users.
    1.  A CloudFront distribution will be configured with the **API Gateway** as its origin for serving the frontend application's static assets.
    2.  A second CloudFront "behavior" will be configured with the **S3 Bucket** as its origin for serving user-uploaded images.
    End users will now download these assets from a nearby CDN edge location, dramatically reducing latency.

*   **Trade-offs:**
    *   **Pros:**
        *   **Drastically Reduced Latency:** Users worldwide will experience significantly faster load times for the UI and images.
        *   **Reduced Origin Load:** The CDN absorbs the vast majority of traffic for static assets, reducing the load and data transfer costs from our EC2 instance and S3 bucket.
        *   **Enhanced Reliability & Security:** Provides an additional layer of caching and DDoS protection.
    *   **Cons:**
        *   **Cache Invalidation Complexity:** When we deploy a new version of the frontend, we must invalidate the CDN cache to ensure users see the latest version. This adds a step to our deployment process.
        *   **Increased Cost:** While often offset by reduced data transfer costs from the origin, a CDN is an additional service with its own pricing.

---

#### **Logical View (C4 Component Diagram)**

The logical view now shows the `CDN` as the primary point of contact for the user for all static content. API calls still go through the `API Gateway`.

```mermaid
C4Container
    title Logical Architecture for ClassifyHub - CDN

    Person(user, "Authenticated User", "A user of the application.")

    System_Boundary(c1, "ClassifyHub System") {
        Container(cdn, "CDN", "AWS CloudFront", "Caches and serves static assets (UI, images) from global edge locations.")
        Container(api_gateway, "API Gateway", "Single entry point for dynamic API traffic.")
        Container(frontend_service, "Frontend Service", "Origin for the CDN. Provides the UI assets.")
        Container(backend_service, "Backend Service", "Handles business logic.")
        Container(search_service, "Search Service", "Provides the search API.")
        ContainerDb(object_store, "Object Store", "AWS S3", "Origin for the CDN. Stores user-uploaded images.")
        
        %% --- Other Internal Components ---
        Container(notification_worker, "Notification Worker", "Processes background jobs.")
        Container(cache_service, "Cache Service", "Internal data cache.")
        ContainerDb(message_queue, "Message Queue", "Buffers notification jobs.")
        ContainerDb(sql_database, "SQL Database", "The single source of truth.")
    }

    System(email_service, "Email Service", "External service that delivers emails.")

    Rel(user, cdn, "Loads UI and images from", "HTTPS")
    Rel(user, api_gateway, "Makes API/Search requests to", "HTTPS")

    Rel(cdn, frontend_service, "Pulls/caches static assets from")
    Rel(cdn, object_store, "Pulls/caches images from")
    
    Rel(api_gateway, backend_service, "Routes API requests")
    Rel(api_gateway, search_service, "Routes search queries")

    Rel(backend_service, message_queue, "Publishes notification jobs")
    Rel(notification_worker, message_queue, "Consumes jobs")
    Rel(notification_worker, email_service, "Sends emails")

    Rel_Back(sql_database, backend_service, "R/W")
    Rel_Back(cache_service, backend_service, "R/W")
    Rel_Back(search_service, backend_service, "Updates index")
    Rel_Back(object_store, backend_service, "Writes files")
```

---

#### **Physical View (AWS Deployment Diagram)**

The physical view now clearly separates traffic. Static content requests go to CloudFront, while dynamic API requests go to the API Gateway.

```mermaid
graph TD
    %% --- Define Actors & External Systems ---
    subgraph "Internet"
        user_client["User's Browser"]
    end
    
    email_service["External Email Service"]

    %% --- Define Cloud Infrastructure ---
    subgraph AWS_Cloud ["AWS Cloud"]
        cloudfront["CloudFront Distribution (CDN)"]
        s3_bucket["S3 Bucket"]
        sqs_queue["SQS Queue"]

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
    %% 1. User traffic is now split
    user_client -- "Loads UI & Images (HTTPS)" --> cloudfront
    user_client -- "API & Search Calls (HTTPS)" --> api_gateway_container

    %% 2. CloudFront origins
    cloudfront -- "Origin 1: Pulls UI assets" --> api_gateway_container
    cloudfront -- "Origin 2: Pulls images" --> s3_bucket

    %% 3. API Gateway routing (unchanged)
    api_gateway_container -- "Routes to" --> frontend_container & backend_container & search_container
    
    %% 4. Async Notification Flow (unchanged)
    backend_container -- "Publish" --> sqs_queue
    notification_worker_container -- "Consume" --> sqs_queue
    notification_worker_container -- "Send" --> email_service

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

We add the new `CDN` component to our mapping table.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **CDN** | AWS CloudFront | A global, managed CDN that caches content close to users. This is the most effective way to reduce latency for static assets (UI files, images) for a worldwide audience. |
| **API Gateway** | Docker Container on a single EC2 Instance | No change. Now handles only dynamic API traffic. |
| **Frontend Service** | Docker Container on a single EC2 Instance | No change in resource, but is now an "origin" for the CDN. |
| **Object Store** | AWS S3 Bucket | No change in resource, but is now an "origin" for the CDN. |
| **... (Other components)** | ... (No change) | ... |
