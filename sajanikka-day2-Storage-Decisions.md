# BookSwap — Storage and Cache Decisions





## Day02 - Deliverable 2

##### 



##### 1\. Data Inventory





The book listing estimate of approximately 50,000 is provided in the challenge. Other figures are planning estimates and should be reviewed against actual usage data.





|**Data type**|**Example record**|**Volume estimate (1 year)**|**Read/write ratio**|
|-|-|-|-|
|Book listing|Book ID, title, author, condition, owner ID, availability|Approximately 50,000 listings|10:1|
|Member|Member ID, name, email, building|Approximately 10,000 members (planning estimate)|5:1|
|Borrow request|Request ID, book ID, borrower ID, status|Approximately 100,000 requests (planning estimate)|3:1|
|Loan record|Loan ID, book ID, borrower ID, borrowed date, return date|Approximately 50,000 records (planning estimate)|3:1|
|Book photo|Photo ID, book ID, image location|Up to 50,000 photos (planning estimate; actual count may vary)|10:1|
|Notification job|Job ID, recipient, notification type, status|Approximately 200,000 jobs (planning estimate)|1:1|





The read/write ratios are initial estimates, not measured production statistics. Book listings and photos are expected to be read frequently, while borrow requests and loans need reliable updates when users borrow or return books.







##### 2\. Storage Selection







|**Data type**|**Chosen store**|**Why this store?**|**Why not the alternatives?**|
|-|-|-|-|
|Book listing|Azure SQL Database|Book listings have structured fields and relationships with members, borrow requests, and loans. SQL supports transactions and constraints to protect data consistency.|A document database is not necessary for this relational data. Keeping the records in a simple file would make searching, updating, and enforcing relationships harder.|
|Member|Azure SQL Database|Member information has a defined structure and connects to book listings and borrowing activity through member IDs.|Blob Storage is designed for files, not frequent member lookups and relational queries.|
|Borrow request|Azure SQL Database|Requests need reliable status changes and relationships to books and members. Transactions help prevent inconsistent updates.|Redis is temporary cache storage and should not be the permanent record of a request. A file-based store would make concurrent updates and queries difficult.|
|Loan record|Azure SQL Database|Loan and return records need accurate dates, statuses, and links to books and borrowers.|A cache or message queue is not suitable as the permanent source of truth for loan history.|
|Book photo|Azure Blob Storage|Photos are binary files and may be large. Blob Storage is designed for storing files and can scale independently from the database. SQL stores the photo reference and related metadata.|Storing image files directly in SQL would increase database size and backup/restore costs. Redis is not suitable for permanent image storage.|
|Cached book listings and search results|Azure Cache for Redis|Redis can return frequently requested data quickly and reduce repeated read queries to SQL.|SQL remains the permanent source of truth. Redis should not be used as the only store because cached data can expire or be lost.|
|Notification jobs|Azure Service Bus|A queue separates the main application from slower background work. Messages can wait safely while a worker is unavailable, subject to configured retention and capacity.|Sending every notification directly during an API request can increase response time and make the request depend on the email service being available. Redis is not a durable job queue replacement.|
|Email delivery|Azure Communication Services Email|It provides the service used by a background worker to send email notifications. Delivery attempts and outcomes should be recorded where needed.|Azure SQL stores application records but does not itself send emails. Service Bus carries jobs; it does not deliver the email to the recipient.|







###### Source of Truth



The source of truth is the permanent authoritative location for each type of data.





* **Book listings, members, borrow requests, and loan records:** Azure SQL Database.



* **Original book photos:** Azure Blob Storage. Azure SQL Database stores the photo reference and related metadata.



* **Cached book listings and search results:** Azure Cache for Redis is temporary only. Azure SQL Database remains authoritative.



* **Pending notification work:** Azure Service Bus holds queued messages until they are processed or handled according to queue policies.



* **Notification history and processing status:** Store these in Azure SQL Database if the application needs a durable audit trail.



