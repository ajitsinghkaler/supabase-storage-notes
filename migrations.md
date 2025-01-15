# Supabse storage migrations

Let's learn about database migrations in this Supabase Storage repository, covering both single-tenant and multi-tenant setups, and exploring the three different migration strategies: "on request," "progressive," and "full fleet."

## **Single-Tenant Migrations**

In a single-tenant setup, we have one admin application interacting with a single database. Here's how migrations are handled:

1.  **Migration Files:**

    -   Migrations are stored in the `migrations/tenant` directory.
    -   Each migration file is a SQL file and has a numbered prefix, e.g., `0001-initialmigration.sql`, `0002-storage-schema.sql`, that indicates in which order they should be applied.
        -   They may contain operations to create tables, indices, change columns, or define functions, etc.
    -   **`internal/database/migrations/types.ts`:** This file defines a mapping of the different migration names with their associated id.

2.  **`runMigrationsOnTenant` Function (src/internal/database/migrations/migrate.ts):**

    -   **Connection:** This function creates a database connection by using the `databaseUrl` and creates a `Client` instance from `pg` library.
    -   **`connectAndMigrate` Function:** This helper function is invoked with the database configuration and path for the migrations, and it takes care of applying the migrations to that given database.
        -   It reads all the files in a particular directory using `loadMigrationFilesCached`.
        -   If the `migrations` table is not present in the database it creates it.
        -   It reads the applied migrations using the `migrations` table.
            -   If there are applied migrations, it refreshes the migration position. This is for backport compatibility.
        -   It compares the applied migrations with the available migrations in a directory using `validateMigrationHashes`, and if there are some issues then it will throw an error and stop migrations unless `dbRefreshMigrationHashesOnMismatch` option is set to true, this will update migration hashes on the `migrations` table if there is a mismatch.
        -   It will filter migrations that are pending to be run using `filterMigrations`.
        -   For each pending migration, the system runs `runMigration` with the SQL file that will perform the schema updates.
        -   It sets the `search_path` for every session, by using `SET search_path TO ...`.

3.  **How It's Triggered:**

    -   When the main storage application starts up (`src/start/server.ts`) the function `runMigrationsOnTenant` is called using the `databaseURL`.
    -   On every request the migrations are verified to be up to date, if not, the `db-init` will trigger the method to make sure the database is up to date.
        -   This is only enabled when the application is not in multi-tenant mode.

**Multi-Tenant Migrations**

In a multi-tenant setup, we have multiple tenants sharing the same application instance, each with its own data but potentially sharing some database schemas. Here's how migrations are handled:

1.  **Multi-Tenant Database Migrations:**

    -   **Files:**  These migrations are stored in the `migrations/multitenant` directory and they have the same schema as single-tenant migrations.
    -   **Purpose:** These migrations are used for database changes that affect the multi-tenant database schema.
    -   **Execution:** The `runMultitenantMigrations` function is called when the server starts to apply changes to the database which is used to manage the different tenants (`docker-compose-multi-tenant.yml`).
    -   **Connection**: The function uses the `multitenantKnex` database client to connect to the multitenant database.

2.  **Tenant-Specific Migrations:**
    -   **Tenant `migrations` Table:** Each tenant has their own private database schema and the system tracks migration state in a `migrations` table for every tenant.
    -   **`runMigrationsOnTenant` Function:**
        -   It uses the `databaseUrl` from each tenant to connect to each database.
        -   The process from applying those migrations is identical to the single tenant migrations.
    -   **Tenant Tracking**:
        -   The `tenants` table in the multi-tenant database, holds the `migrations_version` and `migrations_status` which indicates up to what migration has been applied to a particular tenant and if the migration was successful or not.
        -   The system uses the `cursor_id` to paginate through the available tenants.
        -   The `listTenantsToMigrate` returns an async iterator that lists all the tenants where `migrations_status` is not set to `"COMPLETED"` or `migrations_version` is different than the latest migration.

## **Migration Strategies (Multi-Tenant)**

In a multi-tenant environment, the repository uses three different strategies to apply pending migrations:

1.  **`MultitenantMigrationStrategy.ON_REQUEST` (On Request):**

    -   **How it works:**
        -   Every request is checked using the `db` plugin for pending migrations using the tenant information present on the request.
        -   The plugin uses the function `hasMissingSyncMigration` which checks if the migrations have been completed or not on the database, before proceeding with the request.
        -   If the migration status is not set to `COMPLETED` or `migrations_version` is not the latest then all pending migrations will be executed for the specific tenant. This operation happens every time an application does a database query for this specific tenant.
        -   To avoid concurrent migrations on the same tenant the migrations use `createMutexByKey` to execute the migration sequentially and only once per tenant at a time.
    -   **Use Case:** This is useful if you want every request to have all the migrations applied, and you want to prioritize safety instead of performance.

2.  **`MultitenantMigrationStrategy.PROGRESSIVE` (Progressive):**

    -   **How it Works:**
        -   The `progressiveMigrations` class (in `src/internal/database/migrations/progressive.ts`) is configured to run in a loop at a specific interval checking for any tenants that need to be migrated, this check will be done in the background on a specific server.
        -   If a new tenant is added, or there is a new migration available, then the tenant will be added to a list which is used by `createJobs` to perform the migrations asynchronously.
        -   If a request is made and it has pending migrations, the plugin will trigger the migration immediately using the `runMigrationsOnTenant` and use `updateTenantMigrationsState` to set the migrations as completed, in this case, the migrations will block the requests, so you don't have a version inconsistency.
            -   It checks on each request if there is a SYNC migration pending, if there is, it means that a migration will have to be run during the main thread.
        -   The list is limited by the `maxSize` property which if reached will trigger the creation of migration jobs.
        -   The background task will use the list of tenants and use the `RunMigrationsOnTenants` class to send messages to the queue for each tenant that has pending migrations. The queue workers will then pick those jobs and apply the required migrations.
        -   In this case the main request will be blocked until the migrations are completed.
        -   The system will set the status to failed if the migrations failed multiple times.
    -   **Use Case:** Suitable for applications that want to apply migrations in the background.

3.  **`MultitenantMigrationStrategy.FULL_FLEET` (Full Fleet):**

    -   **How it Works:**
        -   The `runMigrationsOnAllTenants` is called asynchronously on start of the application.
        -   It uses the advisory locks to prevent concurrent executions.
        -   It iterates through the tenants that require a migration using `listTenantsToMigrate` and sends a queue message to the `RunMigrationsOnTenants` queue using `RunMigrationsOnTenants.batchSend`, so each tenant migration is performed by the queue workers asynchronously.
    -   **Use Case:** This method is useful if you have a large fleet of tenants, and you want the migrations to be handled asynchronously.

**Code Flow**

1.  **Initial Setup:**
    -   On the application startup, depending if it is a multi-tenant environment or not, the migration strategy will be determined.
    -   Single tenant migrations are run when the server boots up using `runMigrationsOnTenant`
    -   Multi-tenant migration are run in the multi tenant database. `runMultitenantMigrations`.
    -   If the strategy is set to `PROGRESSIVE` or `FULL_FLEET`, an async migration process will be started in the background using `startAsyncMigrations`, to continue to migrate all the tenants while the application is running.
2.  **Request Processing (Multi-Tenant):**
    -   When a request is made, depending on the selected strategy, one of three possible approaches to trigger the migrations is used.
    -   If migrations are run on request, then the request processing will be blocked while migrations are being executed.
    -   If migrations are run in a `PROGRESSIVE` mode, and there is a pending `---SYNC---` migration, then the requests are also blocked until migrations are done, else they will be queued to run in background.
    -   The system uses the `tenantId` to verify if migrations have been done in the current tenant.
    -   If `FULL_FLEET` is being used, then a queue is used to migrate the tenants in background.
3.  **Database Updates:** All changes are performed to the specific tenant's database, and the migrations info on the `tenants` table are updated.
4.  **Async Process**: In the progressive case, the background process will send messages to the queue so that worker instances can pick them up.

**Key Takeaways**

-   **Single vs. Multi-Tenant:**  Single-tenant deployments use a straightforward approach, running migrations on server start. Multi-tenant deployments are more flexible, allowing migrations on each request, progressively, or in full fleet.
-   **Flexibility and Control:** Three different types of migration strategies are offered to fit different environments.
-   **Data Integrity:**  The use of version tracking helps ensure the integrity of the schema migrations.
-   **Automated Migration:** This system uses a combination of row level security and migrations to enforce the structure of the data.

This should provide a solid understanding of how migrations are handled in single and multi-tenant modes.
