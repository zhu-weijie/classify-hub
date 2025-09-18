#### **ARCH-14: Implement Geo-Partitioning and Routing**

*   **Problem:** While our system is highly available within a single AWS region, it will provide a high-latency experience for users located on different continents. Furthermore, it cannot meet potential data residency requirements (e.g., GDPR) that mandate storing user data within a specific geographic area.

*   **Solution:** We will evolve the architecture into a fully **multi-region, geo-partitioned deployment**.
    1.  **Replicate the Regional Stack:** The entire highly available, multi-AZ architecture we designed in Issue #12 will be treated as a "regional stamp" and deployed into multiple AWS regions (e.g., North America, Europe, Asia).
    2.  **Introduce GeoDNS:** We will use **Amazon Route 53** with a **geolocation routing policy**. This is a smart DNS service that will detect where a user's request originates from and automatically route them to the AWS region that is geographically closest, ensuring the lowest possible latency.
    3.  **Partition Data:** Each regional deployment will be self-contained and will operate on its own independent dataset. For example, a user routed to the `eu-west-1` (Ireland) region will only see and create posts within that region's database.

*   **Trade-offs:**
    *   **Pros:**
        *   **Optimal Global Performance:** Users are always served from the nearest data center, providing the fastest possible experience.
        *   **Data Residency Compliance:** Natively supports requirements to keep a region's data within that region.
        *   **Ultimate Fault Tolerance:** Provides a disaster recovery solution, as the system can withstand the failure of an entire AWS region.
    *   **Cons:**
        *   **Highest Cost and Complexity:** This is the most complex and expensive architecture to operate, as it involves managing multiple, independent deployments of the entire stack.
        *   **Partitioned User Experience:** By default, users in one region cannot see data from another. Implementing cross-region search or data sharing is a significant undertaking and is considered out of scope for this design.

---

#### **Logical View (C4 Component Diagram)**

The logical view is elevated to show the global scale. A new `GeoDNS` component now sits in front of multiple, identical `ClassifyHub Regional Deployment` systems.

```mermaid
C4Container
    title Logical Architecture for ClassifyHub - Multi-Region Deployment

    Person(user, "Global User", "A user located anywhere in the world.")

    System_Boundary(c1, "Global ClassifyHub System") {
        Container(geo_dns, "GeoDNS", "AWS Route 53", "Detects user location and routes them to the nearest regional deployment.")

        System_Boundary(region1, "ClassifyHub Regional Deployment (e.g., North America)") {
             Container(regional_system_1, "Complete HA Stack", "ALB, ECS, RDS, etc.", "The full, highly available system designed in Issue #12.")
        }
        
        System_Boundary(region2, "ClassifyHub Regional Deployment (e.g., Europe)") {
             Container(regional_system_2, "Complete HA Stack", "ALB, ECS, RDS, etc.", "An independent deployment of the full system.")
        }

        System_Boundary(region3, "ClassifyHub Regional Deployment (e.g., Asia)") {
             Container(regional_system_3, "Complete HA Stack", "ALB, ECS, RDS, etc.", "An independent deployment of the full system.")
        }
    }

    Rel(user, geo_dns, "Resolves classifyhub.com", "DNS")
    
    %% FIX: Relationships now point to the container inside the boundary, not the boundary itself.
    Rel(geo_dns, regional_system_1, "Routes NA users to")
    Rel(geo_dns, regional_system_2, "Routes EU users to")
    Rel(geo_dns, regional_system_3, "Routes AP users to")
```

---

#### **Physical View (AWS Deployment Diagram)**

This is our highest-level physical view, abstracting the details of each regional stack to focus on the global routing pattern.

```mermaid
graph TD
    subgraph "Internet"
        user_na["User in <br/> North America"]
        user_eu["User in <br/> Europe"]
        user_ap["User in <br/> Asia"]
    end

    subgraph AWS_Global ["AWS Global Infrastructure"]
        route53["Route 53 <br/> (GeoDNS)"]
    end
    
    subgraph AWS_Region_NA ["AWS Region: us-east-1"]
        regional_stack_na["[Regional HA Stack] <br/> ALB, ECS, RDS, etc."]
    end

    subgraph AWS_Region_EU ["AWS Region: eu-west-1"]
        regional_stack_eu["[Regional HA Stack] <br/> ALB, ECS, RDS, etc."]
    end
    
    subgraph AWS_Region_AP ["AWS Region: ap-southeast-1"]
        regional_stack_ap["[Regional HA Stack] <br/> ALB, ECS, RDS, etc."]
    end

    %% --- Define Relationships ---
    user_na & user_eu & user_ap -- "DNS Lookup for classifyhub.com" --> route53

    route53 -- "Routes to nearest region" --> user_na & user_eu & user_ap

    user_na -- "Connects to" --> regional_stack_na
    user_eu -- "Connects to" --> regional_stack_eu
    user_ap -- "Connects to" --> regional_stack_ap
    
    %% --- Styling ---
    classDef global fill:#9C27B0,stroke:white,color:white;
    classDef region fill:#FF9900,stroke:#333;

    class AWS_Global global
    class AWS_Region_NA,AWS_Region_EU,AWS_Region_AP region
```

