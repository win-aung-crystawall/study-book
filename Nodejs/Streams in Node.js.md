# Streams in Node.js

## Introduction

Streams are one of the fundamental concepts in Node.js that enable handling data in a continuous, efficient manner. Instead of loading entire files or datasets into memory, streams allow you to process data piece by piece (in chunks), making them ideal for working with large files, network operations, and real-time data processing.

## Why Streams?

### Memory Efficiency

```javascript
// ❌ Without streams: Load entire file into memory
const fs = require('fs');
fs.readFile('large-file.txt', (err, data) => {
  // Entire file is in memory (could be GB!)
  console.log(data.toString());
});

// ✅ With streams: Process data in chunks
const stream = fs.createReadStream('large-file.txt');
stream.on('data', (chunk) => {
  // Process small chunks (default 64KB)
  console.log('Chunk:', chunk.length, 'bytes');
});
```

### Time Efficiency

Streams allow you to start processing data as soon as it's available, rather than waiting for the entire dataset to be loaded.

```javascript
// Start processing immediately, even if file is 10GB
const stream = fs.createReadStream('huge-file.txt');
stream.on('data', (chunk) => {
  processChunk(chunk); // Process while still reading
});
```

## Types of Streams

Node.js provides four fundamental stream types:

1. **Readable** - Data can be read from (e.g., `fs.createReadStream()`)
2. **Writable** - Data can be written to (e.g., `fs.createWriteStream()`)
3. **Duplex** - Both readable and writable (e.g., TCP sockets)
4. **Transform** - A duplex stream that can modify data as it passes through (e.g., compression)

## Readable Streams

Readable streams represent a source of data that can be consumed.

### Creating Readable Streams

```javascript
const fs = require('fs');
const { Readable } = require('stream');

// Method 1: Using fs.createReadStream()
const fileStream = fs.createReadStream('file.txt');

// Method 2: Creating custom readable stream
const readable = new Readable({
  read() {
    // Called when stream needs more data
  }
});
```

### Reading Modes

Readable streams operate in two modes:

#### 1. Flowing Mode

Data is automatically read and provided to your application as fast as possible.

```javascript
const fs = require('fs');
const stream = fs.createReadStream('file.txt');

// Flowing mode - data is pushed automatically
stream.on('data', (chunk) => {
  console.log('Received chunk:', chunk.length, 'bytes');
  console.log(chunk.toString());
});

stream.on('end', () => {
  console.log('Stream ended');
});
```

#### 2. Paused Mode

Stream starts in paused mode. You must explicitly call `read()` to get chunks.

```javascript
const fs = require('fs');
const stream = fs.createReadStream('file.txt');

// Paused mode - must read manually
stream.on('readable', () => {
  let chunk;
  while (null !== (chunk = stream.read())) {
    console.log('Read chunk:', chunk.length, 'bytes');
  }
});

stream.on('end', () => {
  console.log('Stream ended');
});
```

### Switching Between Modes

```javascript
const stream = fs.createReadStream('file.txt');

// Switch to flowing mode
stream.resume();

// Switch to paused mode
stream.pause();

// Check current mode
console.log(stream.readableFlowing); // true, false, or null
```

### Custom Readable Stream

```javascript
const { Readable } = require('stream');

class CounterStream extends Readable {
  constructor(options) {
    super(options);
    this.max = options.max || 100;
    this.index = 1;
  }

  _read() {
    if (this.index > this.max) {
      this.push(null); // End of stream
      return;
    }

    const chunk = `${this.index}\n`;
    this.push(chunk);
    this.index++;
  }
}

// Usage
const counter = new CounterStream({ max: 10 });
counter.on('data', (chunk) => {
  process.stdout.write(chunk);
});
// Output: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
```

### Readable Stream Events

```javascript
const stream = fs.createReadStream('file.txt');

stream.on('readable', () => {
  // Data is available to read
});

stream.on('data', (chunk) => {
  // Chunk of data received (flowing mode)
});

stream.on('end', () => {
  // No more data will be provided
});

stream.on('error', (err) => {
  // An error occurred
  console.error('Stream error:', err);
});

stream.on('close', () => {
  // Stream is closed and no more events will be emitted
});
```

