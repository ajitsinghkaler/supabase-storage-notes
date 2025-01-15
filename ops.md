# Supabase Storage how deployment works

Let's break down the deployment aspects of this Supabase Storage repository, focusing on how the various tools connect in the `docker-compose.yml` files, their interactions. We'll try to cover the full operations side of the repository.

## **Deployment Overview**

The repository provides several Docker Compose files to manage different deployment scenarios, primarily focusing on:

- **`docker-compose.yml`**: This file sets up a single-tenant environment with a PostgreSQL database, a connection pooler (pgBouncer), a MinIO S3-compatible storage, and the Storage API itself.
- **`docker-compose-multi-tenant.yml`**: This file sets up a multi-tenant environment, adding a multi-tenant database, Supavisor (a connection pooler and proxy for multi-tenant Postgres), MinIO, and the Storage API (configured for multi-tenancy).
- **`./.docker/docker-compose-infra.yml`**: Defines the basic infrastructure components shared between the single-tenant and multi-tenant setups.
- **`./.docker/docker-compose-monitoring.yml`**: Defines the basic monitoring components shared between the single-tenant and multi-tenant setups.

### **Component Interaction in `docker-compose.yml` (Single-Tenant):**

Here's a breakdown of the services defined in `docker-compose.yml` and their interactions:

1. **`storage` (Supabase Storage API):**

   - **Image:** `supabase/storage-api:latest`
   - **Purpose:** This is the core service that provides the Storage API, handling object storage logic, auth, etc.
   - **Ports:** Exposes port `5000` for incoming HTTP requests.
   - **Dependencies:** Depends on `tenant_db`, `pg_bouncer`, and `minio_setup`. It requires the tenant database to be ready and migrations to be run, also requires a bucket on the S3 compatible provider before starting; that's why `minio_setup` is a dependency here.
        <details>
        <summary>Environment Variables</summary>
        <ul>
            <li><code>SERVER_PORT: 5000</code>: the exposed port on the container</li>
            <li><code>AUTH_JWT_SECRET</code>: JWT secret to validate requests</li>
            <li><code>AUTH_JWT_ALGORITHM</code>: Algorithm to use with the JWT library</li>
            <li><code>DATABASE_URL</code>: Connection URL to PostgreSQL</li>
            <li><code>DATABASE_POOL_URL</code>: Connection URL to <code>pgBouncer</code> connection pooler</li>
            <li><code>DB_INSTALL_ROLES: true</code>: Indicates if roles need to be installed on the database (if it is set to <code>false</code>, it needs to be managed outside of the application)</li>
            <li><code>STORAGE_BACKEND: s3</code>: which type of backend to use (<code>s3</code> or <code>file</code>)</li>
            <li><code>STORAGE_S3_BUCKET</code>: Name of the S3 bucket to use</li>
            <li><code>STORAGE_S3_ENDPOINT</code>: S3 endpoint to use</li>
            <li><code>STORAGE_S3_FORCE_PATH_STYLE: true</code>: If true, uses the path instead of subdomains</li>
            <li><code>STORAGE_S3_REGION</code>: S3 bucket region</li>
            <li><code>AWS_ACCESS_KEY_ID</code> and <code>AWS_SECRET_ACCESS_KEY</code>: MinIO credentials</li>
            <li><code>UPLOAD_FILE_SIZE_LIMIT</code>, <code>UPLOAD_FILE_SIZE_LIMIT_STANDARD</code>: limits for file uploads</li>
            <li><code>TUS_URL_PATH</code>, <code>TUS_URL_EXPIRY_MS</code>: TUS settings</li>
            <li><code>IMAGE_TRANSFORMATION_ENABLED: "true"</code>: Enables or disables image transformations</li>
            <li><code>IMGPROXY_URL</code>, <code>IMGPROXY_REQUEST_TIMEOUT</code>: Configuration for the imgproxy URL and timeout</li>
            <li><code>S3_PROTOCOL_ACCESS_KEY_ID</code> and <code>S3_PROTOCOL_ACCESS_KEY_SECRET</code>: If the access key or secret are set on the request, those values will be used instead of the static <code>AWS_ACCESS_KEY_ID</code>, <code>AWS_SECRET_ACCESS_KEY</code></li>
            <li><code>S3_PROTOCOL_ALLOWS_SERVICE_KEY_AS_SECRET</code>: if true, allows using service keys as secrets</li>
        </ul>
        </details>

   - **Functionality:** This is the core of the storage engine; it uses various libraries from the repo, in particular:
     - The database classes such as `TenantConnection` to connect to the database
     - The `Storage` class to interact with the database and the underlying storage
     - All the auth validations using `jwt` and `apikey` plugins
     - Registers all the TUS, object, bucket, S3, and render HTTP endpoints

