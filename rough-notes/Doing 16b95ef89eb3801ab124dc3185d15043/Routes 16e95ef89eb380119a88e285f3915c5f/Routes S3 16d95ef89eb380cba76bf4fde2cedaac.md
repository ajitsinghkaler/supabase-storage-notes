# Routes S3

index.ts

Check if s3 protocol is enabled from env. If not return. 
now lets build the paths for S3 request. We create a s3 router. From taht router get routes which is a map of routes for S3 . not feom taht map we make an array of its keys now for each type we will add its methids. For taht we get the routes methods and for each methiod we get routes. tehn using an inner plugin we again check if s3 is ennabled for this tenant. We check if any method needs to disable content parsing and if it needs it we disable it Then we add a json to XMl plugin we register a new plugin taht parses from json to xml in this in preserialization when rewuwst is serilized we cange it from json to x. After that we add signature v4 , db, storage plugins. After taht for each mothod we refister the route path and teh route handler with and without trailing / disable default fastify validator and error handler as s3errorHandler. How route handler works.  

```tsx
 /**
     * The s3Routes is a Map with the following structure:
     *
     * Map {
     *  '/:bucketName/*' => [
     *   {
     *      method: 'GET',
     *      schema: { ... },
     *      handler: [Function],
     *      compiledSchema: [Function],
     *      operation: 'GET_OBJECT',
     *      disableContentTypeParser: false
     *  },
     * {
     *      method: 'DELETE',
     *      schema: { ... },
     *      handler: [Function],
     *      compiledSchema: [Function],
     *      operation: 'DELETE_OBJECT',
     *      disableContentTypeParser: false
     *  }
     * ]
     * }
     * {
  '/buckets': [
    { method: 'GET', handler: listBuckets },
    { method: 'POST', handler: createBucket }
  ],
  '/buckets/{bucket}/objects': [
    { method: 'GET', handler: listObjects },
     { method: 'POST', handler: createObject }
      { method: 'DELETE', handler: deleteObjects }
       { method: 'PUT', handler: copyObject }
        { method: 'POST', handler: moveObject }
    { method: 'PUT', handler: putObject }
  ]
}
     * 
     */

    Array.from(s3Routes.keys()).forEach((routePath) => {
      // routePath = '/:bucketName/*'
      // Get all the routes for the given routePath
      const routes = s3Routes.get(routePath)

      if (!routes || routes?.length === 0) {
        return
      }

      // eslint-disable-next-line @typescript-eslint/ban-ts-comment
      // @ts-ignore
      const methods = new Set(routes.map((e) => e.method))
      // methods = Set { 'GET', 'DELETE' }
      methods.forEach((method) => {
        // method = 'GET'

        // routesByMethod = [
        //   {
        //     method: 'GET',
        //     schema: { ... },
        //     handler: [Function],
        //     compiledSchema: [Function],
        //     operation: 'GET_OBJECT',
        //     disableContentTypeParser: false
        //   }
        // ]
        // There may be multiple routes for the same method like get object and get object info

        const routesByMethod = routes.filter((e) => e.method === method)

        const routeHandler: RouteHandlerMethod = async (req, reply) => {
          // route  = All the routes for the given method
          // {
          //     method: 'GET',
          //     schema: { ... },
          //     handler: [Function],
          //     compiledSchema: [Function],
          //     operation: 'GET_OBJECT',
          //     disableContentTypeParser: false
          //   }
          for (const route of routesByMethod) {
            // route = {
            //     method: 'GET',
            //     schema: { ... },
            //     handler: [Function],
            //     compiledSchema: [Function],
            //     operation: 'GET_OBJECT',
            //     disableContentTypeParser: false
            //   }

```

error-handler.ts

Here we check which tyope of error is happeing and return accordin to it. Like validation, S3ServiceException, DatabaseError, StorageBackendError, 

router.ts

First create a list of all s3 commands that are allowed. Then with get router we pass each of these commands or s3 router instance and return that router.Create a map of routes. Then we create ajv whioch validates json and may change it. to register a route we get its header and params. Clean it get its options related to it. Then we create a schema to comple and whater is in it we add from schema like headers query body. Check which keys of schema a re required if schema ofject has required property we return that property.Then we create a new ajv which will validate json. Then after taht we create a new route passing everything and the created schema. If it is already in map we push the route other wise create a new key. after taht we have various types of registering methods get post put delete. parse query info which gives params 

For example, a string like `"path?name=john&age=25|Content-Type=json"` would be parsed into:

- Queries: `[{key: "name", value: "john"}, {key: "age", value: "25"}]`
- Headers: `["Content-Type=json"]`

in match route if there is header we match both header and params if not onky params,. 
For parsm match each param or match it has *. Find path in array schemas return all paths in schema 

```tsx
const schema = {
  type: 'object',
  properties: {
    users: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          addresses: { type: 'array' }
        }
      }
    }
  }
}
findArrayPathsInSchemas([schema]) // Returns: ['users', 'users.addresses']
```

For all commands when we create a router we get the list of commands and run it on the router and that is when commands creates the _routes and commands have no role after taht.