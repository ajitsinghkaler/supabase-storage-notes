# Routes buckets

Why is migrations a prehandler hook

Learn about migration startergy for multiTenant apps

jwt.ts

do learn about getJwtSecret verifyJWT from jwt.ts

First there is a jwt plugin which runs before all requests because of the fastify.addPlugin(preHandler)

We also have to duse decorateRequest because request is basic and we want to add additional elemnets to it. Fastify module is the default way to extend fastify

db.ts(later)

This first add a migrations fastify plugin. Which is also a prehandler hook. Why is 

storage.ts(Study more in detail)

Two types of storage backend type - file, S3

Then it creates that type of a backend - file or s3

The add storage key to request and we create a new StorageKnexDb for each request using the prehandler hook, because of multiTenancy request scoped transactions. With connection from last db.ts, the tenant id and the latest migration., Then we attach this backend and a new storaghe instance to the request

Createbucket.ts

Fist we create the required schemas. then we do a fastify post request. Where define the operation type for logging. the define which resources are we chaning so that we can check access control. Then we get the owner which we get from jwt.ts. Destructure the body to get everything for creating a bucket. Then from the storage instance which we added in storage.ts add the bucket is created. This only allow s3 compatible bucket names and there should be a name. Check if file size limit is set lest thamn the global tentant file size limit and check if valid file size is given Set null if not given. Check if valid mime type. Then using the db intance created in last time create new bucket.

EmptyBucket.ts

Emopty bucket same as above create schema, get id. To empty a bucket find it’ name get list of all objects based on requestUrlLength. Then delete all these from Db.then each file is deleted from S3 or file based system based on the adapter used in storage.ts we store two files one id the file one is info file which store info about that object. If some files are not deleted we say you dont have permission to delete them

List buckets.ts

Create schema - List all buckets from db. with the given colums.

getBucket.ts

Create schema get a specific bucket from Db.

updateBucket.ts

Create Schema. Check all like create Bucket and update the data

deleteBucket.ts

Create a schema. Do it in a Db transaction find the bucket if its empty delete it if not delet there is no bucket like this