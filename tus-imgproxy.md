# TUS and Imgproxy
Today we will learn at how external elements like TUS and Imgproxy are used in the Supabase Storage engine. TUS is used for resumable uploads and Imgproxy is used for image transformations. Now let's dive into the details.

## 1. TUS Protocol for Resumable Uploads

- **Purpose:** The TUS protocol is implemented to handle resumable uploads, particularly useful for large files or unreliable network connections. This ensures that uploads can be paused and resumed without losing progress.
- **Key Use Cases:**
  - **Large Files:** When a client uploads large files through TUS, the upload is broken down into smaller, manageable chunks, allowing for resumable transfers.
  - **Unreliable Networks:** It is used in the event of connectivity issues, as the upload can be continued after the network becomes available.
  - **Multipart Upload Abstraction:** For large files, TUS acts as a high-level protocol abstraction over S3's multipart uploads.
- **Lifecycle:**
  1. **Initialization (TUS Client):**
     - A TUS client, such as `tus-js-client`, is used by the client application and it initiates an upload by making a POST request to the `/upload/resumable` endpoint.
     - The POST request includes a `Upload-Length` header to specify the file's total size and additional metadata about the file.
  2. **Create Request (`/upload/resumable`, HTTP Layer):**
     - **Extraction:** The TUS middleware detects a new upload using the `Upload-Length` and `Upload-Metadata` headers in the request and calls the `namingFunction`, which does the following:
       - Parses the metadata from the header, including the `bucketName` and `objectName`.
       - Validates that the bucket and key are valid using `mustBeValidBucketName` and `mustBeValidKey`.
       - The system creates a new `UploadId` with `tenantId`, `bucketName`, `objectName`, and a `version`, which is a UUID generated on every request.
     - The new `UploadId` is encoded into a base64url string and is added to the response `Location` header to create the unique upload ID URL.
     - **Logging:** The TUS server logs when the upload begins, as well as the request headers and parameters using Pino.
     - **Security:** The request also checks if the user has the authorization to create this object and the request is authenticated.
  3. **Pre-Upload Checks (`onCreate` hook):**
     - The `onCreate` hook (in `src/http/routes/tus/lifecycle.ts`) is called before the upload happens; it performs the following:
       - The TUS upload data is saved into the file system or S3.
       - Validates the content type if available.
     - If the max file size exceeds the configured value, then it aborts the upload.
     - The `onCreate` hook returns the headers for the responses.
  4. **Uploading Parts (PUT/PATCH `/upload/resumable/{uploadId}`, TUS Layer):**
     - **Request Check:** Each individual PUT or PATCH request is checked against the `onIncomingRequest` middleware, which validates if:
       - The request is for a signed upload URL.
       - The request has valid authentication.
       - Validates the file size limit using the headers `Upload-Offset`, `Upload-Length`, or by checking the `content-length` header, also checking the configuration or tenant limit.
     - **Data Handling:** The `TUS` middleware (in `@tus/server`) handles each PUT/PATCH request to upload a part or a chunk of the file.
     - **Data Storage:**
       - If using `S3Store`, parts are uploaded to the S3 compatible object storage.
         - The `UploadPartCommand` is used to send upload part requests.
         - **Tracing:** Each of the calls in the `S3Store` has a trace using `ClassInstrumentation`.
       - If using `FileStore`, parts are stored locally in the server's file system.
       - **Metrics:** The `S3UploadPart` histogram is used to record the time taken to complete the upload process.
     - **Metadata Update:** If the used adapter is S3 storage, the `updateMultipartUploadProgress` is used in the database to persist information about the multipart upload state.
       - The `uploadPartCopy` is used when a copy operation is done, and the metadata attributes are used to ensure the file was properly copied.
     - **Concurrency Control:** To avoid concurrency issues during multi-part uploads, a mutex is created by the `PgLocker` class to handle the concurrent uploads and avoid race conditions.
       - When the lock is created, a `pg_try_advisory_xact_lock` is created using Postgres, which allows the backend to block and wait if a particular ID is being uploaded.
       - The advisory lock ensures that concurrent uploads to the same file are done one at a time.
     - **Error Handling:** If any errors occur during part uploads or on the mutex locking, appropriate errors are handled, and a 4xx response is sent back; the error is also logged.
  5. **Upload Completion (PUT `/upload/resumable/{uploadId}`, TUS Layer):**
     - Once all parts of the file have been uploaded, the client sends a final PUT request which triggers the `onUploadFinish` hook.
     - **Object Creation:** The `completeMultipartUpload` from the storage backend is called to assemble the parts into the final object in the file system or S3.
     - **Metadata Storage:** The system saves the file metadata into the `storage.objects` table.
     - **Tracing:** A new span called `Storage.completeUpload` is created by the `ClassInstrumentation` plugin.
     - **Webhook Trigger:** The `ObjectCreated` event is fired using the queue.
  6. **Error Handling:**
     - If there is any failure during any phase of the upload, the server will return an error to the client.
     - If a fatal error happens, the upload will be cleaned up using the `ObjectAdminDelete` event.
  7. **Cleanup:** Once the upload is completed (either finished or aborted), temporary data or the upload folder will be cleaned up.