---

#### **Component-to-Resource Mapping Table**

We add our final component and update the system's overall description.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **GeoDNS** | **AWS Route 53** with Geolocation Routing Policy | A highly available and scalable managed DNS service. Its geolocation routing feature is the ideal tool for directing users to the lowest-latency regional endpoint. |
| **ClassifyHub Regional Deployment** | The entire Multi-AZ architecture from Issue #12, deployed as a stamp in multiple AWS Regions. | Replicating the entire stack ensures that each region is independent, resilient, and provides the full set of application features to its local users. |

#### **Overall Logical Architecture Diagram**

```mermaid
C4Container
    title Overall Logical Architecture for ClassifyHub (Final)

    Person(user, "Authenticated User", "A user of the application.")
    Person(engineer, "On-Call Engineer", "Receives alerts about system health.")
    System(email_service, "Email Service", "External service that delivers emails.")

    System_Boundary(c1, "ClassifyHub System") {
    
        %% --- Edge & Entry Points ---
        Container(cdn, "CDN", "AWS CloudFront", "Caches and serves static assets (UI, images) from global edge locations.")
        Container(api_gateway, "API Gateway", "Highly available, load-balanced single entry point for all API traffic. Enforces rate limiting.")

        %% --- Core Application Services ---
        Container(frontend_service, "Frontend Service", "Provides the UI assets (Origin for CDN).")
        Container(backend_service, "Backend Service", "Horizontally scalable service. Handles business logic, auth, and data orchestration.")
        Container(search_service, "Search Service", "Managed, Multi-AZ OpenSearch Cluster. Provides the full-text search API.")
        Container(notification_worker, "Notification Worker", "Processes background jobs asynchronously from the message queue.")

        %% --- Data & Messaging Stores ---
        ContainerDb(sql_database, "SQL Database Cluster", "Managed, Multi-AZ PostgreSQL (RDS). The single source of truth.")
        ContainerDb(cache_service, "Cache Service", "Managed, Multi-AZ Redis (ElastiCache). Stores cached data and rate-limit counters.")
        ContainerDb(object_store, "Object Store", "AWS S3. Stores all user-uploaded images.")
        ContainerDb(message_queue, "Message Queue", "AWS SQS. Buffers notification jobs for reliable, asynchronous processing.")

        %% --- Observability Stack (Cross-Cutting Concern) ---
        System_Boundary(observability_stack, "Observability Stack") {
            Container(logging, "Log Aggregation", "AWS CloudWatch Logs", "Collects logs from all services.")
            Container(metrics, "Metrics & Monitoring", "AWS CloudWatch Metrics", "Collects metrics and provides dashboards.")
            Container(alerting, "Alerting", "AWS CloudWatch Alarms & SNS", "Triggers and sends alerts based on metric thresholds.")
        }
    }

    %% --- Relationships ---

    %% 1. User Interaction Flow
    Rel(user, cdn, "Loads UI and Images", "HTTPS")
    Rel(user, api_gateway, "Makes API/Search Requests", "HTTPS")

    %% 2. Edge & Routing
    Rel(cdn, frontend_service, "Pulls UI Assets (Origin)")
    Rel(cdn, object_store, "Pulls Images (Origin)")
    Rel(api_gateway, backend_service, "Routes API Requests")
    Rel(api_gateway, search_service, "Routes Search Queries")
    Rel(api_gateway, cache_service, "Checks Rate-Limit Counters")

    %% 3. Backend Service Orchestration
    Rel(backend_service, sql_database, "Reads/Writes Data")
    Rel(backend_service, cache_service, "Reads/Writes Post Cache")
    Rel(backend_service, search_service, "Writes to Search Index")
    Rel(backend_service, object_store, "Writes Image Files")
    Rel(backend_service, message_queue, "Publishes Notification Job")

    %% 4. Asynchronous Notification Flow
    Rel(notification_worker, message_queue, "Consumes Job")
    Rel(notification_worker, email_service, "Sends Email")
    
    %% 5. Observability Data Flow (Examples)
    Rel(backend_service, logging, "Sends Logs")
    Rel(backend_service, metrics, "Sends Metrics")
    Rel(sql_database, metrics, "Sends Performance Metrics")
    Rel(metrics, alerting, "Feeds Metrics Into")
    Rel(alerting, engineer, "Sends Alerts To")
```

#### **Overall Physical Architecture Diagram**

