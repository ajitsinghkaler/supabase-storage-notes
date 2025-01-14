# Tracing and Metrics
Lets learn more about how tracing and metrics are collected and used within the system. From a request perspectivewe will check they are added and implemented.

## Tracing Lifecycle

Tracing is used to observe the path a request takes through the system, providing a holistic view of the processing flow, including execution times, dependencies, and errors along the way. Here’s a detailed breakdown:

1. **Request Ingress (HTTP Layer):**
   - **Instrumentation**: OpenTelemetry (OTEL) `HttpInstrumentation` automatically creates a new root span when a request comes in. The span's name is `http.request`. The span's context (traceId, spanId) can be accessed through the `trace.getActiveSpan()` function.
   - **Attributes**: The instrumentation also sets relevant attributes to the span including the tenant id (if available), method, url, status code, route, headers, and more, based on `applyCustomAttributesOnSpan`, `headersToSpanAttributes` configuration in the `HttpInstrumentation`.
   - **Logging**: The `logRequest` plugin is executed, capturing the request information in a log line.
   - **Context Propagation**: The span's context (including `traceId` and `spanId`) is attached to the current asynchronous execution context, ensuring consistent propagation to downstream operations.

2. **Authentication and Authorization (HTTP Layer):**
   - **Plugin**: The `jwt` plugin is activated.
   - **Span Creation**: The `ClassInstrumentation` plugin detects `jwt.verify` calls, creates, and ends a new span named `jwt.verify`, which represents authentication.
   - **Span Attributes**: The span has metadata such as the role that comes from the payload.

3. **Database Operations (Internal Layer):**
   - **Plugin**: The `db` plugin initializes the database connection pool and retrieves a connection.
   - **Span Creation**: The `ClassInstrumentation` plugin detects `StorageKnexDB.runQuery` calls and creates and ends a new span named `StorageKnexDB.runQuery.{{queryName}}`.
   - **Span Attributes**: The spans store the query name if present.
   - **Context Propagation**: The database query is instrumented using `@opentelemetry/instrumentation-knex`, and any calls to the database or database pool will be automatically linked to the main trace context.

4. **Business Logic (Storage Layer):**
   - **Span Creation**: When `Storage`, `ObjectStorage`, `Uploader`, etc. methods are called, `ClassInstrumentation` creates a span such as `Storage.createBucket` or `Uploader.upload`, using the `methodsToInstrument` configuration.
   - **Span Attributes**: Span's name and attributes are configured by `setName` and `setAttributes` functions when available.
   - **Chaining**: If nested calls within the same trace are made, those spans will automatically be children of the parent operation; this also happens with database calls using the knex instrumentation.

5. **Backend Interactions (Storage/Backend Layer)**
   - **Span Creation**: When a function like `S3Backend.getObject` or `S3Backend.uploadObject` is called, a new span with the name `S3Backend.getObject` or `S3Backend.uploadObject` is created by `ClassInstrumentation`, respectively.
   - **Span Attributes**: Span contains attributes such as `operation: command.constructor.name`.
   - **Context Propagation**: When an API call is made using `aws-sdk`, the request is automatically added to the OTEL context to ensure all spans are connected; this happens because of `@opentelemetry/instrumentation-aws-sdk` instrumentation.

6. **Async Operations:**
   - **Context Propagation**: If async operations are performed, and a new span is to be created, the context should be carried forward using the `trace.getTracer().startActiveSpan` function to create and automatically activate the span, so that the new span will correctly be associated with the request.
   - **Span Attributes**: It might contain attributes depending on what method is calling it.

7. **Response Phase (HTTP Layer):**
   - **Plugin**: `traceServerTime` is used to measure the server time it took to complete the response, capturing time spent in queue, database, HTTP operations, etc.
   - **Span Attributes**: If the tracing mode is set to `debug`, spans collected using the TraceCollector are serialized as JSON and added as a `stream` attribute to the main HTTP span.
   - **Span End**: All spans created using the active context, including the root `http.request` span, are ended. This finalizes the span, collects metrics, and prepares it for export.
   - **Logging**: The `logRequest` plugin logs the request, including the status code and response time, and the `serverTimes` that were captured with the `traceServerTime` plugin.

8. **Exporting Spans:**
   - **OTLP Exporter**: If configured, the OpenTelemetry Collector `BatchSpanProcessor` batches up the created spans using the `OTLPTraceExporter` and sends them to the OTEL endpoint.
     - This is done asynchronously in the background.

## Metrics Collection

Metrics provide a numerical representation of system behavior, such as request rates, duration, etc. This system exposes metrics via a Prometheus endpoint, and here's how they are collected:

1. **Request Level (HTTP Layer):**
   - **Counters**: The `fastify-metrics` plugin collects HTTP request-related metrics such as request counts, duration, and error counts. It also stores the request data into the `storage_api_http_request_duration_seconds` metrics.
   - **Labels**: Metrics collected by the `fastify-metrics` plugin include `method`, `route`, and `status_code`.
   - **Label Aggregation**: All Prometheus labels for HTTP are stored in `fastify-metrics` as tags.

2. **Database Operations (Internal Layer):**
   - **Histograms**: The `DbQueryPerformance` histogram records the time it takes for a database query to complete. It captures the time spent waiting for connection and the database query time as well as labels `region` and the method name, stored as `name`.
   - **Connections**: The metrics `DbActivePool` and `DbActiveConnection` track the pool connection counts and active connections.

3. **S3 Operations (Storage/Backend Layer):**
   - **Histograms**: The `S3UploadPart` histogram records the time it takes to upload a part of a large file to the object storage service. It has one label, `region`.

4. **Uploads:**
   - **Gauges**: `FileUploadStarted` is incremented when the file upload process starts. `FileUploadedSuccess` is incremented when an upload completes successfully.
   - **Labels**: The upload-related metrics include labels for region and upload type (standard or multipart).

5. **Queue Operations (Internal Layer):**
   - **Histograms**: `QueueJobSchedulingTime` records the time it took to schedule the message to the queue, labeled with `name`, which is usually the queue name and `region`.
   - **Gauges**: `QueueJobScheduled` for the messages scheduled to be processed by the queue, labeled with `name` and `region`, `QueueJobCompleted` for the number of completed messages, `QueueJobRetryFailed` for the number of failed retries on each message, and `QueueJobError`, which is the total count of errored messages. The labels used here are the queue names and region.

6. **HTTP Agent Metrics**
   - **Gauges**: `HttpPoolSocketsGauge` for the number of active sockets, `HttpPoolFreeSocketsGauge` for the number of free sockets, `HttpPoolPendingRequestsGauge` for the pending requests, `HttpPoolErrorGauge` for the errors, each one of them having `name`, `region`, `protocol`, and `type` as labels.

7. **Supavisor Metrics**
   - **Custom Exporter**: The Supavisor has a custom Prometheus exporter that collects information about pool sizes, tenant status, connected clients, etc. These are collected by the Prometheus config file.

## Key Takeaways:

- **End-to-End Visibility**: Tracing provides a complete view of the request lifecycle, including HTTP, DB, and file I/O operations.
- **Resource-Specific Metrics**: Metrics provide an overview of request performance with different labels.
- **Integration with OpenTelemetry**: The use of OpenTelemetry allows traces to be sent to observability backends.
- **Integration with Prometheus**: The usage of Prometheus makes it easy to collect and visualize metrics.