- **Asynchronous Nature:** The TUS protocol is asynchronous and allows the server to handle large uploads concurrently. When an S3 upload finishes, the system relies on a background task to clean up any remaining temporary files.

## 2. Imgproxy for Image Transformations

- **Purpose:** Imgproxy is an external service used for on-demand image processing and transformations. It allows the Supabase Storage system to serve resized, cropped, and optimized images without requiring pre-generated versions of each object.
- **Key Use Cases:**
  - **Dynamic Image Resizing:** When an image is requested with specified width and height parameters, imgproxy applies these transformations.
  - **Image Format Conversion:** Imgproxy is used to convert images to optimized formats such as WebP or AVIF.
  - **Watermarks and Other Effects:** Can handle other image transformations such as watermarks.
- **Lifecycle:**
  1. **Image Request (HTTP Layer):**
     - A client requests a particular image via the `/render/image` or a signed URL from the `/render/sign` endpoint with a variety of transformations specified via URL parameters.
     - **Request Parameters:** Query parameters such as `width`, `height`, and `resize` are parsed from the URL string by the `ImageRenderer` class.
  2. **Private Object URL Creation (Storage Layer):**
     - The system calls the S3 storage backend to generate a signed presigned URL to allow `imgproxy` to read from the bucket.
     - **Tracing:** A new span named `S3Backend.privateAssetUrl` is created using the `ClassInstrumentation` plugin.
     - **Performance:** If there is a local file, this path is returned instead of the S3 one.
  3. **Imgproxy Request (HTTP Layer):**
     - The system constructs a URL to the `imgproxy` service based on transformation parameters. The URL includes parameters to configure imgproxy to perform operations such as resizing, cropping, and format conversion, as well as the signed internal URL to fetch the original image data.
     - The HTTP client `Axios` is used to call `imgproxy` with the constructed URL.
     - **Tracing:** A new span named `axios.get` is created with all the requested parameters.
     - **Request Timeout:** The connection is configured to have a timeout to avoid long-running calls and also to make use of connection pooling.
     - **Error Handling:** In case of an error on the response, the body response is streamed and added as an error message.
  4. **Image Transformation (Imgproxy Service):**
     - Imgproxy, using the signed URL, fetches the object and applies the requested transformations, storing it in memory before streaming it back.
  5. **Response Streaming (HTTP Layer):**
     - The `imgproxy` response stream (with the transformed image) is streamed back to the client. If a response is not successful, an error is raised.
     - **Caching:** The caching headers such as `cache-control` and `etag` are extracted from the original request or the backend.
     - **Tracing:** The `axios.get` span will be closed, either with a status of OK or with an error.
- **Asynchronous Nature:** The integration with `imgproxy` involves making an asynchronous network call, but the use of HTTP streams makes the process non-blocking.

## Usage Details:

- **TUS Workflow:**
  1. A client sends a POST request to `/upload/resumable` to initiate a new upload with the total file size.
  2. The server creates a TUS upload resource and responds with a unique upload ID (URL).
  3. The client makes multiple PUT or PATCH requests to the `Location` URL returned in step 2, uploading the file in smaller parts.
  4. The client makes a final PUT request to the `Location` URL; the server then stitches the parts into one final object in S3 or on disk.

- **Imgproxy Workflow:**
  1. A client requests an image through the `/render/image` endpoint, specifying transformation parameters in the query string.
  2. The server generates a signed URL for access to the original image in object storage using `privateAssetUrl`.
  3. The server calls `imgproxy` with the transformation options and the signed URL.
  4. Imgproxy processes the image, and the server returns the processed image via a stream to the client.

## Key Takeaways:

- **TUS for Resumable Uploads:** This feature enhances file upload resilience and efficiency. It provides support for uploading big files and for unreliable connections.
- **Imgproxy for Dynamic Transformations:** This service enables flexible image transformations without altering original files and without the need for pre-generated files, saving costs and reducing complexity.
- **Independent Services:** Both TUS and Imgproxy are implemented in a way that these services are external and not a core dependency for the application.
- **Streaming Data:** Streams are used to process large files efficiently and to have non-blocking I/O operations.
- **Tracing:** Both of these steps are instrumented with OpenTelemetry and can be visualized by using a compatible backend.
