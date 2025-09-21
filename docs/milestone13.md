#### **ARCH-13: Implement a Centralized Observability Stack**

*   **Problem:** Our system is now a complex, distributed application. When issues occur, it's nearly impossible to trace problems across multiple services, containers, and managed resources. We have no centralized way to view logs, monitor performance metrics, or receive alerts, failing to meet our maintainability and operational requirements (`NFR-5.1`, `5.2`, `5.3`).

*   **Solution:** We will introduce a dedicated, managed **Observability Stack** using AWS services. This stack will centralize the three pillars of observability:
    1.  **Logging:** All services (ECS tasks, Lambda functions, etc.) will be configured to stream their logs to **AWS CloudWatch Logs**. This provides a central place to search and analyze logs from the entire system.
    2.  **Metrics:** All AWS resources (ALB, ECS, RDS, ElastiCache, etc.) will automatically publish performance metrics to **AWS CloudWatch Metrics**. We will create consolidated dashboards in CloudWatch to visualize system health.
    3.  **Alerting:** We will create **AWS CloudWatch Alarms** that trigger on specific metric thresholds (e.g., API latency > 1s, 5xx error rate > 1%, RDS CPU > 80%). These alarms will publish a message to an **AWS Simple Notification Service (SNS)** topic, which can then notify the on-call team via email, SMS, or other channels.

*   **Trade-offs:**
    *   **Pros:**
        *   **Deep System Insight:** Provides the essential tools to monitor, debug, and understand the behavior of our distributed system in real-time.
        *   **Faster Incident Response:** Centralized data and automated alerts drastically reduce the Mean Time To Detect (MTTD) and Mean Time To Resolve (MTTR) for production issues.
        *   **Managed Service Benefits:** Using CloudWatch and SNS is highly reliable and removes the significant operational burden of building, scaling, and maintaining our own complex observability infrastructure.
    *   **Cons:**
        *   **Cost:** CloudWatch pricing is based on data ingestion, storage, and queries. This is a necessary but additional operational cost.
        *   **Vendor Lock-in:** Deeply integrating with a specific cloud provider's observability suite can make future migrations to other clouds more complex.

---

#### **Logical View (C4 Component Diagram)**

The logical view introduces the `Observability Stack` as a cross-cutting concern that receives data from all other system components and sends alerts to an `On-Call Engineer`.

```mermaid
C4Container
    title Logical Architecture for ClassifyHub - Observability

    Person(user, "Authenticated User", "A user of the application.")
    Person(engineer, "On-Call Engineer", "Receives alerts about system health.")

    System_Boundary(c1, "ClassifyHub System") {
        Container(api_gateway, "API Gateway", "Load-balanced entry point.")
        Container(backend_service, "Backend Service", "Horizontally scalable service.")
        
        System_Boundary(observability_stack, "Observability Stack") {
            Container(logging, "Log Aggregation", "AWS CloudWatch Logs", "Collects logs from all services.")
            Container(metrics, "Metrics & Monitoring", "AWS CloudWatch Metrics", "Collects metrics from all services. Provides dashboards.")
            Container(alerting, "Alerting", "AWS CloudWatch Alarms & SNS", "Triggers and sends alerts based on metric thresholds.")
        }
        
        %% --- Other components for context ---
        ContainerDb(sql_database, "SQL Database Cluster", "Managed, Multi-AZ Cluster (RDS).")
    }

    Rel(user, api_gateway, "Makes API requests")
    Rel(api_gateway, backend_service, "Routes requests")
    Rel(backend_service, sql_database, "Reads/Writes data")

    Rel(api_gateway, logging, "Sends Access Logs")
    Rel(api_gateway, metrics, "Sends Performance Metrics")
    
    Rel(backend_service, logging, "Sends Application Logs")
    Rel(backend_service, metrics, "Sends Application Metrics")
    
    Rel(sql_database, logging, "Sends Database Logs")
    Rel(sql_database, metrics, "Sends Performance Metrics")
    
    Rel(metrics, alerting, "Feeds metrics into")
    Rel(alerting, engineer, "Sends alerts to")
```

---

#### **Physical View (AWS Deployment Diagram)**

The physical view shows how the managed observability services sit alongside our VPC and collect data from all the resources within it.

```mermaid
graph TD
    subgraph "Internet"
        user_client["User's Browser"]
        on_call_engineer["On-Call Engineer"]
    end

    subgraph AWS_Cloud ["AWS Cloud"]
        cloudfront["CloudFront (CDN)"]
        s3_bucket["S3 Bucket"]
        
        %% Observability Services
        subgraph Observability
            direction LR
            cloudwatch["CloudWatch <br/>(Logs, Metrics, Alarms)"]
            sns["SNS Topic"]
        end

        subgraph VPC ["VPC"]
            subgraph Public_Subnets
                alb["Application Load Balancer"]
            end

            subgraph AZ1 ["Availability Zone 1"]
                subgraph Private_Subnet_1 ["Private Subnet"]
                    ecs_tasks_1["ECS Tasks"]
                    rds_primary["RDS Primary"]
                end
            end
            
            subgraph AZ2 ["Availability Zone 2"]
                subgraph Private_Subnet_2 ["Private Subnet"]
                    ecs_tasks_2["ECS Tasks"]
                    rds_replica["RDS Standby Replica"]
                end
            end
        end
    end
    
    %% --- Define Relationships ---
    user_client -- "API Traffic" --> alb
    
    %% All infrastructure sends data TO CloudWatch
    alb --> cloudwatch
    ecs_tasks_1 --> cloudwatch
    ecs_tasks_2 --> cloudwatch
    rds_primary --> cloudwatch
    rds_replica --> cloudwatch

    %% Alerting Flow
    cloudwatch -- "1. Triggers Alarm" --> sns
    sns -- "2. Sends Notification" --> on_call_engineer

    %% --- Styling ---
    classDef aws_cloud fill:#FF9900,stroke:#333;
    classDef vpc_style fill:#f2f2f2,stroke:#333;
    classDef az fill:#DDEBF8,stroke:#333;
    classDef obs fill:#9C27B0,stroke:white,color:white;
    
    class AWS_Cloud aws_cloud
    class VPC,Public_Subnets,Private_Subnet_1,Private_Subnet_2 vpc_style
    class AZ1,AZ2 az
    class cloudwatch,sns obs
```

---

#### **Component-to-Resource Mapping Table**

We add the new `Observability Stack` to our mapping table.

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Observability Stack** | **AWS CloudWatch** (Logs, Metrics, Alarms) and **AWS SNS** | A suite of tightly integrated, managed services that provide a comprehensive observability solution out-of-the-box. This is the most efficient and reliable way to gain deep insight into our AWS-based distributed system. |
| **... (Other components)** | ... (No change in resources, but now configured to export logs/metrics) | ... |