2. **`tenant_db` (PostgreSQL Database):**

   - **Image:** `postgres:15`
   - **Purpose:** Provides the PostgreSQL database that stores the buckets, objects, and file metadata
   - **Ports:** Exposes port `5432` for database access
   - **Healthcheck:** Check if the service is healthy using `pg_isready`
   - **Environment:**
     - `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`: PostgreSQL credentials

3. **`pg_bouncer` (pgBouncer Connection Pooler):**

   - **Image:** `bitnami/pgbouncer:latest`
   - **Purpose:** Manages a pool of connections to the PostgreSQL database. This helps in preventing database connection overload when a large number of connections come at once and also allows for transaction-level pooling
   - **Ports:** Exposes port `6432` for connections from the `storage` service
   - **Environment:**
     - `POSTGRESQL_USERNAME`, `POSTGRESQL_HOST`, `POSTGRESQL_PASSWORD`: PostgreSQL connection details
     - `PGBOUNCER_POOL_MODE`: Sets the pool mode to transaction, which is recommended for serverless or edge environments
     - `PGBOUNCER_IGNORE_STARTUP_PARAMETERS`: List of parameters to ignore on startup
     - `PGBOUNCER_STATS_USERS`: Users that are able to query stats

4. **`minio` (MinIO S3-Compatible Storage):**

   - **Image:** `minio/minio`
   - **Purpose:** Provides a local S3-compatible object storage that the Storage API will use
   - **Ports:** Exposes ports `9000` (API) and `9001` (console)
   - **Healthcheck:** Check if the service is healthy via TCP connection
   - **Environment:**
     - `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`: MinIO credentials
   - **Command:** `server --console-address ":9001" /data` starts the server in the defined port, exposing the console and also where the files are located

5. **`minio_setup` (MinIO Setup):**

   - **Image:** `minio/mc`
   - **Purpose:** Configures MinIO by setting up an alias and creating a bucket
   - **Dependencies:** Depends on `minio`
   - **Entrypoint:** Runs the `mc` command to create a bucket in MinIO

6. **`imgproxy`:**
   - **Image:** `darthsim/imgproxy`
   - **Purpose:** Handles image transformation using the imgproxy project
   - **Ports:** Exposes port `8080` for incoming requests
   - **Volumes:** The local folder `data` is mapped to the Docker folder `/images/data`
   - **Environment:**
     - `IMGPROXY_WRITE_TIMEOUT` and `IMGPROXY_READ_TIMEOUT`: Sets the read and write timeout for image operations
     - `IMGPROXY_REQUESTS_QUEUE_SIZE`: The requests to the queue size
     - `IMGPROXY_LOCAL_FILESYSTEM_ROOT`: The root folder where images are stored
     - `IMGPROXY_USE_ETAG`: Enable the usage of ETag when available
     - `IMGPROXY_ENABLE_WEBP_DETECTION`: Enable detection of WebP format images

### **Component Interaction in `docker-compose-multi-tenant.yml` (Multi-Tenant):**

This configuration includes all of the above plus additional services for multi-tenancy:

1. **`multitenant_db` (PostgreSQL Database):**

   - **Image:** `postgres:15`
   - **Purpose:** Manages the database schema for multiple tenants. It stores metadata and configuration information for each tenant
   - **Ports:** Exposes port `5433` for database access
   - **Healthcheck:** Check if the service is healthy using `pg_isready`
   - **Environment:**
     - `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`: PostgreSQL credentials
   - **Configs:** Loads the SQL schema on `/docker-entrypoint-initdb.d/init.sql`

