# Node.js Elements Quick Reference

## Table of Contents
- [Core Modules](#core-modules)
- [Module System](#module-system)
- [File System Operations](#file-system-operations)
- [Streams & Buffers](#streams--buffers)
- [Event Emitter](#event-emitter)
- [HTTP Module](#http-module)
- [Path Module](#path-module)
- [Process & Environment](#process--environment)

## Core Modules

### Built-in Modules
```javascript
// Common core modules
const fs = require('fs');
const path = require('path');
const http = require('http');
const events = require('events');
const crypto = require('crypto');
const os = require('os');
const url = require('url');

// ES Modules syntax
import fs from 'fs';
import path from 'path';
```

### Module Usage
```javascript
// Using multiple core modules
const { createServer } = require('http');
const { join } = require('path');
const { readFile } = require('fs').promises;

// Module information
require.main === module  // Check if file is run directly
module.exports          // Export module contents
exports                 // Shorthand for module.exports
```

## Module System

### Creating Modules
```javascript
// module.exports
module.exports = {
    add(x, y) {
        return x + y;
    },
    subtract(x, y) {
        return x - y;
    }
};

// ES6 export
export const add = (x, y) => x + y;
export const subtract = (x, y) => x - y;
```

### Importing Modules
```javascript
// CommonJS require
const math = require('./math');
const { add, subtract } = require('./math');

// ES6 import
import math from './math';
import { add, subtract } from './math';
```

## File System Operations

### Basic File Operations
```javascript
const fs = require('fs');

// Synchronous operations
const content = fs.readFileSync('file.txt', 'utf8');
fs.writeFileSync('file.txt', 'Hello World');

// Asynchronous operations
fs.readFile('file.txt', 'utf8', (err, data) => {
    if (err) throw err;
    console.log(data);
});

// Promise-based operations
const { readFile, writeFile } = require('fs').promises;

async function handleFile() {
    const content = await readFile('file.txt', 'utf8');
    await writeFile('new.txt', content);
}
```

### Directory Operations
```javascript
// Create directory
fs.mkdirSync('newDir');
fs.mkdir('newDir', { recursive: true }, (err) => {});

// Read directory
const files = fs.readdirSync('.');
fs.readdir('.', (err, files) => {});

// Check if exists
fs.existsSync('file.txt');

// Get file info
const stats = fs.statSync('file.txt');
```

## Streams & Buffers

### Stream Types
```javascript
const { createReadStream, createWriteStream } = require('fs');

// Reading streams
const readStream = createReadStream('file.txt', {
    encoding: 'utf8',
    highWaterMark: 64 * 1024 // 64KB chunks
});

// Writing streams
const writeStream = createWriteStream('output.txt');

// Piping streams
readStream.pipe(writeStream);
```

### Buffer Operations
```javascript
// Creating buffers
const buf1 = Buffer.alloc(10);
const buf2 = Buffer.from('Hello World');
const buf3 = Buffer.from([1, 2, 3]);

// Buffer operations
buf2.toString();               // Convert to string
buf2.length;                   // Get size
buf2.slice(0, 5);             // Get portion
Buffer.concat([buf1, buf2]);   // Combine buffers
```

## Event Emitter

### Basic Usage
```javascript
const EventEmitter = require('events');

class MyEmitter extends EventEmitter {}
const myEmitter = new MyEmitter();

// Event handling
myEmitter.on('event', (data) => {
    console.log('Event occurred:', data);
});

myEmitter.once('oneTime', (data) => {
    console.log('This runs once only');
});

// Emit events
myEmitter.emit('event', { message: 'Hello' });
```

### Event Methods
```javascript
// Event listener management
myEmitter.addListener('event', callback);    // Add listener
myEmitter.removeListener('event', callback); // Remove specific
myEmitter.removeAllListeners('event');      // Remove all
myEmitter.listenerCount('event');           // Count listeners
myEmitter.listeners('event');               // Get listeners
```

## HTTP Module

### Creating Servers
```javascript
const http = require('http');

// Basic server
const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Hello World\n');
});

server.listen(3000, () => {
    console.log('Server running at http://localhost:3000/');
});

// Making requests
http.get('http://example.com', (res) => {
    let data = '';
    res.on('data', chunk => data += chunk);
    res.on('end', () => console.log(data));
}).on('error', console.error);
```

### Request & Response
```javascript
// Request object properties
req.method           // GET, POST, etc.
req.url             // Request URL
req.headers         // Request headers
req.statusCode      // Status code

// Response methods
res.writeHead(200, { 'Content-Type': 'application/json' });
res.write('Some data');
res.end('Final data');
```

## Path Module

### Path Operations
```javascript
const path = require('path');

// Path methods
path.join('/foo', 'bar', 'baz');    // Join paths
path.resolve('foo', 'bar');         // Resolve to absolute
path.dirname('/foo/bar/file.txt');  // Get directory name
path.basename('/foo/bar/file.txt'); // Get file name
path.extname('file.txt');           // Get extension

// Path properties
path.sep                            // Platform separator
path.delimiter                      // Path delimiter
```

## Process & Environment

### Process Information
```javascript
// Process properties
process.pid                 // Process ID
process.version            // Node.js version
process.platform          // Operating system
process.arch              // Architecture
process.uptime()          // Process uptime
process.memoryUsage()     // Memory usage

// Environment variables
process.env.NODE_ENV      // Environment mode
process.env.PORT          // Port number

// Process events
process.on('exit', code => {
    console.log('Exiting with code:', code);
});

process.on('uncaughtException', err => {
    console.error('Uncaught Exception:', err);
    process.exit(1);
});
```

### Command Line Arguments
```javascript
// Access arguments
process.argv             // Array of arguments
process.argv.slice(2)    // User arguments

// Exit process
process.exit(0);         // Success exit
process.exit(1);         // Error exit

// Current directory
process.cwd()            // Current working directory
process.chdir('/tmp')    // Change directory
```