## Writable Streams

Writable streams represent a destination for data.

### Creating Writable Streams

```javascript
const fs = require('fs');
const { Writable } = require('stream');

// Method 1: Using fs.createWriteStream()
const fileStream = fs.createWriteStream('output.txt');

// Method 2: Creating custom writable stream
const writable = new Writable({
  write(chunk, encoding, callback) {
    // Process chunk
    callback(); // Call when done
  }
});
```

### Writing to Streams

```javascript
const fs = require('fs');
const stream = fs.createWriteStream('output.txt');

// Method 1: Using write()
stream.write('Hello ');
stream.write('World');
stream.end(); // Close the stream

// Method 2: Using end() with data
stream.end('Hello World');

// Method 3: Chaining
stream
  .write('Line 1\n')
  .write('Line 2\n')
  .end('Line 3\n');
```

### Handling Backpressure

Backpressure occurs when the data source is faster than the destination can handle.

```javascript
const fs = require('fs');
const writable = fs.createWriteStream('output.txt');

writable.on('drain', () => {
  // Stream is ready to receive more data
  console.log('Drained, can write more');
});

function writeData(data) {
  if (!writable.write(data)) {
    // Stream is full, wait for drain event
    writable.once('drain', () => {
      writeData(data);
    });
  }
}
```

### Custom Writable Stream

```javascript
const { Writable } = require('stream');

class LoggerStream extends Writable {
  _write(chunk, encoding, callback) {
    const message = chunk.toString();
    console.log(`[LOG] ${new Date().toISOString()}: ${message}`);
    callback(); // Signal that write is complete
  }
}

// Usage
const logger = new LoggerStream();
logger.write('User logged in');
logger.write('User performed action');
logger.end('User logged out');
```

### Writable Stream Events

```javascript
const stream = fs.createWriteStream('output.txt');

stream.on('drain', () => {
  // Stream is ready to receive more data (backpressure released)
});

stream.on('finish', () => {
  // All data has been flushed to the underlying system
  console.log('All data written');
});

stream.on('error', (err) => {
  // An error occurred
  console.error('Write error:', err);
});

stream.on('close', () => {
  // Stream is closed
});
```

## Duplex Streams

Duplex streams are both readable and writable (like a two-way street).

### Creating Duplex Streams

```javascript
const { Duplex } = require('stream');

class EchoStream extends Duplex {
  constructor(options) {
    super(options);
    this.buffer = [];
  }

  _read() {
    // Read from buffer
    if (this.buffer.length > 0) {
      this.push(this.buffer.shift());
    }
  }

  _write(chunk, encoding, callback) {
    // Write to buffer
    this.buffer.push(chunk);
    callback();
  }
}

// Usage
const echo = new EchoStream();
echo.on('data', (chunk) => {
  console.log('Echoed:', chunk.toString());
});

echo.write('Hello');
echo.write('World');
echo.end();
```

### Real-World Example: TCP Socket

```javascript
const net = require('net');

const server = net.createServer((socket) => {
  // socket is a Duplex stream
  console.log('Client connected');

  // Readable side
  socket.on('data', (data) => {
    console.log('Received:', data.toString());
    
    // Writable side
    socket.write(`Echo: ${data}`);
  });

  socket.on('end', () => {
    console.log('Client disconnected');
  });
});

server.listen(3000);
```

## Transform Streams

Transform streams are a special type of Duplex stream where the output is computed from the input.

### Creating Transform Streams

```javascript
const { Transform } = require('stream');

class UpperCaseTransform extends Transform {
  _transform(chunk, encoding, callback) {
    // Transform the chunk
    const upper = chunk.toString().toUpperCase();
    this.push(upper);
    callback();
  }
}

// Usage
const upper = new UpperCaseTransform();
upper.on('data', (chunk) => {
  console.log(chunk.toString());
});

upper.write('hello');
upper.write('world');
upper.end();
// Output: HELLOWORLD
```

