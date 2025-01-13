# Routes Render

These routes are only enabled if imageTransformation is enabled on tenant. This uses a helper `reuiretenantFeature` which is a fastify plugin and returns error if not enabled. 

Then after taht if rate limiter is enabled we register the rateLimiter plugin. The rate limiter plugin puts rate limits on these

renderAuthenticatedimage.ts

First Create the schema. then get params from body. Find the object on which transformations done or rendering needs to be done. After that we get the image renderer. If it is multitenant set tenant set tenant maxresolution. Set transformations on the renderer and render the image to get transformations.

renderSignedImage.ts(no auth)

Create the schema. then get token and verify it get the payload get transformations url and expiry date from token get if they are not give error get bucketname from path . Now get the object with storage. now everyThing is same as above.

renderPublicImage.ts

Everything same as render authImage but get evrything as superuser.