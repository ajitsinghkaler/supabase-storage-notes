# Routes Object

First like before 3 plugins jwt, db , storage

deleteObject.ts

Create Request with schema get bucket which returns object storage which has all operations that can be performed on the object in a particular bucket and after that start a trasaction in Db find that object in dbDelete that object in db if possible then we delete the object and with a webhook we tell this to all members intrested

deletObjects.ts

Same as above. After that start mumtiple transactions where we get all the prefixes equal to the length of requesturl. then start a transaction and delete all objects from db in this batch (a batch is checked by length of uri) make a list of all files that are needed to be deleted. There is an info file if there is a version. After that delet all files using the backend(file/s3) . then send webhooks for all Deleted files

GetObjects

2 types of routes one is authenticated for privateBuckets and other is public for open buckets. On authentiocated we use the auth schema otherwise not. Then we create athe path of the bucket and find it as superuse and if not authentiocated check if it is public. If public get it as super user or as simple user. After that it gets the actual file and return it;

getSIgnedurl

CreateSchema getvarious names. Check if image transformation is allowed. If image transformation is allowed. If they are allowed convert them into the reuired whay in thich transformations are applied in imgProxy after that we get the object from db Remove all null and undefined. Then we remove base url get jwt and sign this url with the jwt and create a return path based if it has transformations or not and add the jwt to it.

getSignedUploadUrl

Create schema. Create url get the request parms. Create url path. Check if the currentuser can upload. get its jwt create a new jwt for this request . and create a new url where user can upload this url will be automatically handled by the respective route.

getSignedUrls

make schema get all paths of objects from body. then get all paths based of request length limit find all the objects concat them in a result all those which we dont have access to wont return. Create new setWith result. now for all paths create jwt and return signed paths as in getSignedUrl. Check if all are in result if they are not we dont have permission to tehm.

moveObject.ts

makeSchema get source and desti nation in buckets or spreate buckets. Check if there is a valid destinbation name and check if you ahve permission to get and update the selected object. Find the source object if source and destination are same do nothing. Copy the read file in S3 from one place into another using the adapter.ts(file or s3)  get metadata using adapter. Start a stransaction where you get the object you are moving. Find the source object in db and update it with the new destonation. delete original with admin creds once all are done send two webhooks once for deletion and one for creation. and rerturn the new move if it fails you delete the original copy that you created.

listObject.ts

first create the schema. After that get all params from body. and then with db adapter get info of all objects

updateObject.ts

first we crete schema then we only allow one file upload and fields using the fastify mutipart plugin becuase we get the raw request as we stopped encoding.  the get params from request body. Check if new objectname is valid. Get the bucket data from db test iof tehre is update permission The using uploader we upload. We will understand uploader lateron,.This gives us meta data and id using the earlier path. were we uploaded we return . path too..

getObjectinfo

authentiocated routes

We create schema with auth find bucket as superuser get params from body. If request is not authenticated return. If bucket is public get object as suoper user or as default user. and using head or info renderer get the info using the render method. Get data from db using storage and using the renderer with is either info or head (depending upon the one you choose you can get more info or head  info is more dtailed.) . there are 8 routes 4 authenticated 4 nit authenticated 4 for fastify.request one for head one for not head head is for getting info about the object and get is for getting the actual object.

CopyObject 

Copy object is just move object without deleting. When we move.

CreateObject.ts

For creatinga. object we do checks of update object. then we check if name is valid. get bucket as super user and then upload usiong the uploader aboput which we study later.

getPublicObject.ts

Same as hetObject we jsut get everything as superuser. and rerturn it 

getSignedObject.ys

we get the object taht we signed earlier. We get tenant secret. verify the token we attached to url the check isf path is same as we encoded. Get id and version of id object and get the image and return it .

uploadSignedObject.ts

First we check if the objkect is upload ed we check same multipart data as in create The verify the signature for object. Toi verify wwe check with tenanat id and if the path in jwt is same from the token we check if upserung is allowed.Then we use the the create user uploader upload and finalize it .