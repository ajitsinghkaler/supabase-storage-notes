# Migration

progressiveMigrations

In create jobs we get the maxtenants that can be done according to max jobs taht can be run. Then we map each of these tenant to its migrations whose migratiosn are not upto date and not fulfilled.  and add them to the queue. In prgressgive we do this after each inbterval,. Sor startinbg thes e kjigrations we start the job. With draion we run all jobs in the queue. In add tenant we add tent ofn already not in queue,.

migrate.ts

Load and memoise migrations. After that dependin g on the stratergy run migration. You can run migrations on request you can run tehm progressivelyor run them on all. If status is not there check if there are any sycn migratiosn using if there is sync token in them ---SYNC---. Or run the check on last migrations To run migrations on all yopu get the lock chheck it and run migrations of all tenants batch wise and log it .For running multitenant mogrations you get Db config connect it get lock and run migrations on it. For running m igration on single we do all the same with gicving it a single tenant. Fpr running migrations we check if table exist if it exists we get all the migrations from the migration table. And we refresh migration position as in the folliwng example 

```
// Initial state example:
const appliedMigrations = [
  { id: 1, name: "create_users", hash: "abc" },
  { id: 2, name: "add_email", hash: "def" }
];

const intendedMigrations = [
  { id: 1, name: "create_users", hash: "abc" },
  { id: 1.5, name: "add_username", hash: "xyz" }, // New migration to backport
  { id: 2, name: "add_email", hash: "def" }
];

const backportMigrations = [
  { index: 1, from: "add_email" } // Insert at index 1, replacing "add_email"
];
```

```tsx
// 1. Create a copy of existing migrations
let newMigrations = [...appliedMigrations]
// newMigrations = [
//   { id: 1, name: "create_users", hash: "abc" },
//   { id: 2, name: "add_email", hash: "def" }
// ]

// 2. Flag to track if we need to update
let shouldUpdateMigrations = false

// 3. Process each backport request
backportMigrations.forEach((migration) => {
  // 4. Get the migration at the insertion point
  const existingMigration = newMigrations?.[migration.index]
  // existingMigration = { id: 2, name: "add_email", hash: "def" }

  // 5. Verify we can backport at this position
  if (!existingMigration || existingMigration.name !== migration.from) {
    return
  }

  // 6. Keep all migrations before the insertion point
  const migrations = newMigrations.slice(0, migration.index)
  // migrations = [{ id: 1, name: "create_users", hash: "abc" }]

  // 7. Insert the new migration
  migrations.push(intendedMigrations[migration.index])
  // migrations = [
  //   { id: 1, name: "create_users", hash: "abc" },
  //   { id: 1.5, name: "add_username", hash: "xyz" }
  // ]

  // 8. Update IDs of subsequent migrations
  const afterMigration = newMigrations.slice(migration.index).map((m) => {
    m.id = m.id + 1     // Increment IDs to make room
    m.hash = intendedMigrations[m.id].hash  // Update hash
    return m
  })
  // afterMigration = [{ id: 3, name: "add_email", hash: "def" }]

  // 9. Combine everything
  migrations.push(...afterMigration)
  newMigrations = migrations
  // Final newMigrations = [
  //   { id: 1, name: "create_users", hash: "abc" },
  //   { id: 1.5, name: "add_username", hash: "xyz" },
  //   { id: 3, name: "add_email", hash: "def" }
  // ]

  shouldUpdateMigrations = true
})
```

After this there are some migrations to update we update the migrations after running a transaction. If the table does not exist we create the schema After taht we validate if intenetd and applied migrations are same. If they are not same we update there hashes get all that mismatch and redefine there hash. after thsi we filter what migrations have not been applied. After this we apply thoise migrations.