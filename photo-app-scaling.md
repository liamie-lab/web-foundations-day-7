
# SnapShare Photo App Scaling Plan

## 1. Assumptions and Daily Active Users

- Registered users: 10 million.
- 10% of registered users are active daily.
- Each active user uploads 1 photo per day.
- Each active user views 50 feed pages per day.
- Average photo size is 2 MB.
- Each photo has a 50 KB thumbnail.
- A day has 86,400 seconds, and a year has 365 days.
- Peak traffic is 5 times average traffic.

Daily Active Users (DAU):

DAU = 10,000,000 × 10%

DAU = 1,000,000 users per day

## 2. Traffic and Storage Calculations

### Uploads per Second

Uploads per day = 1,000,000

Average uploads per second = 1,000,000 ÷ 86,400
= 11.57 uploads/second

Peak uploads per second = 11.57 × 5
= 57.87 uploads/second

### Feed Views per Second

Feed views per day = 1,000,000 × 50
= 50,000,000 views

Average feed views per second = 50,000,000 ÷ 86,400
= 578.70 views/second

Peak feed views per second = 578.70 × 5
= 2,893.52 views/second

### Photo Storage Per Year

Original photos = 1,000,000 × 2 MB × 365
= 730 TB per year

Thumbnails = 1,000,000 × 50 KB × 365
= 18.25 TB per year

Total storage = 730 + 18.25
= 748.25 TB per year
## 3. Is the System Read-Heavy or Write-Heavy?

SnapShare is a read-heavy system because users view 50 feed pages per day but upload only 1 photo per day.

This means the system handles many more read requests than write requests.

To handle this, we should use a CDN to deliver photos quickly, a cache to store frequently accessed data, and a read replica to handle database read requests without overloading the main database.

## 4. Why Photos Should Not Be Stored in the Database

Photos should be stored in object storage, such as Amazon S3, because image files are large and require scalable, cost-effective storage.

The database should store metadata such as photo ID, user ID, upload time, caption, and the object's storage URL.

This keeps the database smaller and makes photo delivery more efficient.

These estimates exclude backups, replication and other overhead.

## 5. SnapShare Architecture Diagram

```text
                     USERS
                       |
                       v
                      CDN
                       |
                       v
                 LOAD BALANCER
                       |
                       v
                  APP SERVERS
                  /          \
                 v            v
              CACHE        DATABASE
                              |
                              v
                         READ REPLICA

APP SERVERS
     |
     v
OBJECT STORAGE (Original Photos)
     |
     v
MESSAGE QUEUE
     |
     v
THUMBNAIL WORKER
     |
     v
OBJECT STORAGE (Thumbnails)
```

## 6. Explanation of Each Architecture Component

1. **CDN:** Delivers photos and thumbnails from servers close to users, reducing loading time and traffic to the main servers.

2. **Load Balancer:** Distributes incoming requests across multiple application servers to prevent any one server from becoming overloaded.

3. **Application Servers:** Handle user requests, manage uploads, retrieve feed data, and communicate with the database and object storage.

4. **Cache:** Stores frequently requested data so the application can respond faster and reduce database workload.

5. **Database:** Stores structured information such as user details, photo metadata, captions, and storage references.

6. **Read Replica:** Handles database read queries to reduce the workload on the primary database.

7. **Object Storage:** Stores original photos and generated thumbnails separately from the database, allowing image storage to scale efficiently.

8. **Message Queue:** Holds thumbnail-generation jobs until a worker is ready to process them.

9. **Thumbnail Worker:** Processes queued jobs by creating smaller versions of uploaded photos and saving them to object storage.

## 7. Photo Upload Flow

1. The user selects a photo and uploads it through the application.
2. The load balancer routes the request to an available application server.
3. The application server validates the photo and checks the user's permissions.
4. The original photo is uploaded to object storage.
5. The application server saves the photo metadata and storage reference in the database.
6. A thumbnail-generation job is added to the message queue.
7. A thumbnail worker picks up the job, creates a smaller version of the photo, and saves it to object storage.
8. The CDN delivers the original photo or thumbnail to users when they view the feed.

This asynchronous process allows photo uploads to continue without waiting for thumbnail generation to finish.

## 8. Trade-offs and Design Decisions

### 1. Object Storage vs Database Storage
Object storage is suitable for photos because it scales well and is cost-effective. However, the application must manage storage references and permissions carefully.

### 2. Caching vs Fresh Data
Caching improves response times and reduces database load. However, cached information can become outdated, so the system needs a cache invalidation strategy.

### 3. Asynchronous Thumbnail Generation
Using a message queue and background workers makes uploads faster because users do not need to wait for thumbnails to be generated. However, thumbnails may take a short time to become available.

### 4. CDN vs Direct Server Delivery
A CDN reduces latency and application server workload by delivering images from nearby locations. However, CDN services introduce additional costs and require cache management.

### 5. Read Replicas vs Data Consistency
Read replicas improve read capacity, but updates may take time to appear on replicas. The application should use the primary database when immediately consistent data is required.

### 6. Scalability vs Cost
Using multiple servers, caching, replicas, and object storage improves scalability and reliability. However, these services increase infrastructure costs and require monitoring.


