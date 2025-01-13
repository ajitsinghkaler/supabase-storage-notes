# Tenant routes

We check if headers have the admin apiKey.

/ 

On this route using multitenantKnex get all tenants and then decrypt things and send back.

/:tenantid

We geta. specific tenant using same aas above

/tenatid
We create anew tenant and run migratiosn on it. update also use patch and with knex run migration son it  but we wncypt too.