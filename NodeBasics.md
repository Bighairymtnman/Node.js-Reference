# Node.js Basics Guide

## Table of Contents
- [Introduction to Node.js](#introduction-to-nodejs)
- [Setting Up Development Environment](#setting-up-development-environment)
- [Node.js Fundamentals](#nodejs-fundamentals)
- [Working with NPM](#working-with-npm)
- [Basic Server Creation](#basic-server-creation)
- [File Operations](#file-operations)
- [Error Handling](#error-handling)
- [Debugging Techniques](#debugging-techniques)

## Introduction to Node.js

### What is Node.js?
```javascript
// Node.js allows you to run JavaScript on your computer/server instead of just in browsers
// It's built on Chrome's V8 engine and adds capabilities like file system access

// Here's a basic server example showing Node's power:
const http = require('http');

// Create a simple web server that responds to all requests
const server = http.createServer((req, res) => {
    // Send "Hello from Node.js!" to any browser that connects
    res.end('Hello from Node.js!');
});

// Start the server on port 3000
server.listen(3000, () => {
    console.log('Server running on port 3000');
    console.log('Open your browser and visit: http://localhost:3000');
});
```

### Key Features
```javascript
// 1. Asynchronous Operations
// Node.js is non-blocking - it can do other things while waiting for operations to complete
const fs = require('fs');

console.log('Starting file read...'); // This runs first
fs.readFile('file.txt', (err, data) => {
    // This runs third, after file is read
    if (err) throw err;
    console.log('File contents:', data);
}); 
console.log('Doing other work...'); // This runs second

// 2. Event-Driven Programming
// Node.js uses events to handle multiple operations
const EventEmitter = require('events');
const myEmitter = new EventEmitter();

// Set up an event listener
myEmitter.on('userLoggedIn', (username) => {
    console.log(`${username} just logged in!`);
});

// Trigger the event somewhere in your code
myEmitter.emit('userLoggedIn', 'John');

// 3. Cross-Platform Compatibility
// Your Node.js code works on Windows, Mac, and Linux
console.log(`This code is running on: ${process.platform}`);
console.log(`Using Node.js version: ${process.version}`);
```

## Setting Up Development Environment

### Project Initialization
```javascript
// Understanding Project Structure
/*
project/
├── node_modules/    <- All installed packages live here
├── src/            <- Your source code goes here
│   └── index.js    <- Main application file
├── package.json    <- Project configuration and dependencies
└── package-lock.json <- Exact dependency versions
*/

// Initialize a new Node.js project
// Run this in your terminal:
// $ npm init -y

// package.json - The heart of any Node.js project
{
    "name": "my-node-project",
    "version": "1.0.0",
    "main": "index.js",
    // Scripts automate common tasks
    "scripts": {
        "start": "node src/index.js",        // Production start
        "dev": "nodemon src/index.js",       // Development with auto-reload
        "test": "jest",                      // Run tests
        "lint": "eslint ."                   // Check code quality
    }
}
```

### Essential Development Tools
```javascript
// Must-have development tools for Node.js

// 1. Nodemon - Automatically restarts your app when files change
// Install with: npm install nodemon --save-dev
{
    "scripts": {
        "dev": "nodemon src/index.js"
    }
}

// 2. Environment Variables - Keep configuration secure
// Create a .env file:
DATABASE_URL=mongodb://localhost:27017
API_KEY=your_secret_key
NODE_ENV=development

// Load environment variables in your code
require('dotenv').config();
const dbUrl = process.env.DATABASE_URL;  // Access variables safely

// 3. ESLint - Catch common mistakes and enforce code style
// Install with: npm install eslint --save-dev
// .eslintrc configuration
{
    "env": {
        "node": true,     // Node.js global variables
        "es6": true      // Enable ES6 features
    },
    "extends": "eslint:recommended",
    "rules": {
        "no-console": "warn"    // Warn about console.log usage
    }
}
```

## Node.js Fundamentals

### Module Patterns
```javascript
// Different ways to organize and share code between files

// 1. CommonJS Modules (Traditional Node.js)
// math.js - Creating a module
module.exports = {
    add: (a, b) => {
        return a + b;  // Simple addition
    },
    subtract: (a, b) => {
        return a - b;  // Simple subtraction
    },
    // You can add as many functions as needed
    multiply: (a, b) => a * b
};

// Using the math module in another file
const math = require('./math');
console.log(math.add(5, 3));      // Output: 8
console.log(math.multiply(4, 2)); // Output: 8

// 2. ES Modules (Modern JavaScript)
// utils.js
export const formatDate = (date) => {
    // Convert date to user-friendly format
    return new Date(date).toLocaleDateString();
};

export const capitalizeString = (str) => {
    // Capitalize first letter of string
    return str.charAt(0).toUpperCase() + str.slice(1);
};

// Using ES modules (need "type": "module" in package.json)
import { formatDate, capitalizeString } from './utils.js';
```

### Callback Pattern
```javascript
// Understanding callbacks - functions that run after something completes

// 1. Basic Callback Example
function fetchUserData(userId, callback) {
    // Simulate database query
    setTimeout(() => {
        const user = {
            id: userId,
            name: 'John Doe',
            email: 'john@example.com'
        };
        callback(null, user);  // Success case
    }, 1000);
}

// Using the callback
fetchUserData(123, (error, user) => {
    if (error) {
        console.error('Failed to fetch user:', error);
        return;
    }
    console.log('User data:', user);
});

// 2. Error-First Callback Pattern (Node.js standard)
function readConfig(path, callback) {
    fs.readFile(path, 'utf8', (error, data) => {
        // Always handle errors first
        if (error) {
            callback(error);
            return;
        }
        
        try {
            // Parse JSON data
            const config = JSON.parse(data);
            callback(null, config);  // Success case
        } catch (parseError) {
            callback(parseError);    // Handle JSON parsing errors
        }
    });
}

// Using error-first callback
readConfig('config.json', (error, config) => {
    if (error) {
        console.error('Config error:', error);
        return;
    }
    console.log('Configuration loaded:', config);
});
```



## Working with NPM

### Package Management
```javascript
// NPM (Node Package Manager) - Essential for managing project dependencies

// 1. Installing Packages
// Common npm commands:
// $ npm install express    -> Install production dependency
// $ npm install --save-dev jest    -> Install development dependency
// $ npm install -g nodemon -> Install global package

// package.json dependency management
{
    "dependencies": {
        // Production packages - needed to run your app
        "express": "^4.17.1",     // Web framework
        "mongoose": "^5.12.3",    // MongoDB integration
        "dotenv": "^10.0.0"       // Environment variables
    },
    "devDependencies": {
        // Development packages - only needed for development
        "nodemon": "^2.0.7",      // Auto-reload during development
        "jest": "^26.6.3",        // Testing framework
        "eslint": "^7.32.0"       // Code quality checking
    }
}

// Version numbers explained:
// ^4.17.1 means: can update to 4.x.x but not 5.x.x
// ~4.17.1 means: can update to 4.17.x but not 4.18.x
// 4.17.1 exact means: use exactly this version
```

### Creating NPM Scripts
```javascript
// NPM scripts automate common tasks in your project
{
    "scripts": {
        // Basic scripts
        "start": "node index.js",  // Production startup
        "dev": "nodemon index.js", // Development with auto-reload
        
        // Build process scripts
        "clean": "rm -rf dist",    // Remove build directory
        "build": "npm run clean && babel src -d dist",  // Clean and build
        
        // Environment-specific scripts
        "start:dev": "NODE_ENV=development nodemon src/index.js",
        "start:prod": "NODE_ENV=production node dist/index.js",
        
        // Testing and quality scripts
        "test": "jest",            // Run all tests
        "test:watch": "jest --watch",  // Run tests in watch mode
        "lint": "eslint .",        // Check code quality
        "lint:fix": "eslint . --fix"  // Fix code quality issues
    }
}

// Running scripts:
// $ npm run dev        -> Starts development server
// $ npm test          -> Runs tests (special command, doesn't need 'run')
// $ npm run build     -> Builds the project
```

## Basic Server Creation

### HTTP Server
```javascript
// Creating a basic HTTP server without frameworks
const http = require('http');

const server = http.createServer((req, res) => {
    // Basic routing system
    switch (req.url) {
        case '/':
            // Home page
            res.writeHead(200, { 
                'Content-Type': 'text/html'
            });
            res.end('<h1>Welcome to our server!</h1>');
            break;
            
        case '/api/users':
            // API endpoint
            if (req.method === 'GET') {
                res.writeHead(200, { 
                    'Content-Type': 'application/json'
                });
                res.end(JSON.stringify({
                    users: [
                        { id: 1, name: 'John' },
                        { id: 2, name: 'Jane' }
                    ]
                }));
            }
            break;
            
        default:
            // 404 Not Found
            res.writeHead(404);
            res.end('Page not found');
    }
});

// Start server with error handling
server.listen(3000, (error) => {
    if (error) {
        console.error('Failed to start server:', error);
        return;
    }
    console.log('Server running at http://localhost:3000');
});
```

### Express Server
```javascript
// Express - Popular web framework for Node.js
const express = require('express');
const app = express();

// Middleware - Functions that process requests
app.use(express.json());  // Parse JSON request bodies
app.use(express.urlencoded({ extended: true }));  // Parse URL-encoded bodies

// Custom middleware example
app.use((req, res, next) => {
    console.log(`${req.method} ${req.url} at ${new Date()}`);
    next();  // Continue to next middleware/route
});

// Route handling
app.get('/', (req, res) => {
    res.send('Welcome to Express!');
});

// RESTful API routes
app.get('/api/users', (req, res) => {
    // Get all users
    res.json([
        { id: 1, name: 'John' },
        { id: 2, name: 'Jane' }
    ]);
});

app.post('/api/users', (req, res) => {
    // Create new user
    const { name, email } = req.body;
    // Validate input
    if (!name || !email) {
        return res.status(400).json({
            error: 'Name and email are required'
        });
    }
    // Process the data...
    res.status(201).json({
        message: 'User created successfully'
    });
});

// Error handling middleware - Always put last
app.use((err, req, res, next) => {
    console.error(err.stack);
    res.status(500).json({
        error: 'Something went wrong!',
        message: err.message
    });
});

// Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Express server running at http://localhost:${PORT}`);
});
```


## File Operations

### Basic File Handling
```javascript
// Working with files in Node.js using promises
const fs = require('fs').promises;

// 1. Reading Files
async function readFileContent(path) {
    try {
        // Read file contents
        const content = await fs.readFile(path, 'utf8');
        return content;
    } catch (error) {
        // Handle specific error types
        if (error.code === 'ENOENT') {
            console.error('File not found:', path);
        } else {
            console.error('Error reading file:', error.message);
        }
        throw error;
    }
}

// Example usage:
async function processConfigFile() {
    try {
        const config = await readFileContent('config.json');
        const settings = JSON.parse(config);
        console.log('Configuration loaded:', settings);
    } catch (error) {
        console.error('Failed to load configuration');
    }
}

// 2. Writing Files
async function writeFileContent(path, content) {
    try {
        // Write or overwrite file
        await fs.writeFile(path, content);
        console.log('File written successfully:', path);
        
        // You can also append to files
        await fs.appendFile(path, '\nNew content');
        console.log('Content appended');
    } catch (error) {
        console.error('Error writing file:', error.message);
        throw error;
    }
}

// Example: Creating a log file
async function logActivity(activity) {
    const timestamp = new Date().toISOString();
    const logEntry = `${timestamp}: ${activity}\n`;
    await writeFileContent('app.log', logEntry);
}
```

### Working with Directories
```javascript
// Directory operations in Node.js

// 1. Creating Directories
async function createDirectory(path) {
    try {
        // Create directory (and parent directories if needed)
        await fs.mkdir(path, { recursive: true });
        console.log('Directory created:', path);
        
        // You can also set permissions
        await fs.chmod(path, 0o755);  // rwxr-xr-x
    } catch (error) {
        console.error('Error creating directory:', error.message);
        throw error;
    }
}

// 2. Reading Directory Contents
async function listDirectory(path) {
    try {
        // Get all files and folders
        const entries = await fs.readdir(path, { withFileTypes: true });
        
        // Separate files and directories
        const files = entries
            .filter(entry => entry.isFile())
            .map(entry => entry.name);
            
        const directories = entries
            .filter(entry => entry.isDirectory())
            .map(entry => entry.name);
            
        return { files, directories };
    } catch (error) {
        console.error('Error reading directory:', error.message);
        throw error;
    }
}

// Example: Process all files in a directory
async function processDirectory(path) {
    const { files, directories } = await listDirectory(path);
    
    console.log('Files found:', files);
    console.log('Directories found:', directories);
    
    // Process each file
    for (const file of files) {
        const content = await readFileContent(`${path}/${file}`);
        // Process content...
    }
}
```

## Error Handling

### Try-Catch Pattern
```javascript
// Modern error handling in Node.js

// 1. Async/Await Error Handling
async function fetchUserData(id) {
    try {
        // Database operation that might fail
        const user = await database.findUser(id);
        
        if (!user) {
            // Custom error for missing user
            throw new Error(`User ${id} not found`);
        }
        
        return user;
    } catch (error) {
        // Log error details for debugging
        console.error('Error fetching user:', {
            userId: id,
            error: error.message,
            stack: error.stack
        });
        
        // Rethrow with more context
        throw new Error(`Failed to fetch user: ${error.message}`);
    }
}

// 2. Promise-based Error Handling
function processData(data) {
    return new Promise((resolve, reject) => {
        // Validate input
        if (!data) {
            reject(new Error('No data provided'));
            return;
        }
        
        // Process data asynchronously
        setTimeout(() => {
            try {
                const result = someRiskyOperation(data);
                resolve(result);
            } catch (error) {
                reject(new Error(`Processing failed: ${error.message}`));
            }
        }, 100);
    });
}
```

### Custom Error Types
```javascript
// Creating specialized error types for better error handling

// 1. Base Custom Error
class AppError extends Error {
    constructor(message, statusCode = 500) {
        super(message);
        this.name = this.constructor.name;
        this.statusCode = statusCode;
        Error.captureStackTrace(this, this.constructor);
    }
}

// 2. Specific Error Types
class ValidationError extends AppError {
    constructor(message) {
        super(message, 400);  // 400 Bad Request
        this.validationErrors = [];
    }
    
    addError(field, message) {
        this.validationErrors.push({ field, message });
    }
}

class DatabaseError extends AppError {
    constructor(message, operation) {
        super(message, 500);  // 500 Internal Server Error
        this.operation = operation;
    }
}

// Using custom errors
function validateUser(user) {
    const error = new ValidationError('Invalid user data');
    
    if (!user.name) {
        error.addError('name', 'Name is required');
    }
    
    if (!user.email) {
        error.addError('email', 'Email is required');
    }
    
    if (error.validationErrors.length > 0) {
        throw error;
    }
}

// Example usage in Express
app.post('/users', (req, res, next) => {
    try {
        validateUser(req.body);
        // Process valid user...
    } catch (error) {
        if (error instanceof ValidationError) {
            res.status(error.statusCode).json({
                error: error.message,
                details: error.validationErrors
            });
        } else {
            next(error);  // Pass to error handler
        }
    }
});
```

## Debugging Techniques

### Console Methods
```javascript
// Advanced console usage for debugging

// 1. Different Console Methods
console.log('Basic logging');
console.info('Information', { user: 'John' });  // Info level
console.warn('Warning!', 'Low memory');         // Warning level
console.error('Error occurred', new Error());   // Error level

// 2. Structured Console Output
// Table format for arrays/objects
console.table([
    { name: 'John', age: 30 },
    { name: 'Jane', age: 25 }
]);

// Time operations
console.time('operation');
someTimeConsumingOperation();
console.timeEnd('operation');  // Shows duration

// Group related logs
console.group('User Processing');
console.log('Fetching user...');
console.log('Processing data...');
console.groupEnd();

// 3. Custom Console Formatting
// Create visual separators
console.log('\n=== Server Starting ===\n');

// Color output (works in most terminals)
console.log('\x1b[32m%s\x1b[0m', 'Success!');  // Green text
console.log('\x1b[31m%s\x1b[0m', 'Error!');    // Red text
```

### Debugging with Node.js
```javascript
// Professional debugging techniques

// 1. Using Debug Package
const debug = require('debug');

// Create named debuggers
const dbDebug = debug('app:db');
const authDebug = debug('app:auth');

// Use in your code
dbDebug('Connected to database');
authDebug('User authenticated: %O', user);

// Enable in terminal:
// $ DEBUG=app:* node app.js
// $ DEBUG=app:db node app.js  (only database logs)

// 2. Node.js Inspector
// Start app with inspector:
// $ node --inspect app.js
// Then open Chrome DevTools

// Add debugger statements in code
function complexCalculation(x, y) {
    debugger;  // Code will pause here in DevTools
    const result = x * y / (x - y);
    return result;
}

// 3. Performance Monitoring
const { performance, PerformanceObserver } = require('perf_hooks');

// Create performance observer
const obs = new PerformanceObserver((list) => {
    const entries = list.getEntries();
    entries.forEach((entry) => {
        console.log(`${entry.name}: ${entry.duration}ms`);
    });
});
obs.observe({ entryTypes: ['measure'] });

// Measure performance
performance.mark('A');
// ... some code to measure
performance.mark('B');
performance.measure('Operation', 'A', 'B');
```




