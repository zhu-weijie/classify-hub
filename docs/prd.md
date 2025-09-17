### **Project Requirement Document: Classifieds Web Application**

**Version:** v1.0.0
**Date:** 17 September 2025

#### **1. Introduction**

This document specifies the functional and non-functional requirements for the development of a classifieds web application. The system is designed to connect two primary user types: **Posters**, who create listings, and **Viewers**, who browse and respond to those listings. The requirements below serve as the formal agreement on the system's scope and behavior for the initial production release.

---

### **2. Functional Requirements (FR)**

#### **2.1 Core User Management**

| ID | Requirement | Description |
| :--- | :--- | :--- |
| **FR-1.1.1** | User Registration | Users must be able to register for an account using a unique email address and a password. A verification email will be sent to confirm the user's email address. |
| **FR-1.1.2** | User Login | Registered users must be able to log in using their email and password. The system must include a "Forgot Password" functionality. |
| **FR-1.1.3** | User Logout | Authenticated users must have a clear option to log out of their session. |
| **FR-1.1.4** | Account Deletion | Users must be able to permanently delete their own account. Upon deletion, all associated personal data and active posts will be removed. |

#### **2.2 Poster Functionality**

| ID | Requirement | Description |
| :--- | :--- | :--- |
| **FR-2.1.1** | Create Post | An authenticated user (Poster) must be able to create a new post with the following mandatory fields: Title, Description, Price, Location (City, Neighborhood), and Category. |
| **FR-2.1.2** | Edit Post | A Poster must be able to edit the content (all fields and images) of their own active posts. |
| **FR-2.1.3** | Delete Post | A Poster must be able to delete their own posts. This will be a soft delete, marking the post as inactive and removing it from public view. |
| **FR-2.1.4** | Manage Own Posts | A Poster must have a dashboard to view and search a list of all their posts (both active and expired). |
| **FR-2.1.5** | Post Renewal | Posts expire automatically after 30 days. A Poster will receive an email notification 48 hours before expiration with a link to renew the post for another 30 days. |
| **FR-2.2.1** | Image Uploads | A Poster can upload up to 10 images per post. Each image must be in `.jpg` or `.png` format and not exceed 1 MB in size. The system will generate thumbnails for list views. |
| **FR-2.2.2** | Post Content Fields | The "Create Post" form will contain the following fields: <br> • **Title:** Text, 5-100 characters. <br> • **Description:** Text, max 4000 characters. <br> • **Price:** Numeric value. A checkbox for "Free" or "Contact for Price" must be available. <br> • **Item Condition:** A selectable value from a predefined list (e.g., New, Excellent, Good, Acceptable). |

#### **2.3 Viewer Functionality**

| ID | Requirement | Description |
| :--- | :--- | :--- |
| **FR-3.1.1** | View Posts | A Viewer can browse all active posts within a selected city. The default view is sorted with the most recently created posts first. |
| **FR-3.1.2** | Post Detail View | Clicking a post from the list view will navigate the Viewer to a detailed page showing all post information and images in a gallery view. |
| **FR-3.1.3** | Geolocation Routing | The system will attempt to auto-detect the user's city based on their IP address and show relevant posts. A Viewer must be able to manually change their selected city. |
| **FR-3.1.4** | Pagination | Search results and category listings will be paginated, displaying a limited number of results per page with clear navigation to subsequent pages (or a "Load More" button). |
| **FR-3.2.1** | Search Posts | A Viewer must be able to perform a keyword search on active posts within the selected city. The search will query the post's Title and Description fields. |
| **FR-3.2.2** | Filter Results | Viewers must be able to refine search results using the following filters: <br> • **Price:** A minimum and maximum value. <br> • **Item Condition:** Multi-select options. <br> • **Neighborhood:** Multi-select options. <br> • **Category:** Selectable from a predefined list. |
| **FR-3.2.3** | Sort Results | Viewers must be able to sort the results list by: <br> • Newest First (Default) <br> • Price: Low to High <br> • Price: High to Low |
| **FR-3.3.1** | Contact Poster | A Viewer can contact a Poster via a form on the post detail page. The system will use an anonymized email relay to forward the message, hiding both parties' email addresses. |
| **FR-3.3.2** | Report Post | A Viewer can report a post for inappropriate content. Report reasons will include: "Spam," "Prohibited Item," and "Misleading." If a post receives a set number of reports, it will be automatically hidden and flagged for administrative review. |

---

### **3. Non-Functional Requirements (NFR)**

#### **3.1 Performance**

| ID | Requirement | Description |
| :--- | :--- | :--- |
| **NFR-1.1** | Read Latency | P99 latency for all read operations (viewing a post, searching, filtering) must be under 1 second. |
| **NFR-1.2** | Write Latency | P99 latency for core write operations (user registration, post creation) must be under 3 seconds. |
| **NFR-1.3** | Time-to-Visibility | A newly created or edited post must be visible in search results and listings within 10 seconds of submission. |

#### **3.2 Scalability**

| ID | Requirement | Description |
| :--- | :--- | :--- |
| **NFR-2.1** | User Load | The architecture must support up to 10 million users within a single large metropolitan area (e.g., city partition). |
| **NFR-2.2** | Concurrent Connections| The system must handle 100,000 concurrent read-heavy users per major city partition during peak hours. |
| **NFR-2.3** | Write Throughput| The system must be able to process at least 10,000 new posts per hour during peak times in a major city partition without performance degradation. |

#### **3.3 Availability**

| ID | Requirement | Description |
| :--- | :--- | :--- |
| **NFR-3.1** | Uptime | The system must achieve 99.9% uptime, excluding scheduled maintenance windows announced 24 hours in advance. |
| **NFR-3.2** | Disaster Recovery | The system must be backed up regularly. In the event of a catastrophic failure, the Recovery Time Objective (RTO) is 4 hours, and the Recovery Point Objective (RPO) is 1 hour. |

#### **3.4 Security**

| ID | Requirement | Description |
| :--- | :--- | :--- |
| **NFR-4.1** | Data Encryption | All data in transit between the client and server must be encrypted using current TLS/SSL standards. Sensitive data at rest (e.g., passwords) must be hashed and salted. |
| **NFR-4.2** | Authentication | The system must enforce strong password policies. All endpoints for creating or modifying data must be protected and accessible only to authenticated and authorized users. |
| **NFR-4.3** | Vulnerability Protection | The application must be protected against the OWASP Top 10 web vulnerabilities, including SQL Injection, Cross-Site Scripting (XSS), and Cross-Site Request Forgery (CSRF). |
| **NFR-4.4** | Rate Limiting | API endpoints must be rate-limited to prevent denial-of-service (DoS) attacks and other forms of abuse from a single source. |

#### **3.5 Maintainability & Observability**

| ID | Requirement | Description |
| :--- | :--- | :--- |
| **NFR-5.1** | Logging | All services must produce structured, centralized logs to facilitate debugging and auditing. |
| **NFR-5.2** | Monitoring | Key system metrics (e.g., request latency, error rates, CPU/memory utilization) for all services must be collected and visualized on monitoring dashboards. |
| **NFR-5.3** | Alerting | An automated alerting system must be in place to notify the on-call team of critical issues, such as service unavailability, database connection failures, or significant spikes in error rates. |