### Common Transform Examples

#### 1. Line Splitter

```javascript
const { Transform } = require('stream');

class LineSplitter extends Transform {
  constructor(options) {
    super(options);
    this.buffer = '';
  }

  _transform(chunk, encoding, callback) {
    this.buffer += chunk.toString();
    const lines = this.buffer.split('\n');
    this.buffer = lines.pop(); // Keep incomplete line

    lines.forEach(line => {
      this.push(line);
    });

    callback();
  }

  _flush(callback) {
    // Handle remaining data
    if (this.buffer) {
      this.push(this.buffer);
    }
    callback();
  }
}
```

#### 2. JSON Parser

```javascript
const { Transform } = require('stream');

class JSONParser extends Transform {
  constructor(options) {
    super(options);
    this.buffer = '';
  }

  _transform(chunk, encoding, callback) {
    this.buffer += chunk.toString();
    
    try {
      let index;
      while ((index = this.buffer.indexOf('\n')) !== -1) {
        const line = this.buffer.slice(0, index);
        this.buffer = this.buffer.slice(index + 1);
        
        if (line.trim()) {
          const obj = JSON.parse(line);
          this.push(JSON.stringify(obj) + '\n');
        }
      }
    } catch (err) {
      callback(err);
      return;
    }
    
    callback();
  }
}
```

#### 3. Data Filter

```javascript
const { Transform } = require('stream');

class FilterStream extends Transform {
  constructor(filterFn, options) {
    super(options);
    this.filterFn = filterFn;
  }

  _transform(chunk, encoding, callback) {
    if (this.filterFn(chunk)) {
      this.push(chunk);
    }
    callback();
  }
}

// Usage: Filter out lines shorter than 10 characters
const filter = new FilterStream((chunk) => {
  return chunk.length >= 10;
});
```

## Piping Streams

Piping connects streams together, automatically handling backpressure and errors.

### Basic Piping

```javascript
const fs = require('fs');

// Read from one file and write to another
fs.createReadStream('input.txt')
  .pipe(fs.createWriteStream('output.txt'));
```

### Chaining Pipes

```javascript
const fs = require('fs');
const zlib = require('zlib');

// Read -> Compress -> Write
fs.createReadStream('file.txt')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('file.txt.gz'));
```

### Piping with Transform

```javascript
const fs = require('fs');
const { Transform } = require('stream');

class AddLineNumbers extends Transform {
  constructor(options) {
    super(options);
    this.lineNumber = 1;
  }

  _transform(chunk, encoding, callback) {
    const lines = chunk.toString().split('\n');
    const numbered = lines
      .map(line => `${this.lineNumber++}: ${line}`)
      .join('\n');
    this.push(numbered);
    callback();
  }
}

fs.createReadStream('file.txt')
  .pipe(new AddLineNumbers())
  .pipe(fs.createWriteStream('numbered.txt'));
```

### Error Handling in Pipes

```javascript
const fs = require('fs');

const source = fs.createReadStream('input.txt');
const dest = fs.createWriteStream('output.txt');

source
  .pipe(dest)
  .on('error', (err) => {
    // Handle errors in destination
    console.error('Destination error:', err);
    source.destroy(); // Stop reading
  });

source.on('error', (err) => {
  // Handle errors in source
  console.error('Source error:', err);
  dest.destroy(); // Stop writing
});
```

## Common Use Cases

### 1. File Processing

```javascript
const fs = require('fs');
const { Transform } = require('stream');

// Process large file line by line
class LineProcessor extends Transform {
  _transform(chunk, encoding, callback) {
    const lines = chunk.toString().split('\n');
    lines.forEach(line => {
      if (line.trim()) {
        // Process each line
        const processed = line.toUpperCase();
        this.push(processed + '\n');
      }
    });
    callback();
  }
}

fs.createReadStream('large-file.txt')
  .pipe(new LineProcessor())
  .pipe(fs.createWriteStream('processed.txt'));
```

### 2. HTTP Request/Response