* **Email sending and delivery outcome:** Azure Communication Services Email performs the sending. The application should record relevant outcomes rather than treating a queued message as proof of delivery.



A cache or queue must not replace the permanent records in SQL or Blob Storage.







##### 3\. Cache Plan







**What is hot enough to cache?**



I would cache frequently accessed book listing details and popular search results because many users may view the same books without changing them. This can reduce repeated database queries and improve response time.



I would not cache the final borrowing decision or use cached availability as the authority for a borrow operation. Availability may change when another user borrows a book. The application must check and update the authoritative SQL records when processing a borrow request.



I would also avoid caching sensitive member information unless there is a clear need and suitable access controls.







**Cache-aside pattern**



The application first checks Redis. If the requested data is not in the cache, it reads from SQL and then stores the result in Redis for a limited time.







FUNCTION getBookListing(cacheKey, sqlQuery):



&#x20;  cachedValue = Redis.GET(cacheKey)





&#x20;  IF cachedValue EXISTS:



&#x20;      RETURN cachedValue





&#x20;  result = AzureSQL.EXECUTE(sqlQuery)





&#x20;  IF result was read successfully:



&#x20;     Redis.SET(cacheKey, result, TTL = 300 seconds)





&#x20;  RETURN result





This is cache-aside: the application manages cache reads and fills the cache after a cache miss. Production code should also handle Redis and SQL errors appropriately.







**TTL and invalidation strategy**







* **TTL:** 300 seconds (5 minutes).



* **Reason:** Book listings may change, so a short TTL limits how long cached information can remain outdated while still reducing repeated database reads.



* **After a successful book update:** Invalidate the affected book-detail cache and any related listing or search cache entries.



* **If invalidation fails:** Record and monitor the failure. The TTL provides a fallback so the cached entry eventually expires.



* **If Redis is unavailable:** Attempt to read from SQL directly, subject to database availability and application limits.



* **For borrowing:** Check current availability and make the authoritative change in SQL within a suitable transaction or concurrency-safe operation. Do not rely on cached availability to approve a borrow.



Redis improves read performance, but SQL remains authoritative.





##### 

##### 4\. Queue Plan





**Which work goes on a queue, and why?**



I would use Azure Service Bus for background tasks that do not need to finish before the API responds, including:





* Sending a notification when a borrow request is created or updated.



* Sending a notification when a book is borrowed or returned.



* Preparing and sending scheduled notification summaries.





The application should first save the relevant business change in Azure SQL Database. It can then publish a notification job to Service Bus for a background worker to process. The worker sends the email through Azure Communication Services Email and records the processing outcome when required.



To reduce the risk of a database update succeeding while its message is never published, I would consider the transactional outbox pattern. The application saves the business change and an outbox event in the same SQL transaction. A separate publisher sends the outbox event to Service Bus and marks it as published after confirmation.







**What happens if the consumer is down for 30 minutes?**



If the notification worker is unavailable for 30 minutes:







1. Messages wait in the queue. Service Bus retains them according to the configured message TTL, queue capacity, and service limits. The queue must be configured to tolerate the expected outage and backlog.
2. The main application can continue where possible. Book and borrowing operations can continue if their required services are healthy and the application can safely record the notification work. Notifications will be delayed.
3. The worker resumes processing. When the worker comes back online, it receives and processes waiting messages.
4. Temporary failures are retried. Use a bounded retry policy with appropriate delays for transient errors.
5. Repeated failures are isolated. Messages that cannot be processed after the configured delivery attempts should be moved to the dead-letter queue (DLQ) for investigation and controlled reprocessing.
6. Duplicate processing is handled safely. Messages may be delivered more than once. Use a stable notification or event ID and idempotent processing so retrying a message does not cause unintended duplicate actions.
7. The system is monitored. Track queue depth, the age of the oldest message, processing failures, retries, and DLQ messages. Alert if the backlog grows or messages remain unprocessed for    too long.



The application must not report that an email was delivered merely because a job was added to the queue. Queueing means the work is waiting to be processed; sending and delivery outcomes are separate stages.

