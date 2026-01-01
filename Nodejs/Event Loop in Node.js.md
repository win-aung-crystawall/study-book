# Event Loop in Node.js

## Introduction

The Event Loop is the core mechanism that enables Node.js to perform non-blocking I/O operations despite JavaScript being single-threaded. It manages the execution of asynchronous operations, callbacks, and handles the order in which different types of tasks are processed.

## Event Loop Phases

The Node.js event loop operates in several distinct phases, each with its own queue of callbacks to execute:

```
┌───────────────────────────┐
┌─>│           timers          │ ← setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │ ← I/O callbacks (deferred)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │ ← Internal use
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │ ← Fetch new I/O events
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │ ← setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │ ← socket.on('close', ...)
   └───────────────────────────┘
```

## Execution Order Overview

The event loop processes tasks in the following priority order:

1. **Microtasks** (highest priority)
   - `process.nextTick()` callbacks
   - Promise callbacks (`.then()`, `.catch()`, `.finally()`)
   - `queueMicrotask()` callbacks

2. **Macrotasks** (processed in phases)
   - **Timers Phase**: `setTimeout()`, `setInterval()`
   - **Pending Callbacks Phase**: I/O callbacks deferred to next iteration
   - **Poll Phase**: Fetch new I/O events; execute I/O callbacks
   - **Check Phase**: `setImmediate()` callbacks
   - **Close Callbacks Phase**: Close event callbacks (e.g., `socket.on('close')`)

## Microtasks

Microtasks are executed immediately after the current operation completes, before any other macrotask. They have the highest priority in the event loop.

### Types of Microtasks

#### 1. `process.nextTick()` (Highest Priority)

`process.nextTick()` has the highest priority among all microtasks. It runs before any Promise callbacks.

```javascript
console.log('Start');

Promise.resolve().then(() => console.log('Promise 1'));
process.nextTick(() => console.log('nextTick 1'));
Promise.resolve().then(() => console.log('Promise 2'));
process.nextTick(() => console.log('nextTick 2'));

console.log('End');

// Output:
// Start
// End
// nextTick 1
// nextTick 2
// Promise 1
// Promise 2
```

#### 2. Promise Callbacks

Promise callbacks (`.then()`, `.catch()`, `.finally()`) are microtasks that execute after `process.nextTick()` but before macrotasks.

```javascript
console.log('Start');

setTimeout(() => console.log('setTimeout'), 0);
Promise.resolve().then(() => console.log('Promise'));
process.nextTick(() => console.log('nextTick'));

console.log('End');

// Output:
// Start
// End
// nextTick
// Promise
// setTimeout
```

#### 3. `queueMicrotask()`

`queueMicrotask()` schedules a function to run as a microtask, similar to Promise callbacks.

```javascript
console.log('Start');

queueMicrotask(() => console.log('queueMicrotask'));
Promise.resolve().then(() => console.log('Promise'));

console.log('End');

// Output:
// Start
// End
// queueMicrotask
// Promise
```

### Microtask Execution Order

Within microtasks, the order is:
1. All `process.nextTick()` callbacks (in order)
2. All Promise callbacks and `queueMicrotask()` callbacks (in order)

```javascript
console.log('1. Start');

process.nextTick(() => console.log('2. nextTick 1'));
Promise.resolve().then(() => console.log('4. Promise 1'));
queueMicrotask(() => console.log('5. queueMicrotask 1'));
process.nextTick(() => console.log('3. nextTick 2'));
Promise.resolve().then(() => console.log('6. Promise 2'));

console.log('7. End');

// Output:
// 1. Start
// 7. End
// 2. nextTick 1
// 3. nextTick 2
// 4. Promise 1
// 5. queueMicrotask 1
// 6. Promise 2
```

## Macrotasks

Macrotasks are processed in phases during each event loop iteration. Each phase has a queue of callbacks to execute.

### 1. Timers Phase

Executes callbacks scheduled by `setTimeout()` and `setInterval()`.

```javascript
console.log('Start');

setTimeout(() => console.log('setTimeout 1'), 0);
setTimeout(() => console.log('setTimeout 2'), 0);

console.log('End');

// Output:
// Start
// End
// setTimeout 1
// setTimeout 2
```

**Note**: The timer callbacks are executed only if the minimum delay has elapsed. A `setTimeout(callback, 0)` doesn't guarantee immediate execution; it schedules the callback to run in the next timers phase.

### 2. Pending Callbacks Phase

Executes I/O callbacks deferred to the next loop iteration. These are typically system-level callbacks that were deferred.

### 3. Poll Phase

The poll phase has two main functions:
1. Calculate how long it should block and wait for I/O
2. Process events in the poll queue

```javascript
const fs = require('fs');

console.log('Start');

fs.readFile(__filename, () => {
  console.log('I/O callback');
  
  setTimeout(() => console.log('setTimeout in I/O'), 0);
  setImmediate(() => console.log('setImmediate in I/O'));
});

console.log('End');

// Output:
// Start
// End
// I/O callback
// setImmediate in I/O
// setTimeout in I/O
```