```javascript
const http = require('http');
const fs = require('fs');

const server = http.createServer((req, res) => {
  // Response is a writable stream
  if (req.url === '/file') {
    const fileStream = fs.createReadStream('large-file.txt');
    fileStream.pipe(res); // Stream file to response
  } else {
    res.end('Hello World');
  }
});

server.listen(3000);
```

### 3. Data Transformation Pipeline

```javascript
const fs = require('fs');
const { Transform, pipeline } = require('stream');
const zlib = require('zlib');

// Step 1: Parse CSV
class CSVParser extends Transform {
  _transform(chunk, encoding, callback) {
    const lines = chunk.toString().split('\n');
    lines.forEach(line => {
      if (line.trim()) {
        const fields = line.split(',');
        this.push(JSON.stringify(fields) + '\n');
      }
    });
    callback();
  }
}

// Step 2: Filter data
class DataFilter extends Transform {
  _transform(chunk, encoding, callback) {
    const data = JSON.parse(chunk.toString());
    if (data[2] === 'active') { // Filter by status
      this.push(chunk);
    }
    callback();
  }
}

// Pipeline: Read -> Parse -> Filter -> Compress -> Write
pipeline(
  fs.createReadStream('data.csv'),
  new CSVParser(),
  new DataFilter(),
  zlib.createGzip(),
  fs.createWriteStream('filtered-data.json.gz'),
  (err) => {
    if (err) {
      console.error('Pipeline error:', err);
    } else {
      console.log('Pipeline completed');
    }
  }
);
```

### 4. Log Processing

```javascript
const fs = require('fs');
const { Transform } = require('stream');

class LogParser extends Transform {
  constructor(options) {
    super(options);
    this.stats = {
      errors: 0,
      warnings: 0,
      info: 0
    };
  }

  _transform(chunk, encoding, callback) {
    const line = chunk.toString();
    
    if (line.includes('[ERROR]')) {
      this.stats.errors++;
      this.push(`ERROR: ${line}`);
    } else if (line.includes('[WARN]')) {
      this.stats.warnings++;
    } else if (line.includes('[INFO]')) {
      this.stats.info++;
    }
    
    callback();
  }

  _flush(callback) {
    this.push(`\nStats: ${JSON.stringify(this.stats)}\n`);
    callback();
  }
}

fs.createReadStream('app.log')
  .pipe(new LogParser())
  .pipe(fs.createWriteStream('errors.log'));
```

### 5. Real-Time Data Processing

```javascript
const { Readable, Transform } = require('stream');

class DataGenerator extends Readable {
  constructor(options) {
    super(options);
    this.count = 0;
    this.max = options.max || 100;
  }

  _read() {
    if (this.count >= this.max) {
      this.push(null);
      return;
    }

    const data = {
      id: this.count++,
      timestamp: Date.now(),
      value: Math.random() * 100
    };

    this.push(JSON.stringify(data) + '\n');
  }
}

class DataProcessor extends Transform {
  _transform(chunk, encoding, callback) {
    const data = JSON.parse(chunk.toString());
    
    // Process data
    data.processed = true;
    data.result = data.value * 2;
    
    this.push(JSON.stringify(data) + '\n');
    callback();
  }
}

// Generate -> Process -> Output
new DataGenerator({ max: 10 })
  .pipe(new DataProcessor())
  .pipe(process.stdout);
```

## The `pipeline()` Function

The `pipeline()` function is the recommended way to pipe streams together, as it properly handles errors and cleanup.

```javascript
const { pipeline } = require('stream');
const fs = require('fs');
const zlib = require('zlib');

pipeline(
  fs.createReadStream('input.txt'),
  zlib.createGzip(),
  fs.createWriteStream('output.txt.gz'),
  (err) => {
    if (err) {
      console.error('Pipeline failed:', err);
    } else {
      console.log('Pipeline succeeded');
    }
  }
);
```

### Benefits of `pipeline()`

