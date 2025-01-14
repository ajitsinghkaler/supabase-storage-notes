# Storage Backends

A storage backend is an abstraction layer. Instead of directly interacting with specific storage systems (like a local file system, AWS S3, Google Cloud Storage, etc.), the repository interacts with a generic interface. This interface defines a set of operations (like getObject, putObject, deleteObject, etc.) that any conforming storage backend must implement. 

There are two storage backends used in this Supabase Storage repository. They are `FileBackend` and `S3Backend`. lets learn more about them and how they interact with the file system and the database, and the key differences between them.

## 1. `FileBackend` Storage Adapter

- **Purpose:** The `FileBackend` adapter is designed to store and retrieve data using the local file system.

- **Key Features:**
  - **Local File Storage:** Files are stored directly on the server's disk, organized within a directory structure based on the tenant, bucket, key (object path), and version information.
  - **Metadata Storage:** The file metadata is stored within the extended file attributes of the file system. Extended attributes are part of the file system and are a better approach than using a metadata file since it avoids having multiple files in the same location for just the file and the metadata.
  - **Direct File Access:** Operations read and write to disk directly, without any intermediate layer or external service.
  - **Multi-Part Uploads:** Creating parts in a temporary directory and then concatenating them into a single file when the upload is complete.

- **Implementation Details:**
  - **Initialization:** The constructor fetches the storage path from the configuration or from the `FILE_STORAGE_BACKEND_PATH` env variable.
  - **`getObject` Method:**
    - It constructs the file path using the bucket name, object key, and version.
    - It reads the file from the disk using `fs.createReadStream`.
    - It retrieves metadata (content-type, cache-control) from the extended file attributes, using the `fs-xattr` package to get those values.
    - If the request is using range headers, it will stream data from disk at a specified range.
    - Returns a stream along with the file metadata.
  - **`uploadObject` Method:**
    - The system will ensure a specific folder exists using `fs-extra.ensureFile`.
    - Constructs the file path from given information.
    - Writes the incoming stream to the file system using `fs.createWriteStream`.
    - Sets metadata on the disk using the `xattr` package.
    - Returns the metadata by reading using the `headObject` method.
  - **`deleteObject` Method:**
    - Constructs the file path using the bucket name, object key, and version.
    - Uses `fs.remove` to delete the file from the file system.
  - **`copyObject` Method:**
    - Constructs source and destination file paths.
    - Uses `fs.copyFile` to copy the file and ensures the folder where the files are being copied exists.
    - It copies the metadata information from extended attributes.
  - **`deleteObjects` Method:**
    - Uses `fs.rm` to recursively remove folders and files with the given prefixes.
  - **`headObject` Method:**
    - Constructs the file path based on the request.
    - Uses `fs.stat` to retrieve file system metadata.
    - Reads file metadata using the `xattr` package.
    - Calculates the checksum using the `md5-file` package.
    - Returns relevant file metadata (size, content-type, cache-control, etag, last modified date).
  - **`createMultiPartUpload`:** Creates a folder where the file parts will be stored for multipart uploads, and a metadata file with the extra configurations.
  - **`uploadPart`:** Creates a file part where the data is saved into, and sets the Etag metadata attribute.
  - **`completeMultipartUpload`:** Once all file parts are available, it concatenates all the files together using `multistream` and calls the `uploadObject` method to finalize the upload. It also removes all the temporary file parts.
  - **`abortMultipartUpload`:** Removes the folder where the parts are stored.
  - **`uploadPartCopy`:** Copies parts from already uploaded files and stores them into a newly created temporary file that it later uses in `completeMultipartUpload`.
  - **`privateAssetUrl` Method:**
    - Returns a special local file path for internal processing purposes.
  - **Dependencies:** Uses packages like `fs-extra`, `path`, `md5-file`, `fs-xattr`, and `multistream`.

- **Interaction with the Database:**
  - The `FileBackend` adapter uses the database to retrieve tenant and bucket information. All file operations are performed on the disk and data is stored using `StorageKnexDB`.

- **Pros:**
  - **Simplicity:** It's easy to set up and reason about; it's simple to debug since everything lives on the file system.
  - **No External Dependency:** It does not rely on an external service and can be started quickly on a local environment without the need for external dependencies.
  - **Lower Latency:** File reads and writes are performed locally, avoiding extra network calls.

- **Cons:**
  - **Not Scalable:** It doesn't scale well beyond a single server.
  - **Limited Reliability:** Data is at risk if the server is damaged.
  - **Inadequate for Production:** It is not suitable for production environments if they need reliability and high availability.

## 2. `S3Backend` Storage Adapter

