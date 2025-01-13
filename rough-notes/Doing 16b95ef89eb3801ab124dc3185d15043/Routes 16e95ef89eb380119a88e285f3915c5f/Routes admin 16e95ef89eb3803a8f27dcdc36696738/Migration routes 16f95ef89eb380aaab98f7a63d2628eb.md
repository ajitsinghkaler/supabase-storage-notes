# Migration routes

migrate fleet

If queue is enabl;ed we run migrations on all tenants. Aquire a lock so that some thing does not change it in between after taht we get a list to of tenants which are not latest to 

- Finds tenants where either:
    - Migration version doesn't match current AND status is not FAILED/FAILED_STALE
    - OR migration status is null (never migrated)

now we run migrations on them as a batch. and finally we leave the lock.

prgress we check on migrations queue how long is the list and tell them as remaining.

for failed we get all that failed and return