1. **Automatic error handling** - Errors propagate correctly
2. **Cleanup** - Destroys streams on error
3. **Backpressure handling** - Automatically managed
4. **Callback on completion** - Know when pipeline is done

## Backpressure

Backpressure occurs when data is produced faster than it can be consumed. Streams handle this automatically when piped.

### Understanding Backpressure

```javascript
const { Readable, Writable } = require('stream');

// Fast producer
const producer = new Readable({
  read() {
    this.push('data\n');
  }
});

// Slow consumer
const consumer = new Writable({
  write(chunk, encoding, callback) {
    console.log('Processing:', chunk.toString());
    // Simulate slow processing
    setTimeout(callback, 100);
  }
});

// Piping handles backpressure automatically
producer.pipe(consumer);
```

### Manual Backpressure Handling

```javascript
const fs = require('fs');

const writable = fs.createWriteStream('output.txt');

function writeData(data) {
  if (!writable.write(data)) {
    // Stream is full, wait for drain
    writable.once('drain', () => {
      writeData(data);
    });
  }
}

// High-speed data source
for (let i = 0; i < 1000000; i++) {
  writeData(`Line ${i}\n`);
}
```

## Error Handling

### Error Handling in Pipes

```javascript
const fs = require('fs');

const source = fs.createReadStream('input.txt');
const dest = fs.createWriteStream('output.txt');

source.on('error', (err) => {
  console.error('Source error:', err);
  dest.destroy(err);
});

dest.on('error', (err) => {
  console.error('Destination error:', err);
  source.destroy(err);
});

source.pipe(dest);
```

### Error Handling with `pipeline()`

```javascript
const { pipeline } = require('stream');
const fs = require('fs');

pipeline(
  fs.createReadStream('input.txt'),
  fs.createWriteStream('output.txt'),
  (err) => {
    if (err) {
      console.error('Pipeline error:', err);
    }
  }
);
```

### Error Propagation in Custom Streams

```javascript
const { Transform } = require('stream');

class SafeTransform extends Transform {
  _transform(chunk, encoding, callback) {
    try {
      const result = this.process(chunk);
      this.push(result);
      callback();
    } catch (err) {
      callback(err); // Propagate error
    }
  }

  process(chunk) {
    // Your processing logic
    return chunk;
  }
}
```

## Object Mode

Streams can work with objects instead of buffers/strings.

```javascript
const { Readable, Transform } = require('stream');

// Object mode readable
const objectStream = new Readable({
  objectMode: true,
  read() {
    this.push({ id: 1, name: 'John' });
    this.push({ id: 2, name: 'Jane' });
    this.push(null); // End
  }
});

// Object mode transform
const processor = new Transform({
  objectMode: true,
  transform(obj, encoding, callback) {
    obj.processed = true;
    this.push(obj);
    callback();
  }
});

objectStream
  .pipe(processor)
  .on('data', (obj) => {
    console.log('Received:', obj);
  });
```

## High Water Mark

The high water mark is the maximum number of bytes (or objects) that can be buffered before backpressure kicks in.

```javascript
const { Readable } = require('stream');

// Default high water mark is 16KB (16384 bytes)
const stream1 = new Readable({
  highWaterMark: 16384
});

// Custom high water mark
const stream2 = new Readable({
  highWaterMark: 1024 * 1024 // 1MB
});

// Object mode high water mark (number of objects)
const stream3 = new Readable({
  objectMode: true,
  highWaterMark: 16 // 16 objects
});
```

## Best Practices

### 1. Always Handle Errors

```javascript
// ✅ Good
stream.on('error', (err) => {
  console.error('Stream error:', err);
});

// ❌ Bad
// No error handling
```

### 2. Use `pipeline()` for Multiple Streams

```javascript
// ✅ Good
pipeline(
  source,
  transform1,
  transform2,
  destination,
  (err) => { /* handle error */ }
);

// ❌ Bad
source
  .pipe(transform1)
  .pipe(transform2)
  .pipe(destination);
```

### 3. Destroy Streams on Error

```javascript
// ✅ Good
source.on('error', (err) => {
  dest.destroy(err);
});

dest.on('error', (err) => {
  source.destroy(err);
});
```