- **Purpose:** The `S3Backend` adapter interacts with an S3-compatible object storage service (like AWS S3, MinIO), abstracting away the specifics of the API. This is the preferred implementation if you want to achieve better scalability and high availability.
- **Key Features:**
  - **Remote Object Storage:** Uses an S3-compatible object store for the long-term storage of object data.
  - **AWS SDK:** It uses the AWS SDK for NodeJS to interact with S3 services.
  - **Authentication & Authorization:** It uses AWS SDK's authentication mechanisms based on access keys, secret keys, session tokens, and policies.
  - **Multi-Part Uploads:** It supports multi-part uploads for large files using `uploadPart` and `createMultiPartUpload` operations from the AWS SDK, and also has support for copying parts using `uploadPartCopy`.

- **Implementation Details:**
  - **Initialization:** The constructor gets configuration from the environment, such as storage bucket name, access key, secret key, region, and endpoint. The constructor also creates a custom HTTP agent and monitors it if tracing is enabled to collect HTTP socket usage information.
  - **`getObject` Method:**
    - Constructs an AWS `GetObjectCommand` using the provided bucket name and key (path).
    - Uses the AWS SDK to send this command to the S3 endpoint.
    - The response, including body and metadata, is returned. It also adds all metadata from the S3 response to the metadata object. If a range header is provided, it gets a partial response using ranged requests.
  - **`uploadObject` Method:**
    - Constructs an AWS `PutObjectCommand` with the request body and metadata.
    - If tracing is enabled, it uses the `monitorStream` function to keep track of upload speeds. If the file is larger than the allowed `uploadFileSizeLimit`, the upload will be aborted.
    - Returns the object metadata.
  - **`deleteObject` Method:**
    - Constructs a `DeleteObjectCommand` using bucket name and key.
    - Sends the request to S3 to delete the object.
  - **`copyObject` Method:**
    - Constructs a `CopyObjectCommand` using bucket name, source key, and destination key.
    - Uses the AWS SDK to send the copy command.
  - **`deleteObjects` Method:**
    - Constructs a `DeleteObjectsCommand` using bucket name and keys.
    - Sends the request to S3 to delete multiple objects at once.
    - Returns a list of deleted or failed delete operations.
  - **`headObject` Method:**
    - Constructs an AWS `HeadObjectCommand` using the provided bucket name and key.
    - Returns metadata information from the AWS S3 object.
  - **`privateAssetUrl` Method:**
    - Creates a private URL using the `getSignedUrl` function.
  - **`createMultiPartUpload`:** Creates a multi-part upload using the `CreateMultipartUploadCommand` and returns the `UploadId`.
  - **`uploadPart`:** Creates a part of the upload using `UploadPartCommand` and returns the `eTag` of the upload.
  - **`completeMultipartUpload`:** Commits a multipart upload by using `CompleteMultipartUploadCommand`, which expects the part information. If no parts are provided, it will fetch the list of parts before completing the upload.
  - **`abortMultipartUpload`:** Aborts an upload using `AbortMultipartUploadCommand`.
  - **`uploadPartCopy`:** Copies a specific byte range of an existing object and uploads it as part of a multipart upload using `UploadPartCopyCommand`.
  - **Dependencies:** Depends on the `@aws-sdk/client-s3`, `@aws-sdk/lib-storage`, `@aws-sdk/s3-request-presigner`, and `@smithy/node-http-handler` packages for interacting with S3 services, and also `@internal/http` and `@internal/streams` for stream monitoring and handling.

- **Interaction with the Database:**
  - Like the `FileBackend`, the `S3Backend` doesn't interact directly with the database; instead, it uses the `StorageKnexDB` class to manage the database operations. It only saves metadata information about the object in the database, such as size, mime-type, date of upload, etc.

- **Pros:**
  - **Scalability and High Availability:** Leverages the scalability and reliability of the S3 service.
  - **Durability and Redundancy:** Data is stored across multiple data centers for redundancy and durability.
  - **Better for Production:** Production applications should use an S3-compatible backend.

- **Cons:**
  - **Complexity:** Requires configuration with an S3 provider.
  - **Latency:** Might be prone to higher latency as every request goes through the network.

## Key Differences Summarized

| Feature           | `FileBackend`                   | `S3Backend`                             |
| ----------------- | ------------------------------- | --------------------------------------- |
| Storage Location  | Local file system               | Remote object storage (S3-compatible)   |
| Scalability       | Not scalable                    | Highly scalable                         |
| Reliability       | Limited                         | High                                     |
| Complexity        | Simpler to set up               | Requires S3 API configuration           |
| Data Handling     | Direct file I/O                 | API calls via AWS SDK                   |
| Metadata          | Extended file attributes        | S3 metadata                             |
| Ideal Use Cases   | Development, testing, local use if you are self hosting. | Production, scalable, and reliable storage |
| Authentication    | No authentication               | Supports AWS S3 authentication             |
| Instrumentation   | Limited                         | Support for custom HTTP agent and tracing|

## Key Takeaways

`FileBackend` is for local development if you do self hosting or simple scenarios, while `S3Backend` is for production usage or wherever scalability, high availability, or data durability is required. The way both adapters interact with the database is similar by using `StorageKnexDB`, as the storage layer acts as an abstraction between those adapters and the data persistence layer. The adapters just focus on reading and writing data on the storage backend.