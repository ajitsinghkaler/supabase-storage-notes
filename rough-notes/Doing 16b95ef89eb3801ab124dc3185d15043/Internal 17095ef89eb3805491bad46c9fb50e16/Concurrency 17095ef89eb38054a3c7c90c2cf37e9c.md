# Concurrency

async abort Controller

We create a promise for each lister of abort controller. Make a list of all these promises. and wehn all complete we finish them. 

In. get gorups we set a new group increate its priority by one and return thant group. Then we wait while all promises are settled 

### **Step-by-step example of AsyncAbortController usage**

1. Basic setup and abort example

```tsx
const controller = new AsyncAbortController();
const signal = controller.signal;

// Add async cleanup handler
signal.addEventListener('abort', async () => {
  await closeConnection();
  await cleanupResources();
});

// Later, trigger abort and wait for cleanup
await controller.abortAsync();
```

2. Hierarchical abort with groups

```tsx
const mainController = new AsyncAbortController();

// Child controllers with different priorities
const dbController = mainController.nextGroup;
const cacheController = mainController.nextGroup;

// Setup handlers
dbController.signal.addEventListener('abort', async () => {
  await db.disconnect();
});

cacheController.signal.addEventListener('abort', async () => {
  await cache.clear();
});

// Real-world example with database operations
async function fetchData(signal: AbortSignal) {
  const db = await connectDB();
  
  signal.addEventListener('abort', async () => {
    console.log('Cleaning up DB connection...');
    await db.close();
  });

  try {
    while (true) {
      if (signal.aborted) throw new Error('Aborted');
      await db.query('SELECT * FROM users');
      await sleep(1000);
    }
  } catch (error) {
    console.log('Operation aborted');
  }
}

// Usage
const controller = new AsyncAbortController();
fetchData(controller.signal);

// After 5 seconds, abort and wait for cleanup
setTimeout(async () => {
  await controller.abortAsync();
  console.log('All cleanup completed');
}, 5000);

```

1. Batch operations with multiple resources

```
const controller = new AsyncAbortController();

// Multiple resource handlers
['db', 'cache', 'queue'].forEach((resource) => {
  const resourceController = controller.nextGroup;
  
  resourceController.signal.addEventListener('abort', async () => {
    await cleanup(resource);
  });
});

// Gracefully abort all resources in priority order
await controller.abortAsync();
```

Key benefits:

- Ensures all cleanup operations complete
- Hierarchical abort with priorities
- Handles async cleanup operations
- Graceful shutdown of resources

mutex

In mutes we keep only one process runnig

[Semaphores](../Plugins%2016f95ef89eb3805aaadcc05ba8cff4b4/Semaphores%2016f95ef89eb38023941acf6f3e40cb05.md)