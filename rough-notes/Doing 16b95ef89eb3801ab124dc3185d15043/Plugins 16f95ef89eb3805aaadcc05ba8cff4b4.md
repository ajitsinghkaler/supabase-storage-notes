# Plugins

apikey

 - If you want auth admin keys in the request heade.

migrations

- get last version of the migration for tenent in multenant or last ,igration if single tenant. If migration staratergy is onRequest. Check if mgration are uptodat wiith checking if the lastest version is completed. it is a prehandler so it runs with all requests. After that we create a migration map which is run for a specufc tenant and in the corresponding function we run migartions when migrations are complete we release the sophamaore. this is from stoppping multiple same migrations running.

[Semaphores](Plugins%2016f95ef89eb3805aaadcc05ba8cff4b4/Semaphores%2016f95ef89eb38023941acf6f3e40cb05.md)

in on pregess migrations we run all the sync migrations if there are no sync migration we add it to a migration riunning queue. after taht we add it to a migration queue where migrations are run async. If there are sync migartiosn to be run we run them at the end.

db 

We first regietr the migrations plugin. Then we add teh db property to request then we get admin and user creds for user creds we verify teh jwt and get payload. and we return a postgres connection as request.db . Before serilizng the response we use the we dispose off the connection same on timeout and request abort. we do the same for db super user but this time we pass the user as admin 

logrequest.ts

in log request we get the request start time. Log if request is aboerted or response is finished beofre writing. The. in prehandler hook We get the values of all params that are defineds  after taht we put/ before everything . atfer that we add resources and operation to the request and on response  we log the response. In do log request we make the log request a certain type and log it.

metrics

ion this we use fastifym etrics to set up metrics collection in prometheus. and here we define its config

signals

in signals we create 3 abort controllers which are aborted based on the request currest thing happening. like 

Client terminated the request before the body was fully sent

Client terminated the request before the response sent

Client aboprted request in the middel.

tenant-feature.ts

This makes it such taht a specific feature is required otherwise error is sent.

tenant-id

we set the tentnat id on the request it can be admin or normal.

storage,ts

we create storage on request. set backed as file or s3 backend. Create a storage knexdb instance and add it to the request. and on request end we close all thus

xml.ts

if request comes as xml convert it to json if response is expected as xml converts json to xml

tracing.ts

We check if tracing is enabled. We register traceServerTime which traces if Request was aborted before the server finishes to return a response we crete the appropriate trace for open telementry and convert these spans to server timings. we enrich it with more info like operation query. After taht we delet all traces. Now we get the tenant config set the tracing mode. and get the context ion which tracing is happening after that if full,logs,debug mode happens we trace the logs or stop tracing.,

signature.v4

We create a prehandler hook which extracts signature it first gets the auth header and amz credential which is made of vaiors types of keys. now check if request is multi part and if it is parse that using formdata except file key and retuern that data. This is just two ways of getting the amazon signature for s3. Now this creates the client signauture. For creating server signature we use clinet token and mix data of tenant to create server signature.If the user send keys then we get the client credentails and make s3 serversignature using taht. now if not session tojken and is not multitenant create default signature. Now using the signature we get from server verify client signature. if not verified return error. Now we create secrets verify it we verify the server token. and join required things to the jwt.