# Supabase storage Repository Overview
To understand the supabase storage repository, we need to understand the various components and how they interact with each other. For that we will break down how the various components in this Supabase Storage repository interact with each other. It has many components, but I first list and discuss the major ones:

## **Core Concepts:**

*   **Storage Backend:** This is the service, responsible for managing object storage. It handles 2 storage backends (S3 and local filesystem), manages metadata in a database, enforces access control (RLS), and provides APIs for clients.
*   **Fastify:** This is the web framework used to create the HTTP API for the Storage service.
*   **PostgreSQL:** It is the primary data storage for metadata (objects, buckets, user defined metadata, multipart uploads, etc..)
*   **S3:** It is one of the supported storage backends where the actual files are stored in object stores like AWS S3, Digital Ocean Spaces, Cloudflare R2, etc..
*   **TUS:** It is the protocol implemented for resumable uploads.
*   **Image Proxy ([Imgproxy)[]:** An external service used for transforming images (resizing, formatting).
*   **Metrics:** Prometheus used for tracking the performance of the application and monitoring all operations.
*   **Authentication/Authorization:** Enforces access control using JWTs and Postgres Row Level Security (RLS).
*   **Tenants:**  Used in multi-tenant configurations to isolate data and configurations for different customers. This is how supabase manages multiple clients in a single instance.
*   **Tracing:** OpenTelemetry (OTel) for distributed tracing, used to debug performance issues.
*   **Streaming & Events:** Streams and message queue systems (PgBoss, pubsub) for asynchronous operations, creating, and reacting to events (webhooks).
*   **Schemas:** JSON schemas for request and response validation.

## Interaction from a request lifecycle perspective
Here we will take a look at how the various components interact with each other when a request is made to the storage service so that we can understand the flow of data and how the various components are used to perform the request and interact with the request.

**Interactions in Detail:**

1.  **Client Request:**

    *   A client sends an HTTP request to the Storage API (using Fastify).
    *   The request can be for various operations like:
        *   Creating a bucket.
        *   Uploading/downloading objects.
        *   Transforming images.
        *   Listing objects in a bucket.
        *   Initiating or completing a TUS upload.
        *   S3 protocol operations

2.  **Authentication and Authorization:**

    *   **JWT Authentication:** The `jwt` plugin verifies the request's `Authorization` header for a valid JWT.
    *   **Tenant ID Extraction:** The `tenant-id` plugin extracts the tenant ID from the subdomain of the `x-forwarded-host` header (in multi-tenant setups) or from an environment variable for single-tenant.
    *   **API Key Authentication:** Admin requests (e.g. multi-tenant management) use the `apikey` plugin. Which checks for the `apikey` header in the request.
    *   **Request Context:** After the JWT validation, it adds tenant specific information to the request object.
        *   `request.tenantId` is set with the extracted or default tenant ID.
        *   `request.isAuthenticated` is set to `true` if the JWT is valid, `false` otherwise.
        *   `request.jwtPayload` contains the decoded JWT claims.
        *   `request.owner` contains the subject (`sub`) claim from the JWT.

3.  **Request Handling and Validation:**

    *   **Fastify Routing:** Fastify routes the request to the appropriate handler function.
    *   **Schema Validation:** Each route is associated with a JSON schema. The body, params, and querystring of the request are validated using [ajv](https://github.com/ajv-validator/ajv) against the defined schema, to ensure that request are proper.
    *   **Plugins**: Several plugins are attached to the request object to perform certain action.
        *   **`db`**: It's an authentication specific connection to the database, that has all necessary information to be able to set the correct role in Postgres.This is used to perform all the database operations and helps in managing RLS(Row Level Security) for the tenant.
        *  **`storage`**: Instance of the storage class with all the required backend/database specific instance of the classes.
        *  **`metrics`**: Collects request level metrics like number of requests, timing to provide performance data on the application.

4.  **Storage Layer Interaction:**

    *   **Storage class**: The storage class is an orchestrator for all operations, and interacts with backend and database abstractions to fetch the data or perform an action.
    *   **Object Storage:**
        *   **S3 Backend:** When the storage configuration is S3 then storage operations will be interacting with an S3 compatible object store. The `s3` plugin creates a new S3 client for each request, with the tenant specific settings (region, endpoint etc..).
        *   **File Backend:** When the storage configuration is local then the storage operations will be interacted with a local disk to perform all the file operation.
        *   **`Upload` Class:** It handles all the multipart or single part upload to the backend, this class also handles the logic when upload limits are defined in the bucket configuration.
    *   **Database Interaction:**
        *   The `db` plugin creates an authenticated database connection (`TenantConnection`) using [knex](https://knexjs.org/), allowing for database operations.
        *   Row-Level Security (RLS) is used to enforce access control on the database level, based on JWT claims and the tenant ID.

5.  **Image Transformation:**

    *   If the request is for image transformation, the `ImageRenderer`(This is a class that handles the creation of actual image) is used, and a secure URL is generated for image proxy to fetch the image from object stores and then transform it as requested.
    *   Rate limiting is applied to prevent abuse of the image transformation service when enabled.

6. **TUS Protocol**

    *  When the TUS protocol is used to upload files, the request is routed to `TusServer`, this manages the different operation in the TUS protocol and orchestrates all operations
        *  `FileStore` or `S3Store` manages the storage of the upload chunks (when uploading resumable files)
        * `PgLock` manages the concurrency of upload operations. This makes it such that 2 users are not performing operations on a file at the same time.
        * `AlsMemoryKV` manages the metadata storage in the TUS context. This is an async local storage instance that TUS uses to make resumable uploads.
        
7. **Event System**
    * When an operation succeeds, a queue message might be generated. Events are used for the following operations. This queue is made in Postgres using `PgBoss`.
    *  Webhooks: Used to notify external services about file events (upload/delete).
    *  Migrations: Handles async migrations of tenant databases, when new migrations are available.
    * Object Admin Delete: Asynchronously deletes the previous versions of an object when it gets updated.

8.  **Asynchronous Operations:**
   *  Some operations like webhooks, background migrations, file processing, and deletion can be processed asynchronously through the queue system.
   *  **PgBoss:** Used as the queue mechanism for asynchronous operations, persisting tasks to the database.
        *  `RunMigrationsOnTenants` queue handles the migrations of tenant databases.
        *  `ObjectAdminDelete` queue handles the deletion of an object.
        *  `Webhook` queue handles the sending of webhooks to external services.
   *  **Postgres Pubsub:** For real-time notifications, used for cache invalidation when the multi-tenant setup is in use. This uses `node` EventEmitter to create the pubsub system.
        *   Invalidate the tenant config cache when the tenant information is changed or deleted.
        *   Invalidate the S3 credential cache when the credentials are changed or deleted.

9.  **Metrics and Monitoring:**

    *   **Prometheus:** Used for gathering metrics.
    *  `fastify-metrics`: collects http requests.
    *  `pino-logflare`: is used as a custom pino logging strategy to push logs to logflare.
    *   Custom metrics are exposed through  the `/metrics` endpoint to track database pool usage, queue sizes, s3 object transfers, http request latencies etc...

10. **Tracing:**
    *   **OpenTelemetry:** Used to trace the execution of requests across different components.
    *  Spans are added to method calls for a better way to track executions.
    *  When `debug` or `logs` tracing level is set, OTel data is added to the logs.
    * The OTel collector collects traces in various formats, that are exported to Jaeger or other external consumers.

**Data Flow Summary:**

1.  **Request:** Client makes an HTTP request.
2.  **Authentication:** JWT, API Key, and tenant extraction.
3.  **Authorization**: RLS and other code based authorization.
4.  **Data Access:** Data is read/written from/to the backend or database.
    * If files are needed to be transferred then files are downloaded from S3 or local system.
    *  If only metadata is needed, then data is fetched from database.
5. **Image Transformation:** If transformation is required then Imgproxy url is built and the asset is fetched from it.
6.  **Asynchronous Task:** If needed, a task is enqueued for async processing.
7.  **Response:** Data is returned to client.
8.  **Monitoring & Tracing:**  Metrics and traces are generated along the way, and exported to the corresponding consumers.

## **Key Interaction Points:**

*   **HTTP Layer (Fastify) <-> Storage:** Fastify routes requests to the `Storage` class or `s3protocolHandler` based on the request parameters or headers. The Storage class uses the DB and S3 client to implement the required action.
*   **Storage <-> Database (PostgreSQL):**  The `Storage` class utilizes the database interface for metadata persistence and RLS enforcement.
*   **Storage <-> Storage Backend:** The `Storage` class relies on a backend adapter (either `S3Backend` or `FileBackend`) to perform the object storage operations.
*   **Storage  <-> Imgproxy:** The `ImageRenderer` class communicates with the `imgproxy` for image transformations by generating presigned urls.
*   **Fastify <-> Queue (PgBoss):** Queue jobs are created by Fastify routes and handled by workers in a separated pool, enabling async behaviour.
*   **Fastify <-> Authentication:** Fastify utilizes auth plugins to authorize all the requests and set the `request.jwtPayload`, `request.isAuthenticated`, `request.tenantId`, and `request.owner` variables.
*   **Fastify <-> Logging:** Fastify registers a pino logger in the request context and sets up structured logging for all the requests.
*   **Fastify <-> Monitoring:** Fastify registers prometheus metrics middleware, to track the HTTP metrics and adds OTel spans for tracing purposes.
*   **Internal Components:** The core components use shared utility libraries and classes, to do perform core operations, like streaming, concurrency control, or error handling.

The Supabase Storage repository is a highly flexible object storage service. It combines a core with a set of features to store and manipulate files. It can be deployed in several ways, either a single instance, or a multi-tenant instance with tenant configurations.

## **Key takeaways:**

*   **Pluggable Architecture:** The system is highly modular and built to enable multiple backend and integration points (like S3 or local disk, HTTP, TUS, external image transformers),
*   **Performance:** Caching, efficient connection pooling, and async mechanisms.
*   **Observability:** OTel and prometheus to provide visibility into application's health and performance.
*  **Multi-tenancy support:** Tenant separation with DB isolation using connection pooling and RLS.

This detailed overview should help you understand how the components interact in this system.