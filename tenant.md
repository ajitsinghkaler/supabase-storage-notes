# Tenant Management in Supabase/Storage

Let's learn how tenant data is managed within this Supabase Storage repository, focusing on how it's kept separate and how they are set up in the multi-tenant environment.

## **Tenant Data Separation Lifecycle**

This system implements a multi-tenant architecture, which means that a single instance of the application serves multiple tenants (organizations, projects) while ensuring data isolation and security. Here's the lifecycle of how tenant data is kept separate:

1.  **Tenant Creation (Admin API):**

    -   **Admin Request**: An administrator makes a request using the multi-tenant admin API to create a new tenant. This typically involves sending a POST request to `/admin/tenants/{tenantId}`.
    -   **Authentication:** The request requires a valid admin API Key, which is validated using the `apiKey` plugin.
    -   **Tenant Data:** The request body usually includes data such as:
        -   `tenantId`: A unique identifier for the tenant (often a UUID).
        -   `anonKey`: The anonymous public key for the tenant.
        -   `databaseUrl`: A specific URL to be used for a particular tenant.
        -   `jwtSecret`: A tenant-specific JWT secret to generate and validate JWTs.
        -   `serviceKey`: A service key used for the tenant to bypass row-level security for database reads and writes.
        -   `features`: Features for the tenant such as `imageTransformation` and `s3Protocol`.
    -   **Database Entry:** The request handler calls the `multitenantKnex` client and inserts a new record in the `tenants` table, located in the multitenant database. Note that the sensitive values are encrypted before being persisted in the database.
    -   **Tracing:** If tracing is enabled for the admin server, a trace will be created named `tenants.create` using the `ClassInstrumentation` plugin.

2.  **Tenant Request (HTTP Layer):**

    -   **`x-forwarded-host` Header:** When a request enters the main Storage service, the `tenantId` plugin checks for the `x-forwarded-host` header. This header indicates the specific tenant to whom the request belongs. If the header is not provided, a 400 error is returned.
      -   **RegExp Extraction**: The `REQUEST_X_FORWARDED_HOST_REGEXP` configuration is used to extract a valid tenant ID from the header; if the header doesn't match a regexp, a 400 error is thrown.
    -   **Tenant Id**: The extracted tenant ID is stored in `request.tenantId` for use in subsequent operations.
    -   **Context**: Every request has the `tenantId` available in the request object that can be used to retrieve the data for a particular tenant.
    -   **Database Connection**: When a database connection is created, the multitenant database configuration is used by `multitenantKnex`, which in turn allows for database connection pooling at the application server level.
     - **Tenant-specific database pool:** Every request will create a tenant-specific database pool, which is also used for each database operation.
       - In case a database connection pool is set for a given tenant, each request for that tenant will use its own database connection pool.

3.  **Tenant Configuration Lookup (Internal Layer):**

    -  **Cache**: The `getTenantConfig` function first tries to read from a cache, identified using the tenantId from the request. If the entry is not found, it will continue to the next step.
    -   **Database Fetch**: The system uses the tenant ID to query the `tenants` table in the multi-tenant database and fetch the configuration for that tenant.
        -  **Mutual Exclusion**: To prevent multiple simultaneous lookups that could lead to thundering herd issues, a mutex is implemented using the `createMutexByKey` function. This ensures that only one request can perform the lookup at a time, thereby reducing contention and improving efficiency.
    -   **Data Decryption:** The data, including the `databaseURL`, `serviceKey`, `jwtSecret`, etc., are decrypted using the configured encryption key. This ensures that no sensitive data is visible in the database.
    -   **In-memory cache**: The tenant config, once obtained, is saved in the in-memory cache for fast access in the next requests that come for the same tenant.
    -   **Tracing:** Every time the tenant is fetched, if tracing is enabled, a span named `tenant.fetch` is created with the tenantId.
    -   **Pooling:** The retrieved configuration is used to configure the database connection, or if set, the database connection pool.

4.  **Resource Handling with Tenant Specific Context (Storage, Database Layer):**

    -   **Tenant Specific Connections**: The created `TenantConnection` from `getPostgresConnection` sets up the connection with the tenant data and also the custom settings, which allow using RLS with the `setScope` method.

    -   **RLS Authorization**: Once an object is created or read in the `storage` layer, the underlying SQL calls will automatically respect RLS policies.

    -   **Scoped Operations:** The operations within `Storage`, `ObjectStorage`, and the internal `database` operate within the tenant's context using the `tenantId`, ensuring the correct credentials and settings are used for the given resources.

6. **S3 Credentials Management**
 - **S3 Credentials**: Each tenant can also have a specific S3 key and secret, which is managed in the `tenants_s3_credentials` table.
 - **Unique Access Key**: Each credential for the tenant is tied to a unique access key.
 - **Rotation**: These credentials can be rotated when creating new credentials for a particular tenant.
 - **Access Restrictions**: The service key or anon key cannot be used to sign a request to a particular tenant.
 - **Caching**: The `getS3CredentialsByAccessKey` caches all the S3 credentials in memory for a specific amount of time.
   - If there is an update or deletion, the cache is automatically invalidated via a PubSub listener in `listenForTenantUpdate`.
 - **Automatic Scope**: The `getS3CredentialsByAccessKey` will also return a `claims` payload that can be used to create a scoped JWT token.

**Key Elements for Data Isolation**

-   **Multi-Tenant Database:** All tenant-specific configuration information such as database URLs, JWT secrets, and service keys is managed in a centralized multi-tenant database, separated from the actual storage data.
-   **Request Context:** The tenant ID and configuration are part of the request context, which is used for all subsequent operations.
-   **In-Memory Cache:** Frequent tenant configurations are cached in memory for quick retrieval.
-   **Data Encryption:** Sensitive data, such as passwords and secrets, are encrypted at rest in the multi-tenant database.

**How It Works in Practice:**

1.  **Tenant Setup:** An administrator creates a new tenant using the multi-tenant API, which stores the tenant configurations.

2.  **Client Request:** A client attempts to access a resource, providing a JWT or presigned S3 token in the Authorization header, or using an upload signed URL.

3.  **Tenant Identification:** The system extracts the tenant ID from the request using the `x-forwarded-host` header sent by the load balancer.

4.  **Configuration Lookup:** The `getTenantConfig` function retrieves the specific configuration for the tenant, including the database URL, secrets, etc.

5.  **Database Connection:** A database connection is made using the tenant-specific configuration, including tenant-specific database pool options if any.

6.  **RLS Enforcement:** Queries against the storage database use the custom functions `auth.uid` and `auth.role` using the extracted JWT from the header.

7.  **Response Generation:** The system responds to the client with the requested data, ensuring data isolation and security at all stages.

This is how the multi-tenant application achieves strong data isolation.