2. **`supavisor` (Supavisor Connection Pooler and Proxy):**

   - **Image:** `supabase/supavisor:latest`
   - **Purpose:** Acts as a connection pooler and proxy for routing database connections in a multi-tenant setup. It will have a pool of connections for each tenant
   - **Ports:** Exposes ports `4000` (API), `5452` (session), and `6543` (transaction)
   - **Dependencies:** `multitenant_db`, `tenant_db`
   - **Healthcheck:** Check if the service is healthy by pinging `http://localhost:4000/api/health`
        <details>
        <summary>Environment</summary>
        - `PORT`, `PROXY_PORT_SESSION`, `PROXY_PORT_TRANSACTION`: Supavisor port configuration  
        - `DATABASE_URL`: Connection URL to the multi-tenant database  
        - `CLUSTER_POSTGRES: "true"`: Enable clustered PostgreSQL configuration  
        - `SECRET_KEY_BASE`, `VAULT_ENC_KEY`, `API_JWT_SECRET`, `METRICS_JWT_SECRET`, `REGION`: Other environment variables needed for the application  
        </details>

   - **Command:** Migrates the Supavisor data structure with `/app/bin/migrate` and then
     starts the server with `/app/bin/server`

3. **`supavisor_setup` (Supavisor Setup):**
   - **Image:** `supabase/supavisor:latest`
   - **Purpose:** Sets up an initial tenant in the Supavisor database
   - **Dependencies:** Depends on `supavisor`
   - **Command:** Creates the tenant inside the Supavisor database via an HTTP PUT request

### **The Role of MinIO, pgBouncer, and Supavisor**

- **MinIO:** Acts as a local object storage system that is used by the Storage Service; it allows easy setup and use without the need for a cloud provider. The Storage Service uses a local bucket `supa-storage-bucket` as a default to store files.
- **pgBouncer:** Reduces the number of direct connections to the PostgreSQL database using connection pooling, which helps in preventing database overload and allows for maintaining a healthy database connection. It also enables transaction-level pooling. This is a single connection pooler, which is used only for the storage application.
- **Supavisor:** Supavisor is a multi-tenant connection pooler that connects to the multi-tenant database and acts as a proxy for all the tenant databases. It uses a database to handle connection pooling and manage tenants.

### **Monitoring**

The repository also contains additional configurations and tooling that are important for monitoring:

- **`.docker/docker-compose-monitoring.yml`:** This file sets up monitoring tools such as:

  - `pg_bouncer_exporter`: Exports metrics from pgBouncer
  - `postgres_exporter`: Exports metrics from PostgreSQL
  - `prometheus`: Collects all the metrics from services like storage, PostgreSQL, and Supavisor
  - `grafana`: Used for visualizing collected data from Prometheus, and it has a set of dashboards that you can use for monitoring storage, PostgreSQL, and Supavisor
  - `jaeger`: This is a collector for traces that are created on the application
  - `otel-collector`: This is a collector for traces

- **`.github/workflows/`:** This directory includes GitHub Actions workflow to automate build, test, release, deployment, and documentation tasks
- **`.releaserc`:** This file defines release configurations for `semantic-release`

## **Deployment Process Summary**

1. **Local Development:** For local development `docker-compose-infra.yml` is used to create all the shared services.
2. **Production Deployment:** In production, the `storage` service should be deployed in a cloud environment using a database and an S3-compatible object store.
   - The Docker images used for the deployments are built using the workflow defined on `.github/workflows/release.yml`, which are then published to Docker Hub and GHCR.
   - `mirror.yml` is used to mirror the newly published versions on different providers.
3. **Multi-Tenant Configuration:** When setting up multi-tenancy:
   - Create a multitenant database.
   - Set up a Supavisor instance.
   - Use the admin API to create a new tenant specifying all configurations needed.
   - The `infra` scripts can be used to restart and deploy the basic infrastructure.
4. **Monitoring:**
   - The `docker-compose-monitoring.yml` is used to create the monitoring services.