```mermaid
graph TD
    %% --- Define Actors & External Systems ---
    subgraph "Internet"
        user_client["User's Browser"]
        on_call_engineer["On-Call Engineer"]
    end
    
    email_service["External Email Service"]

    %% --- Define Cloud Infrastructure ---
    subgraph AWS_Cloud ["AWS Cloud"]
    
        %% 1. Global & Edge Services
        route53["Route 53 (GeoDNS)"]
        cloudfront["CloudFront (CDN)"]

        %% 2. Regional Managed Services (outside VPC for clarity)
        s3_bucket["S3 Bucket"]
        sqs_queue["SQS Queue"]
        subgraph Observability
            direction LR
            cloudwatch["CloudWatch <br/>(Logs, Metrics, Alarms)"]
            sns["SNS Topic"]
        end

        %% 3. The Core VPC for a Single Region
        subgraph VPC ["VPC"]
            
            %% Public Subnets with Load Balancer
            subgraph Public_Subnets ["Public Subnets (Multi-AZ)"]
                alb["Application <br/> Load Balancer"]
            end

            %% Availability Zone 1
            subgraph AZ1 ["Availability Zone 1"]
                subgraph Private_Subnet_1 ["Private Subnet"]
                    ecs_tasks_1["ECS Tasks <br/> (API GW, Backend, Frontend, <br/> Search, Notification Worker)"]
                    rds_primary["RDS Primary"]
                    elasticache_1["ElastiCache Node"]
                    opensearch_1["OpenSearch Node"]
                end
            end
            
            %% Availability Zone 2
            subgraph AZ2 ["Availability Zone 2"]
                subgraph Private_Subnet_2 ["Private Subnet"]
                    ecs_tasks_2["ECS Tasks <br/> (API GW, Backend, Frontend, <br/> Search, Notification Worker)"]
                    rds_standby["RDS Standby <br/> Replica"]
                    elasticache_2["ElastiCache Node"]
                    opensearch_2["OpenSearch Node"]
                end
            end
        end
    end
    
    %% --- Define Relationships ---
    %% 1. Initial User Connection
    user_client -- "1.DNS Lookup" --> route53
    route53 -- "2.Routes to nearest region's ALB/CDN" --> user_client

    %% 2. Traffic Flow to Edge
    user_client -- "3a.Loads Static Assets" --> cloudfront
    user_client -- "3b.API & Search Traffic" --> alb

    %% 3. CDN to Origin
    cloudfront -- "Pulls Images" --> s3_bucket
    cloudfront -- "Pulls UI Assets" --> alb

    %% 4. Load Balancer to Application
    alb -- "Distributes Traffic" --> ecs_tasks_1 & ecs_tasks_2
    
    %% 5. Application to Data Stores
    ecs_tasks_1 & ecs_tasks_2 -- "Writes (Primary)" --> rds_primary
    ecs_tasks_1 & ecs_tasks_2 -- "Reads (Standby can be used for reads)" --> rds_standby
    ecs_tasks_1 & ecs_tasks_2 -- "R/W Cache" --> elasticache_1 & elasticache_2
    ecs_tasks_1 & ecs_tasks_2 -- "R/W Search Index" --> opensearch_1 & opensearch_2
    rds_primary -- "Sync Replication" --> rds_standby
    
    %% 6. Async & Object Storage Flow
    ecs_tasks_1 & ecs_tasks_2 -- "Publishes Job" --> sqs_queue
    ecs_tasks_1 & ecs_tasks_2 -- "Consumes Job & Sends Email" --> sqs_queue & email_service
    ecs_tasks_1 & ecs_tasks_2 -- "Uploads to" --> s3_bucket

    %% 7. Observability Flow
    alb & ecs_tasks_1 & ecs_tasks_2 & rds_primary & rds_standby -- "Sends Logs & Metrics" --> cloudwatch
    cloudwatch -- "Triggers Alarm" --> sns
    sns -- "Sends Notification" --> on_call_engineer

    %% --- Styling ---
    classDef aws_cloud fill:#FF9900,stroke:#333;
    classDef global fill:#9C27B0,stroke:white,color:white;
    classDef cdn_style fill:#4CAF50,stroke:#333;
    classDef vpc_style fill:#f2f2f2,stroke:#333;
    classDef az fill:#DDEBF8,stroke:#333;
    classDef managed_service fill:#5A97DB,stroke:#333;
    classDef obs fill:#9C27B0,stroke:white,color:white;
    
    class AWS_Cloud aws_cloud
    class VPC,Public_Subnets,Private_Subnet_1,Private_Subnet_2 vpc_style
    class AZ1,AZ2 az
    class s3_bucket,sqs_queue,rds_primary,rds_standby,elasticache_1,elasticache_2,opensearch_1,opensearch_2 managed_service
    class cloudfront cdn_style
    class route53 global
    class cloudwatch,sns obs
```