**Key Point**: When inside an I/O callback, `setImmediate()` always executes before `setTimeout()` because the poll phase comes before the timers phase in the next iteration.

### 4. Check Phase (setImmediate)

Executes callbacks scheduled by `setImmediate()`. This phase runs after the poll phase completes.

```javascript
console.log('Start');

setImmediate(() => console.log('setImmediate 1'));
setImmediate(() => console.log('setImmediate 2'));

console.log('End');

// Output:
// Start
// End
// setImmediate 1
// setImmediate 2
```

### 5. Close Callbacks Phase

Executes close callbacks (e.g., `socket.on('close', ...)`).

```javascript
const net = require('net');

const server = net.createServer((socket) => {
  socket.on('close', () => {
    console.log('Socket closed');
  });
  
  socket.end();
});

server.listen(0, () => {
  const client = net.createConnection(server.address().port);
  client.on('close', () => {
    console.log('Client closed');
    server.close();
  });
});
```

## Complete Execution Order Example

Here's a comprehensive example showing the complete execution order:

```javascript
console.log('=== Start ===');

// Microtasks
process.nextTick(() => console.log('1. nextTick'));
Promise.resolve().then(() => console.log('2. Promise'));
queueMicrotask(() => console.log('3. queueMicrotask'));

// Macrotasks
setTimeout(() => console.log('4. setTimeout'), 0);
setImmediate(() => console.log('5. setImmediate'));

// I/O operation
const fs = require('fs');
fs.readFile(__filename, () => {
  console.log('6. I/O callback');
  
  // Inside I/O callback
  process.nextTick(() => console.log('7. nextTick in I/O'));
  Promise.resolve().then(() => console.log('8. Promise in I/O'));
  setTimeout(() => console.log('9. setTimeout in I/O'), 0);
  setImmediate(() => console.log('10. setImmediate in I/O'));
});

console.log('=== End ===');

// Output:
// === Start ===
// === End ===
// 1. nextTick
// 2. Promise
// 3. queueMicrotask
// 4. setTimeout
// 5. setImmediate
// 6. I/O callback
// 7. nextTick in I/O
// 8. Promise in I/O
// 10. setImmediate in I/O
// 9. setTimeout in I/O
```

## I/O Callbacks

I/O callbacks are executed during the poll phase. When an I/O operation completes, its callback is added to the poll queue.

### File System I/O

```javascript
const fs = require('fs');

console.log('Start');

fs.readFile('file.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('Error:', err);
    return;
  }
  console.log('File read complete');
});

console.log('End');
// File reading happens asynchronously
```

### Network I/O

```javascript
const http = require('http');

console.log('Start');

http.get('http://example.com', (res) => {
  console.log('HTTP response received');
  res.on('data', () => {});
  res.on('end', () => {
    console.log('Response ended');
  });
});

console.log('End');
```

## CPU Intensive Tasks

CPU-intensive tasks can block the event loop, preventing other operations from executing. This is a critical consideration in Node.js.

### The Problem

```javascript
console.log('Start');

// This blocks the event loop
function cpuIntensiveTask() {
  const start = Date.now();
  while (Date.now() - start < 5000) {
    // Simulate CPU-intensive work
  }
  console.log('CPU task complete');
}

cpuIntensiveTask();

setTimeout(() => {
  console.log('This will be delayed');
}, 0);

console.log('End');

// Output:
// Start
// (5 second delay)
// CPU task complete
// End
// This will be delayed (after the blocking task)
```

### Solutions for CPU-Intensive Tasks

#### 1. Use Worker Threads

```javascript
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
  console.log('Main thread: Start');
  
  const worker = new Worker(__filename);
  
  worker.on('message', (result) => {
    console.log('Main thread: Received result', result);
  });
  
  setTimeout(() => {
    console.log('Main thread: Non-blocking operation');
  }, 100);
  
  console.log('Main thread: End');
} else {
  // Worker thread
  function cpuIntensiveTask() {
    const start = Date.now();
    while (Date.now() - start < 2000) {
      // CPU-intensive work
    }
    return 'Task complete';
  }
  
  const result = cpuIntensiveTask();
  parentPort.postMessage(result);
}
```

#### 2. Break Up Work with setImmediate

```javascript
function processChunk(data, index, callback) {
  if (index >= data.length) {
    callback();
    return;
  }
  
  // Process one chunk
  processItem(data[index]);
  
  // Yield to event loop, then continue
  setImmediate(() => {
    processChunk(data, index + 1, callback);
  });
}

const largeArray = new Array(1000000).fill(0).map((_, i) => i);
processChunk(largeArray, 0, () => {
  console.log('All chunks processed');
});
```

#### 3. Use process.nextTick for Recursive Operations