### 4. Handle Backpressure

```javascript
// ✅ Good
if (!writable.write(data)) {
  writable.once('drain', () => {
    // Continue writing
  });
}
```

### 5. Use `_flush()` for Cleanup

```javascript
class MyTransform extends Transform {
  _transform(chunk, encoding, callback) {
    // Process chunk
    callback();
  }

  _flush(callback) {
    // Cleanup or final processing
    this.push('Final data');
    callback();
  }
}
```

### 6. Avoid Memory Leaks

```javascript
// ✅ Good: Remove listeners
stream.on('data', handler);
stream.on('end', () => {
  stream.removeAllListeners();
});

// ✅ Good: Use once() for one-time events
stream.once('error', handler);
```

## Common Patterns

### Pattern 1: Stream Factory

```javascript
function createProcessor(options) {
  return new Transform({
    objectMode: true,
    transform(chunk, encoding, callback) {
      // Process based on options
      const result = processChunk(chunk, options);
      this.push(result);
      callback();
    }
  });
}

// Usage
const processor = createProcessor({ mode: 'uppercase' });
```

### Pattern 2: Stream Composition

```javascript
function createPipeline(...transforms) {
  return (source) => {
    let stream = source;
    transforms.forEach(transform => {
      stream = stream.pipe(transform);
    });
    return stream;
  };
}

// Usage
const pipeline = createPipeline(
  new Transform1(),
  new Transform2(),
  new Transform3()
);

pipeline(fs.createReadStream('input.txt'))
  .pipe(fs.createWriteStream('output.txt'));
```

### Pattern 3: Conditional Piping

```javascript
const source = fs.createReadStream('input.txt');
const transform = new MyTransform();

if (condition) {
  source.pipe(transform).pipe(fs.createWriteStream('output.txt'));
} else {
  source.pipe(fs.createWriteStream('output.txt'));
}
```

## Performance Tips

### 1. Use Appropriate Chunk Sizes

```javascript
// Larger chunks for better performance with large files
const stream = fs.createReadStream('file.txt', {
  highWaterMark: 1024 * 1024 // 1MB chunks
});
```

### 2. Avoid Unnecessary Transformations

```javascript
// ✅ Good: Direct pipe when possible
readStream.pipe(writeStream);

// ❌ Bad: Unnecessary transform
readStream
  .pipe(new Transform({ transform: (ch, enc, cb) => { this.push(ch); cb(); } }))
  .pipe(writeStream);
```

### 3. Use Object Mode for Structured Data

```javascript
// ✅ Good: Object mode for structured data
const stream = new Transform({
  objectMode: true,
  transform(obj, enc, cb) {
    this.push(processObject(obj));
    cb();
  }
});
```

## Summary

### Key Concepts

- **Streams** process data in chunks, making them memory-efficient
- **Four types**: Readable, Writable, Duplex, Transform
- **Two modes**: Flowing (automatic) and Paused (manual)
- **Piping** connects streams and handles backpressure automatically
- **Backpressure** prevents memory issues when producer is faster than consumer

### When to Use Streams

- Large files (don't load entire file into memory)
- Real-time data processing
- Network operations
- Data transformation pipelines
- Log processing
- File compression/decompression

### Stream Methods Quick Reference

```javascript
// Readable
stream.read()
stream.pause()
stream.resume()
stream.pipe(destination)
stream.on('data', handler)
stream.on('end', handler)
stream.on('error', handler)

// Writable
stream.write(chunk)
stream.end(chunk)
stream.on('drain', handler)
stream.on('finish', handler)
stream.on('error', handler)

// Both
stream.destroy()
stream.on('close', handler)

// Utility
pipeline(...streams, callback)
```

## Additional Resources

- [Node.js Streams Documentation](https://nodejs.org/api/stream.html)
- [Stream Handbook](https://github.com/substack/stream-handbook)
- [Backpressure in Streams](https://nodejs.org/en/docs/guides/backpressuring-in-streams/)

