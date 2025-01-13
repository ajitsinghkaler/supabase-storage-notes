# s3 routes

for s3 we get keys we create new S3 keys key 32 bytes secret 64 bytes. Using the admin claims which comes from the user token of auth we store them in db  so that no one else can get them.

for getting s3 credentials we get themn directluy from db for deletiong we delet s3 credentials