```javascript
function processAsync(items, callback) {
  if (items.length === 0) {
    callback();
    return;
  }
  
  const item = items.shift();
  processItem(item);
  
  // Yield to event loop
  process.nextTick(() => {
    processAsync(items, callback);
  });
}
```

#### 4. Use Clustering for Multiple Processes

```javascript
const cluster = require('cluster');
const os = require('os');

if (cluster.isMaster) {
  const numWorkers = os.cpus().length;
  
  for (let i = 0; i < numWorkers; i++) {
    cluster.fork();
  }
  
  cluster.on('exit', (worker) => {
    console.log(`Worker ${worker.process.pid} died`);
    cluster.fork(); // Restart worker
  });
} else {
  // Worker process
  const http = require('http');
  
  http.createServer((req, res) => {
    // Handle request (can be CPU-intensive)
    res.end('Hello from worker ' + process.pid);
  }).listen(8000);
}
```

## Best Practices

### 1. Avoid Blocking the Event Loop

```javascript
// ❌ Bad: Blocks event loop
function badExample() {
  const start = Date.now();
  while (Date.now() - start < 1000) {
    // Blocking operation
  }
}

// ✅ Good: Non-blocking
function goodExample() {
  setTimeout(() => {
    // Non-blocking operation
  }, 1000);
}
```

### 2. Use setImmediate for Deferred Execution

```javascript
// ✅ Use setImmediate when you want to execute after I/O
fs.readFile('file.txt', () => {
  setImmediate(() => {
    console.log('This runs after I/O callbacks');
  });
});
```

### 3. Use process.nextTick Sparingly

```javascript
// ⚠️ Use process.nextTick carefully - it can starve the event loop
function recursiveNextTick() {
  process.nextTick(recursiveNextTick);
  // This will prevent other callbacks from running!
}

// ✅ Better: Use setImmediate for recursion
function recursiveSetImmediate() {
  setImmediate(recursiveSetImmediate);
  // This allows other phases to run
}
```

### 4. Monitor Event Loop Lag

```javascript
function monitorEventLoop() {
  const start = process.hrtime.bigint();
  
  setImmediate(() => {
    const delta = process.hrtime.bigint() - start;
    const nanosec = Number(delta);
    const millisec = nanosec / 1000000;
    
    if (millisec > 10) {
      console.warn(`Event loop lag: ${millisec.toFixed(2)}ms`);
    }
    
    monitorEventLoop();
  });
}

monitorEventLoop();
```

## Common Patterns and Gotchas

### Pattern 1: setTimeout vs setImmediate

```javascript
// Outside I/O callback
setTimeout(() => console.log('setTimeout'), 0);
setImmediate(() => console.log('setImmediate'));
// Order is non-deterministic (depends on performance)

// Inside I/O callback
fs.readFile(__filename, () => {
  setTimeout(() => console.log('setTimeout'), 0);
  setImmediate(() => console.log('setImmediate'));
  // setImmediate always runs first
});
```

### Pattern 2: Microtask Starvation

```javascript
// ⚠️ This can starve the event loop
function starveLoop() {
  Promise.resolve().then(() => {
    starveLoop();
  });
}

// ✅ Better: Use setImmediate
function betterApproach() {
  setImmediate(() => {
    betterApproach();
  });
}
```

### Pattern 3: Nested Async Operations

```javascript
console.log('Start');

Promise.resolve().then(() => {
  console.log('Promise 1');
  
  Promise.resolve().then(() => {
    console.log('Promise 2 (nested)');
  });
  
  process.nextTick(() => {
    console.log('nextTick (nested)');
  });
});

console.log('End');

// Output:
// Start
// End
// Promise 1
// nextTick (nested)
// Promise 2 (nested)
```

## Summary

### Execution Priority (Highest to Lowest)

1. **Synchronous code** (runs immediately)
2. **Microtasks**:
   - `process.nextTick()` callbacks
   - Promise callbacks (`.then()`, `.catch()`, `.finally()`)
   - `queueMicrotask()` callbacks
3. **Macrotasks** (in phase order):
   - Timers phase: `setTimeout()`, `setInterval()`
   - Pending callbacks phase: Deferred I/O callbacks
   - Poll phase: I/O callbacks
   - Check phase: `setImmediate()` callbacks
   - Close callbacks phase: Close event callbacks

### Key Takeaways

- **Microtasks** run before **macrotasks**
- `process.nextTick()` has the highest priority among microtasks
- Inside I/O callbacks, `setImmediate()` runs before `setTimeout()`
- CPU-intensive tasks block the event loop - use Worker Threads or break up work
- Always yield to the event loop for long-running operations
- Monitor event loop lag in production applications

## Additional Resources

- [Node.js Event Loop Documentation](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- [libuv Documentation](http://docs.libuv.org/en/v1.x/design.html)
- [Understanding the Node.js Event Loop](https://blog.insiderattack.net/event-loop-and-the-big-picture-nodejs-event-loop-part-1-1cb67a182810)

