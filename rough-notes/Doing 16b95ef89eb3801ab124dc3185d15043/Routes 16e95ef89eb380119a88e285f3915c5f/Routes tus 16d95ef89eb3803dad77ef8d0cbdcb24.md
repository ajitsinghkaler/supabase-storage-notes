# Routes tus

index.ts

For tus server we create a lock notifier using postgres pubSub and subscriobe to it . After that we create an agent which gives enhanced featiures on **`http.Agent`**  and we start monitoring that agent. After that we add a hook to close the agent when server closes. Then we create a tusServer which creates a tus store if its is s3 we get s3 from tus lib and if file get file for it . and we create a tus server using various options as we need. Tus server is default tus server with a few Abort Controllers. Check file size lims naming functions, what to do on upload. now using tus server we handle authenticated public and signed routes. 

lifecycle.ts

On incoming request- Delete knex pool on finish if its signed check sign is correct or else check if allowed to upload. these are the l;ifecycle hooks which we use to make it coimpaticle with supabase. generate url we genarte a url for uploading. getFileIdFromRequest we get file id from request.