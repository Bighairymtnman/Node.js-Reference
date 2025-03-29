
# Advanced Node.js Development Guide

## Table of Contents
- [Design Patterns](#design-patterns)
- [Advanced Architecture](#advanced-architecture)
- [Security Implementation](#security-implementation)
- [Performance Optimization](#performance-optimization)
- [Testing Strategies](#testing-strategies)
- [Microservices](#microservices)
- [Database Optimization](#database-optimization)
- [Production Deployment](#production-deployment)

## Design Patterns

### Singleton Pattern
```javascript
// The Singleton pattern ensures a class has only one instance
// Perfect for database connections, logging systems, or configuration managers
// This prevents resource waste and ensures consistency across your application

class Database {
    constructor() {
        // Check if we already have an instance
        if (Database.instance) {
            return Database.instance;
        }
        
        // If no instance exists, create one
        this.connection = null;
        Database.instance = this;
    }

    connect() {
        // Only create a connection if one doesn't exist
        // This saves resources and prevents multiple connections
        if (!this.connection) {
            this.connection = mongoose.connect(process.env.MONGO_URI, {
                useNewUrlParser: true,
                useUnifiedTopology: true
            });
            console.log('New database connection established');
        } else {
            console.log('Using existing database connection');
        }
        return this.connection;
    }
}

// Example usage showing how Singleton works
const db1 = new Database();
const db2 = new Database();
console.log(db1 === db2); // true - both variables reference the same instance
```

### Factory Pattern
```javascript
// The Factory pattern provides a way to create objects without explicitly specifying their exact class
// This is great when you need to create different types of related objects based on conditions

class LoggerFactory {
    // Central method to create different types of loggers
    // This makes it easy to add new logger types without changing existing code
    static createLogger(type, options = {}) {
        // Each case returns a different logger implementation
        switch(type) {
            case 'console':
                return new ConsoleLogger(options);
            case 'file':
                return new FileLogger(options.filename);
            case 'elastic':
                return new ElasticSearchLogger(options.host);
            default:
                throw new Error(`Logger type ${type} not supported`);
        }
    }
}

// Example implementations of different logger types
class ConsoleLogger {
    log(message) {
        console.log(`[${new Date().toISOString()}] ${message}`);
    }
}

class FileLogger {
    constructor(filename) {
        this.filename = filename;
    }
    
    log(message) {
        fs.appendFileSync(this.filename, 
            `[${new Date().toISOString()}] ${message}\n`);
    }
}

// Usage example showing the factory's flexibility
const productionLogger = LoggerFactory.createLogger('elastic', {
    host: 'elasticsearch:9200'
});
const developmentLogger = LoggerFactory.createLogger('console');
```

### Observer Pattern
```javascript
// The Observer pattern creates a subscription system where many objects (observers)
// can watch and react to events/changes in another object (subject)
// Perfect for event-driven architectures and decoupled systems

class EventManager {
    constructor() {
        // Store all event listeners in a Map
        // Map structure: { eventName => [callbacks] }
        this.listeners = new Map();
    }

    // Add a new event listener
    subscribe(event, callback) {
        // Create array for new event types
        if (!this.listeners.has(event)) {
            this.listeners.set(event, []);
        }
        this.listeners.get(event).push(callback);
        
        // Return unsubscribe function for cleanup
        return () => this.unsubscribe(event, callback);
    }

    // Remove an event listener
    unsubscribe(event, callback) {
        if (!this.listeners.has(event)) return;
        
        const callbacks = this.listeners.get(event);
        const index = callbacks.indexOf(callback);
        if (index !== -1) {
            callbacks.splice(index, 1);
        }
    }

    // Trigger an event
    emit(event, data) {
        if (this.listeners.has(event)) {
            // Call all callbacks registered for this event
            this.listeners.get(event).forEach(callback => {
                try {
                    callback(data);
                } catch (error) {
                    console.error(`Error in event listener for ${event}:`, error);
                }
            });
        }
    }
}

// Real-world example: Payment processing system
const events = new EventManager();

// Register multiple listeners for payment events
events.subscribe('payment_received', (data) => {
    notifyUser(data.userId);
});

events.subscribe('payment_received', (data) => {
    updateInventory(data.productId);
});

events.subscribe('payment_received', (data) => {
    generateInvoice(data.orderId);
});

// Trigger the payment event
events.emit('payment_received', {
    userId: 'user123',
    productId: 'prod456',
    orderId: 'ord789'
});
```

## Advanced Architecture

### Clean Architecture Implementation
```javascript
// Clean Architecture separates code into layers with clear responsibilities
// This makes the code more maintainable, testable, and framework-independent

// Domain Layer - Core business rules and entities
// These should have NO dependencies on external frameworks or libraries
class User {
    constructor(id, name, email) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.createdAt = new Date();
    }

    // Domain logic belongs here
    validate() {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        if (!emailRegex.test(this.email)) {
            throw new Error('Invalid email format');
        }
        if (this.name.length < 2) {
            throw new Error('Name must be at least 2 characters');
        }
    }
}

// Use Case Layer - Application specific business rules
// Orchestrates the flow of data and implements use case logic
class CreateUserUseCase {
    constructor(userRepository, emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }

    async execute(userData) {
        // Create and validate domain entity
        const user = new User(null, userData.name, userData.email);
        user.validate();
        
        // Check if user already exists
        const existingUser = await this.userRepository.findByEmail(user.email);
        if (existingUser) {
            throw new Error('Email already registered');
        }
        
        // Save user and send welcome email
        const savedUser = await this.userRepository.save(user);
        await this.emailService.sendWelcomeEmail(user.email);
        
        return savedUser;
    }
}

// Interface Adapters Layer - Converting data between use cases and external agencies
class ExpressUserController {
    constructor(createUserUseCase) {
        this.createUserUseCase = createUserUseCase;
    }

    // Express-specific adapter
    async handleCreateUser(req, res) {
        try {
            // Convert request data to use case format
            const userData = {
                name: req.body.name,
                email: req.body.email.toLowerCase()
            };

            const user = await this.createUserUseCase.execute(userData);
            
            // Convert domain data to API response format
            res.status(201).json({
                id: user.id,
                name: user.name,
                email: user.email,
                created: user.createdAt
            });
        } catch (error) {
            // Convert domain errors to HTTP responses
            if (error.message.includes('Invalid')) {
                res.status(400).json({ error: error.message });
            } else {
                res.status(500).json({ error: 'Internal server error' });
            }
        }
    }
}
```

### Dependency Injection
```javascript
// Dependency Injection makes code more modular and testable
// Instead of creating dependencies inside classes, we pass them in

class Container {
    constructor() {
        // Store service definitions and instances
        this.services = new Map();
        this.singletons = new Map();
    }

    // Register a service with its dependencies
    register(name, definition, dependencies = []) {
        this.services.set(name, { definition, dependencies });
        console.log(`Registered service: ${name}`);
    }

    // Register a service that should have only one instance
    singleton(name, definition, dependencies = []) {
        this.register(name, definition, dependencies);
        this.singletons.set(name, null);
        console.log(`Registered singleton: ${name}`);
    }

    // Get an instance of a service
    get(name) {
        const service = this.services.get(name);
        if (!service) {
            throw new Error(`Service not found: ${name}`);
        }

        // Return existing instance for singletons
        if (this.singletons.has(name)) {
            if (!this.singletons.get(name)) {
                const instance = this.createInstance(service);
                this.singletons.set(name, instance);
            }
            return this.singletons.get(name);
        }

        // Create new instance for regular services
        return this.createInstance(service);
    }

    // Create a new instance with dependencies
    createInstance(service) {
        // Recursively resolve dependencies
        const dependencies = service.dependencies.map(dep => this.get(dep));
        return new service.definition(...dependencies);
    }
}

// Example usage showing how DI simplifies testing and configuration
const container = new Container();

// Register services
container.singleton('database', DatabaseConnection);
container.register('userRepository', UserRepository, ['database']);
container.register('emailService', EmailService, ['emailConfig']);
container.register('createUserUseCase', CreateUserUseCase, 
    ['userRepository', 'emailService']);

// Get fully configured instance
const userUseCase = container.get('createUserUseCase');
```

## Security Implementation

### JWT Authentication
```javascript
// JSON Web Tokens (JWT) provide secure, stateless authentication
// Perfect for APIs and microservices architecture

const jwt = require('jsonwebtoken');

class JWTAuth {
    constructor(secret) {
        this.secret = secret;
        this.algorithms = ['HS256']; // Supported algorithms
    }

    // Generate tokens with configurable options
    generateToken(payload, options = {}) {
        const defaultOptions = {
            expiresIn: '24h',      // Token expiry
            algorithm: 'HS256',     // Signing algorithm
            audience: 'api:access', // Intended token use
            issuer: 'auth-service'  // Token issuer
        };

        return jwt.sign(
            payload, 
            this.secret,
            { ...defaultOptions, ...options }
        );
    }

    // Verify and decode tokens
    verifyToken(token) {
        try {
            const decoded = jwt.verify(token, this.secret, {
                algorithms: this.algorithms,
                audience: 'api:access',
                issuer: 'auth-service'
            });

            // Check additional claims if needed
            if (decoded.type !== 'access') {
                throw new Error('Invalid token type');
            }

            return decoded;
        } catch (error) {
            // Provide detailed error messages for different failure cases
            if (error.name === 'TokenExpiredError') {
                throw new AuthenticationError('Token has expired');
            }
            if (error.name === 'JsonWebTokenError') {
                throw new AuthenticationError('Invalid token');
            }
            throw error;
        }
    }

    // Express middleware for protecting routes
    middleware() {
        return async (req, res, next) => {
            try {
                // Extract token from Authorization header
                const authHeader = req.headers.authorization;
                if (!authHeader?.startsWith('Bearer ')) {
                    throw new AuthenticationError('No token provided');
                }

                const token = authHeader.split(' ')[1];
                const decoded = this.verifyToken(token);

                // Attach user info to request
                req.user = decoded;
                next();
            } catch (error) {
                // Return appropriate error responses
                if (error instanceof AuthenticationError) {
                    res.status(401).json({ error: error.message });
                } else {
                    res.status(500).json({ error: 'Authentication failed' });
                }
            }
        };
    }
}

// Example usage showing secure route protection
const auth = new JWTAuth(process.env.JWT_SECRET);

app.post('/login', async (req, res) => {
    // Generate token on successful login
    const token = auth.generateToken({
        userId: user.id,
        role: user.role,
        type: 'access'
    });
    res.json({ token });
});

// Protect sensitive routes
app.get('/profile', auth.middleware(), (req, res) => {
    // req.user contains decoded token data
    res.json({ profile: req.user });
});
```

### Rate Limiting
```javascript
// Prevent abuse and DOS attacks with intelligent rate limiting
// Uses Redis for distributed rate limiting across multiple servers

const Redis = require('ioredis');

class RateLimiter {
    constructor(options = {}) {
        this.redis = new Redis(options.redisUrl);
        
        // Configure rate limiting parameters
        this.windowMs = options.windowMs || 60000; // 1 minute window
        this.max = options.max || 100;  // Max requests per window
        this.blockDuration = options.blockDuration || 3600; // Block time in seconds
    }

    // Generate unique key for each client
    getKey(identifier) {
        const window = Math.floor(Date.now() / this.windowMs);
        return `ratelimit:${identifier}:${window}`;
    }

    async isRateLimited(identifier) {
        const key = this.getKey(identifier);
        
        // Use Redis transaction for atomic operations
        const multi = this.redis.multi();
        multi.incr(key);
        multi.expire(key, this.windowMs / 1000);

        const [requests] = await multi.exec();
        const currentRequests = requests[1];

        // Check if client is over limit
        if (currentRequests > this.max) {
            // Add client to blocklist if severely over limit
            if (currentRequests > this.max * 2) {
                await this.blockClient(identifier);
            }
            return true;
        }

        return false;
    }

    // Block abusive clients
    async blockClient(identifier) {
        const key = `ratelimit:blocked:${identifier}`;
        await this.redis.setex(key, this.blockDuration, '1');
    }

    // Express middleware implementation
    middleware() {
        return async (req, res, next) => {
            try {
                // Use IP and optional user ID for rate limiting
                const identifier = req.user?.id || req.ip;
                
                // Check if client is blocked
                const blocked = await this.redis.get(
                    `ratelimit:blocked:${identifier}`
                );
                if (blocked) {
                    return res.status(403).json({
                        error: 'Access denied due to abuse'
                    });
                }

                // Check rate limit
                const limited = await this.isRateLimited(identifier);
                if (limited) {
                    return res.status(429).json({
                        error: 'Too many requests',
                        retryAfter: this.windowMs / 1000
                    });
                }

                next();
            } catch (error) {
                console.error('Rate limiting error:', error);
                next(error);
            }
        };
    }
}

// Example usage showing rate limit protection
const limiter = new RateLimiter({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // limit each IP to 100 requests per windowMs
    blockDuration: 60 * 60 // Block for 1 hour if limit exceeded
});

// Apply rate limiting to all routes
app.use(limiter.middleware());
```






## Performance Optimization

### Memory Management
```javascript
// Advanced memory management and leak detection
// Critical for long-running Node.js applications

const heapdump = require('heapdump');
const monitor = require('monitor');

class MemoryMonitor {
    constructor(options = {}) {
        // Configure memory thresholds and monitoring intervals
        this.threshold = options.threshold || 1024 * 1024 * 1024; // 1GB
        this.checkInterval = options.interval || 1000; // 1 second
        this.heapDumpPath = options.dumpPath || './heapdumps';
        this.monitoring = false;
        
        // Track memory trends
        this.memoryTrend = [];
    }

    start() {
        this.monitoring = true;
        console.log('Memory monitoring started');
        this.monitor();
        
        // Set up garbage collection tracking
        this.trackGC();
    }

    async monitor() {
        while (this.monitoring) {
            const memoryUsage = process.memoryUsage();
            
            // Track memory usage over time
            this.memoryTrend.push({
                timestamp: Date.now(),
                heapUsed: memoryUsage.heapUsed,
                heapTotal: memoryUsage.heapTotal,
                external: memoryUsage.external
            });

            // Keep trend data for last hour only
            if (this.memoryTrend.length > 3600) {
                this.memoryTrend.shift();
            }

            // Check for memory issues
            if (memoryUsage.heapUsed > this.threshold) {
                await this.handleHighMemory(memoryUsage);
            }

            // Check for memory leaks
            if (this.detectMemoryLeak()) {
                await this.handleMemoryLeak();
            }
            
            await new Promise(resolve => setTimeout(resolve, this.checkInterval));
        }
    }

    // Detect potential memory leaks by analyzing trends
    detectMemoryLeak() {
        if (this.memoryTrend.length < 10) return false;

        const recentUsage = this.memoryTrend.slice(-10);
        let increasingCount = 0;

        for (let i = 1; i < recentUsage.length; i++) {
            if (recentUsage[i].heapUsed > recentUsage[i-1].heapUsed) {
                increasingCount++;
            }
        }

        // If memory consistently increases, might be a leak
        return increasingCount >= 8;
    }

    async handleHighMemory(memoryUsage) {
        // Generate heap snapshot for analysis
        const timestamp = new Date().toISOString();
        const dumpFile = `${this.heapDumpPath}/heapdump-${timestamp}.heapsnapshot`;
        
        await heapdump.writeSnapshot(dumpFile);

        // Alert monitoring system
        await monitor.alert('High memory usage detected', {
            used: memoryUsage.heapUsed,
            total: memoryUsage.heapTotal,
            threshold: this.threshold,
            dumpFile
        });

        // Attempt to free memory
        this.attemptMemoryRecovery();
    }

    // Track garbage collection events
    trackGC() {
        if (typeof gc === 'function') {
            const startTime = process.hrtime();
            
            gc(); // Force garbage collection
            
            const diff = process.hrtime(startTime);
            const duration = (diff[0] * 1e9 + diff[1]) / 1e6; // Convert to ms
            
            console.log(`Garbage collection took ${duration}ms`);
        }
    }

    attemptMemoryRecovery() {
        // Clear internal caches
        this.memoryTrend = this.memoryTrend.slice(-60); // Keep last minute only
        
        // Force garbage collection if available
        if (typeof gc === 'function') {
            gc();
        }
    }
}

// Example usage with detailed monitoring
const memoryMonitor = new MemoryMonitor({
    threshold: 800 * 1024 * 1024, // 800MB
    interval: 5000, // Check every 5 seconds
    dumpPath: '/var/log/heapdumps'
});

memoryMonitor.start();
```

### Caching Strategies
```javascript
// Implement multi-level caching for optimal performance
// Combines in-memory, distributed, and database caching

const Redis = require('ioredis');
const NodeCache = require('node-cache');

class CacheManager {
    constructor(options = {}) {
        // Initialize different cache layers
        this.localCache = new NodeCache({
            stdTTL: 60,            // 1 minute default TTL
            checkperiod: 120,      // Cleanup every 2 minutes
            maxKeys: 1000          // Prevent memory issues
        });
        
        this.redisCache = new Redis(options.redisUrl);
        
        // Cache statistics
        this.stats = {
            hits: { local: 0, redis: 0 },
            misses: { local: 0, redis: 0 }
        };
    }

    // Smart key generation with versioning
    generateKey(key, version = 'v1') {
        return `cache:${version}:${key}`;
    }

    async get(key, options = {}) {
        const cacheKey = this.generateKey(key, options.version);

        // Try local cache first (fastest)
        const localValue = this.localCache.get(cacheKey);
        if (localValue) {
            this.stats.hits.local++;
            return localValue;
        }
        this.stats.misses.local++;

        // Try Redis cache
        const redisValue = await this.redisCache.get(cacheKey);
        if (redisValue) {
            // Update local cache
            this.localCache.set(cacheKey, redisValue);
            this.stats.hits.redis++;
            return JSON.parse(redisValue);
        }
        this.stats.misses.redis++;

        return null;
    }

    async set(key, value, options = {}) {
        const cacheKey = this.generateKey(key, options.version);
        const ttl = options.ttl || 3600; // 1 hour default

        // Set in both caches
        this.localCache.set(cacheKey, value, ttl);
        await this.redisCache.setex(
            cacheKey, 
            ttl,
            JSON.stringify(value)
        );

        // Optional: Set cache tags for group invalidation
        if (options.tags) {
            await this.setTags(cacheKey, options.tags);
        }
    }

    // Group related cache entries with tags
    async setTags(key, tags) {
        for (const tag of tags) {
            await this.redisCache.sadd(`cachetag:${tag}`, key);
        }
    }

    // Invalidate cache by tags
    async invalidateByTag(tag) {
        const keys = await this.redisCache.smembers(`cachetag:${tag}`);
        if (keys.length > 0) {
            // Remove from both caches
            this.localCache.del(keys);
            await this.redisCache.del(keys);
            // Clean up tag
            await this.redisCache.del(`cachetag:${tag}`);
        }
    }

    // Get cache statistics
    getStats() {
        const totalHits = this.stats.hits.local + this.stats.hits.redis;
        const totalMisses = this.stats.misses.local + this.stats.misses.redis;
        const hitRate = totalHits / (totalHits + totalMisses) * 100;

        return {
            hits: this.stats.hits,
            misses: this.stats.misses,
            hitRate: `${hitRate.toFixed(2)}%`
        };
    }
}

// Example usage showing advanced caching patterns
const cache = new CacheManager({
    redisUrl: 'redis://localhost:6379'
});

// Cache with tags for category-based invalidation
await cache.set('product:123', productData, {
    ttl: 1800,  // 30 minutes
    tags: ['products', 'category:electronics'],
    version: 'v2'
});

// Invalidate all products in a category
await cache.invalidateByTag('category:electronics');

// Monitor cache effectiveness
console.log('Cache Statistics:', cache.getStats());
```


### Worker Threads
```javascript
// Worker Threads enable true parallel processing in Node.js
// Perfect for CPU-intensive tasks and improving application performance

const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

class WorkerPool {
    constructor(numThreads) {
        // Initialize worker pool
        this.workers = [];
        this.freeWorkers = [];
        this.queue = [];
        
        // Create worker threads
        for (let i = 0; i < numThreads; i++) {
            this.addNewWorker();
        }
        
        // Track pool statistics
        this.stats = {
            completedTasks: 0,
            failedTasks: 0,
            averageProcessingTime: 0
        };
    }

    addNewWorker() {
        // Create a new worker with error handling and monitoring
        const worker = new Worker(`
            const { parentPort } = require('worker_threads');

            // Listen for tasks from main thread
            parentPort.on('message', async (task) => {
                try {
                    const startTime = Date.now();
                    
                    // Execute the task
                    const result = await task();
                    
                    // Send back result and execution metrics
                    parentPort.postMessage({
                        success: true,
                        result,
                        executionTime: Date.now() - startTime
                    });
                } catch (error) {
                    parentPort.postMessage({
                        success: false,
                        error: error.message
                    });
                }
            });
        `);

        // Handle worker lifecycle events
        worker.on('online', () => {
            console.log(`Worker #${worker.threadId} is online`);
        });

        worker.on('error', (error) => {
            console.error(`Worker #${worker.threadId} error:`, error);
            this.handleWorkerError(worker);
        });

        worker.on('exit', (code) => {
            if (code !== 0) {
                console.error(`Worker #${worker.threadId} exited with code ${code}`);
                this.replaceWorker(worker);
            }
        });

        this.workers.push(worker);
        this.freeWorkers.push(worker);
    }

    async executeTask(task) {
        return new Promise((resolve, reject) => {
            const startTime = Date.now();

            // Get an available worker or queue the task
            const worker = this.freeWorkers.pop();
            if (worker) {
                this.runTask(worker, task, startTime, resolve, reject);
            } else {
                this.queue.push({ task, resolve, reject });
            }
        });
    }

    runTask(worker, task, startTime, resolve, reject) {
        // Set up task completion handler
        const messageHandler = (message) => {
            // Clean up event listener
            worker.removeListener('message', messageHandler);
            
            // Return worker to pool
            this.freeWorkers.push(worker);
            
            // Process next queued task if any
            if (this.queue.length > 0) {
                const nextTask = this.queue.shift();
                this.runTask(
                    worker, 
                    nextTask.task, 
                    Date.now(), 
                    nextTask.resolve, 
                    nextTask.reject
                );
            }

            // Update statistics
            const processingTime = Date.now() - startTime;
            this.updateStats(message.success, processingTime);

            // Handle task result
            if (message.success) {
                resolve(message.result);
            } else {
                reject(new Error(message.error));
            }
        };

        // Send task to worker
        worker.once('message', messageHandler);
        worker.postMessage(task);
    }

    updateStats(success, processingTime) {
        if (success) {
            this.stats.completedTasks++;
            // Update moving average of processing time
            this.stats.averageProcessingTime = 
                (this.stats.averageProcessingTime * (this.stats.completedTasks - 1) + processingTime) 
                / this.stats.completedTasks;
        } else {
            this.stats.failedTasks++;
        }
    }

    handleWorkerError(worker) {
        // Remove failed worker from pools
        this.workers = this.workers.filter(w => w !== worker);
        this.freeWorkers = this.freeWorkers.filter(w => w !== worker);
        
        // Replace the worker
        this.addNewWorker();
    }

    replaceWorker(worker) {
        this.handleWorkerError(worker);
    }

    // Get pool statistics
    getStats() {
        return {
            ...this.stats,
            activeWorkers: this.workers.length - this.freeWorkers.length,
            queueLength: this.queue.length
        };
    }
}

// Example usage showing CPU-intensive task distribution
const pool = new WorkerPool(4); // Create pool with 4 workers

// CPU-intensive calculation function
function fibonacci(n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// Distribute multiple calculations across worker pool
async function calculateFibonacci() {
    const numbers = [40, 42, 41, 43];
    const results = await Promise.all(
        numbers.map(n => 
            pool.executeTask(() => fibonacci(n))
        )
    );
    
    console.log('Results:', results);
    console.log('Pool Stats:', pool.getStats());
}

calculateFibonacci().catch(console.error);
```

## Testing Strategies

### Integration Testing
```javascript
// Comprehensive integration testing setup with Jest and Supertest
// Tests real interactions between components and external services

const request = require('supertest');
const { MongoMemoryServer } = require('mongodb-memory-server');
const app = require('../app');

describe('User API Integration Tests', () => {
    let mongoServer;
    let authToken;
    
    // Global test setup
    beforeAll(async () => {
        // Setup in-memory MongoDB for isolated testing
        mongoServer = await MongoMemoryServer.create();
        const mongoUri = mongoServer.getUri();
        await mongoose.connect(mongoUri);

        // Create test user and get auth token
        authToken = await generateTestUserToken();
    });

    // Cleanup after tests
    afterAll(async () => {
        await mongoose.disconnect();
        await mongoServer.stop();
    });

    // Reset database between tests
    beforeEach(async () => {
        await mongoose.connection.dropDatabase();
    });

    // User Creation Tests
    describe('POST /api/users', () => {
        it('should create a new user and trigger welcome email', async () => {
            // Mock email service
            const emailService = require('../services/email');
            const sendEmailSpy = jest.spyOn(emailService, 'sendWelcomeEmail');

            const userData = {
                name: 'Test User',
                email: 'test@example.com',
                password: 'SecurePass123!'
            };

            const response = await request(app)
                .post('/api/users')
                .send(userData)
                .expect(201);

            // Assert response structure
            expect(response.body).toMatchObject({
                id: expect.any(String),
                name: userData.name,
                email: userData.email,
                createdAt: expect.any(String)
            });

            // Verify email was triggered
            expect(sendEmailSpy).toHaveBeenCalledWith(userData.email);

            // Verify user in database
            const user = await User.findById(response.body.id);
            expect(user).toBeTruthy();
            expect(user.password).not.toBe(userData.password); // Should be hashed
        });

        // Test validation and error cases
        it('should handle duplicate email addresses', async () => {
            // Create initial user
            await User.create({
                name: 'Existing User',
                email: 'test@example.com',
                password: 'password123'
            });

            // Try to create user with same email
            const response = await request(app)
                .post('/api/users')
                .send({
                    name: 'New User',
                    email: 'test@example.com',
                    password: 'newpassword123'
                })
                .expect(400);

            expect(response.body).toEqual({
                error: 'Email already exists'
            });
        });
    });

    // Authentication Tests
    describe('POST /api/auth/login', () => {
        beforeEach(async () => {
            // Create test user
            await User.create({
                name: 'Test User',
                email: 'test@example.com',
                password: await bcrypt.hash('password123', 10)
            });
        });

        it('should authenticate valid credentials', async () => {
            const response = await request(app)
                .post('/api/auth/login')
                .send({
                    email: 'test@example.com',
                    password: 'password123'
                })
                .expect(200);

            expect(response.body).toHaveProperty('token');
            expect(response.body.token).toMatch(/^[\w-]*\.[\w-]*\.[\w-]*$/); // JWT format
        });
    });
});

// Custom test utilities
class TestDatabase {
    static async populate(data) {
        for (const [model, documents] of Object.entries(data)) {
            await mongoose.model(model).insertMany(documents);
        }
    }

    static async cleanup() {
        const collections = await mongoose.connection.db.collections();
        for (const collection of collections) {
            await collection.deleteMany({});
        }
    }
}

// Mock Factory for test data
class TestFactory {
    static user(overrides = {}) {
        return {
            name: 'Test User',
            email: `test-${Date.now()}@example.com`,
            password: 'password123',
            ...overrides
        };
    }

    static async createUser(overrides = {}) {
        const userData = this.user(overrides);
        return await User.create({
            ...userData,
            password: await bcrypt.hash(userData.password, 10)
        });
    }
}
```

### Load Testing
```javascript
// Advanced load testing setup with Artillery
// Tests system performance under various conditions

const artillery = require('artillery');

class LoadTester {
    constructor(config) {
        this.config = {
            target: 'http://localhost:3000',
            phases: [
                // Ramp up phase
                { duration: 60, arrivalRate: 5, name: 'Warm up' },
                // Sustained load phase
                { duration: 120, arrivalRate: 10, name: 'Sustained load' },
                // Spike phase
                { duration: 30, arrivalRate: 50, name: 'Spike' },
                // Cool down phase
                { duration: 60, arrivalRate: 5, name: 'Cool down' }
            ],
            ...config
        };
    }

    async runTest() {
        const script = {
            config: this.config,
            scenarios: [{
                name: 'API Load Test',
                flow: [
                    // Login flow
                    { 
                        post: {
                            url: '/api/auth/login',
                            json: {
                                email: '{{ $randomEmail }}',
                                password: 'testpass123'
                            },
                            capture: {
                                json: '$.token',
                                as: 'authToken'
                            }
                        }
                    },
                    // Use token in subsequent requests
                    {
                        get: {
                            url: '/api/users/profile',
                            headers: {
                                Authorization: 'Bearer {{ authToken }}'
                            }
                        }
                    },
                    // Simulate user actions
                    {
                        post: {
                            url: '/api/orders',
                            headers: {
                                Authorization: 'Bearer {{ authToken }}'
                            },
                            json: {
                                productId: '{{ $randomProductId }}',
                                quantity: '{{ $randomInt(1,5) }}'
                            }
                        }
                    }
                ]
            }]
        };

        const results = await this.runArtilleryTest(script);
        return this.analyzeResults(results);
    }

    async runArtilleryTest(script) {
        return new Promise((resolve, reject) => {
            artillery.run(script, (err, results) => {
                if (err) reject(err);
                resolve(results);
            });
        });
    }

    analyzeResults(results) {
        const metrics = {
            totalRequests: results.aggregate.requestsCompleted,
            avgResponseTime: results.aggregate.latency.mean,
            p95ResponseTime: results.aggregate.latency.p95,
            errorRate: (results.aggregate.errors / results.aggregate.requestsCompleted) * 100,
            scenariosCompleted: results.aggregate.scenariosCompleted,
            scenariosFailed: results.aggregate.scenariosFailed
        };

        // Performance thresholds
        const thresholds = {
            avgResponseTime: 200,  // 200ms
            errorRate: 1,         // 1%
            p95ResponseTime: 500  // 500ms
        };

        // Check if results meet thresholds
        const issues = [];
        if (metrics.avgResponseTime > thresholds.avgResponseTime) {
            issues.push(`High average response time: ${metrics.avgResponseTime}ms`);
        }
        if (metrics.errorRate > thresholds.errorRate) {
            issues.push(`High error rate: ${metrics.errorRate}%`);
        }
        if (metrics.p95ResponseTime > thresholds.p95ResponseTime) {
            issues.push(`High P95 response time: ${metrics.p95ResponseTime}ms`);
        }

        return {
            metrics,
            issues,
            passed: issues.length === 0
        };
    }
}

// Example usage
const loadTester = new LoadTester({
    target: 'https://api.production.com',
    phases: [
        { duration: 300, arrivalRate: 10, rampTo: 50, name: 'Ramp up load' }
    ]
});

loadTester.runTest()
    .then(results => {
        console.log('Load Test Results:', results);
        if (!results.passed) {
            console.error('Performance issues detected:', results.issues);
        }
    });
```


## Microservices

### Service Discovery
```javascript
// Implement service discovery and registration using Consul
const Consul = require('consul');

class ServiceRegistry {
    constructor(config = {}) {
        this.consul = new Consul({
            host: config.host || 'localhost',
            port: config.port || 8500,
            promisify: true
        });
        
        this.serviceId = `${config.serviceName}-${Date.now()}`;
        this.serviceName = config.serviceName;
        this.servicePort = config.servicePort;
        
        // Track discovered services
        this.servicesCache = new Map();
        this.startCacheRefresh();
    }

    // Register current service
    async register() {
        const registration = {
            id: this.serviceId,
            name: this.serviceName,
            address: process.env.SERVICE_HOST || 'localhost',
            port: this.servicePort,
            tags: ['node', 'api'],
            check: {
                http: `http://localhost:${this.servicePort}/health`,
                interval: '10s',
                timeout: '5s'
            }
        };

        await this.consul.agent.service.register(registration);
        console.log(`Service registered: ${this.serviceName}`);
        
        // Graceful shutdown
        this.handleShutdown();
    }

    // Discover available services
    async discover(serviceName) {
        // Check cache first
        if (this.servicesCache.has(serviceName)) {
            return this.servicesCache.get(serviceName);
        }

        const services = await this.consul.catalog.service.nodes(serviceName);
        const nodes = services.map(node => ({
            id: node.ServiceID,
            address: node.ServiceAddress,
            port: node.ServicePort
        }));

        // Update cache
        this.servicesCache.set(serviceName, nodes);
        return nodes;
    }

    // Keep service cache fresh
    startCacheRefresh() {
        setInterval(async () => {
            for (const [serviceName] of this.servicesCache) {
                const services = await this.consul.catalog.service.nodes(serviceName);
                this.servicesCache.set(serviceName, services);
            }
        }, 30000); // Refresh every 30 seconds
    }

    // Handle graceful shutdown
    handleShutdown() {
        const shutdown = async () => {
            console.log('Deregistering service...');
            await this.consul.agent.service.deregister(this.serviceId);
            process.exit(0);
        };

        process.on('SIGTERM', shutdown);
        process.on('SIGINT', shutdown);
    }
}

// Example usage
const registry = new ServiceRegistry({
    serviceName: 'user-service',
    servicePort: 3000
});

registry.register().then(() => {
    console.log('Service registry initialized');
});
```

### Message Queue Integration
```javascript
// Implement reliable message queuing with RabbitMQ
const amqp = require('amqplib');

class MessageQueue {
    constructor(config = {}) {
        this.url = config.url || 'amqp://localhost';
        this.exchanges = new Map();
        this.connection = null;
        this.channel = null;
    }

    async connect() {
        try {
            this.connection = await amqp.connect(this.url);
            this.channel = await this.connection.createChannel();
            
            // Handle connection issues
            this.connection.on('error', this.handleConnectionError.bind(this));
            this.connection.on('close', this.handleConnectionClose.bind(this));
            
            console.log('Connected to RabbitMQ');
        } catch (error) {
            console.error('Failed to connect to RabbitMQ:', error);
            throw error;
        }
    }

    async createExchange(name, type = 'topic') {
        await this.channel.assertExchange(name, type, {
            durable: true,
            autoDelete: false
        });
        this.exchanges.set(name, type);
    }

    async publish(exchange, routingKey, message, options = {}) {
        if (!this.exchanges.has(exchange)) {
            await this.createExchange(exchange);
        }

        const messageBuffer = Buffer.from(JSON.stringify(message));
        
        return this.channel.publish(exchange, routingKey, messageBuffer, {
            persistent: true,
            ...options
        });
    }

    async subscribe(exchange, pattern, handler, queueOptions = {}) {
        if (!this.exchanges.has(exchange)) {
            await this.createExchange(exchange);
        }

        // Create queue with options
        const queue = await this.channel.assertQueue('', {
            exclusive: true,
            ...queueOptions
        });

        // Bind queue to exchange
        await this.channel.bindQueue(queue.queue, exchange, pattern);

        // Start consuming messages
        await this.channel.consume(queue.queue, async (msg) => {
            if (!msg) return;

            try {
                const content = JSON.parse(msg.content.toString());
                await handler(content);
                this.channel.ack(msg);
            } catch (error) {
                console.error('Error processing message:', error);
                // Requeue message if processing failed
                this.channel.nack(msg, false, true);
            }
        });
    }

    handleConnectionError(error) {
        console.error('RabbitMQ connection error:', error);
        this.reconnect();
    }

    handleConnectionClose() {
        console.log('RabbitMQ connection closed');
        this.reconnect();
    }

    async reconnect() {
        try {
            await this.connect();
            // Reestablish exchanges and subscriptions
            for (const [exchange, type] of this.exchanges) {
                await this.createExchange(exchange, type);
            }
        } catch (error) {
            console.error('Failed to reconnect:', error);
            setTimeout(() => this.reconnect(), 5000);
        }
    }
}

// Example usage
const messageQueue = new MessageQueue({
    url: process.env.RABBITMQ_URL
});

// Initialize messaging system
async function initializeMessaging() {
    await messageQueue.connect();

    // Set up order processing
    await messageQueue.subscribe(
        'orders',
        'order.created.*',
        async (order) => {
            console.log('Processing new order:', order.id);
            await processOrder(order);
        }
    );

    // Publish messages
    await messageQueue.publish(
        'orders',
        'order.created.high_priority',
        {
            id: 'ord_123',
            customer: 'cust_456',
            items: ['item1', 'item2']
        }
    );
}

initializeMessaging();
```




## Database Optimization

### Advanced Query Optimization
```javascript
// Implement sophisticated database query optimization techniques
const mongoose = require('mongoose');

class DatabaseOptimizer {
    constructor(model) {
        this.model = model;
        this.queryStats = new Map();
        this.indexAnalyzer = new IndexAnalyzer(model);
    }

    // Smart query execution with automatic optimization
    async optimizedQuery(query, options = {}) {
        const queryHash = this.generateQueryHash(query);
        
        // Track query performance
        const startTime = Date.now();
        
        try {
            // Apply query optimizations
            const optimizedQuery = this.optimizeQuery(query);
            
            // Execute with lean for better performance
            const result = await this.model
                .find(optimizedQuery)
                .lean()
                .hint(this.indexAnalyzer.suggestIndex(query))
                .exec();

            // Record query statistics
            this.recordQueryStats(queryHash, Date.now() - startTime);

            return result;
        } catch (error) {
            console.error('Query execution error:', error);
            throw error;
        }
    }

    // Optimize query structure
    optimizeQuery(query) {
        const optimized = { ...query };

        // Convert regex to text index where possible
        if (optimized.$or) {
            optimized.$or = optimized.$or.map(condition => 
                this.convertRegexToTextSearch(condition)
            );
        }

        // Use range queries efficiently
        if (optimized.createdAt) {
            optimized.createdAt = this.optimizeDateRange(optimized.createdAt);
        }

        return optimized;
    }

    // Efficient batch processing
    async batchProcess(query, batchSize = 1000, processor) {
        let lastId = null;
        let processed = 0;
        
        while (true) {
            const batchQuery = lastId 
                ? { ...query, _id: { $gt: lastId }}
                : query;

            const batch = await this.model
                .find(batchQuery)
                .sort({ _id: 1 })
                .limit(batchSize)
                .lean();

            if (batch.length === 0) break;

            // Process batch
            await processor(batch);
            
            lastId = batch[batch.length - 1]._id;
            processed += batch.length;

            console.log(`Processed ${processed} documents`);
        }

        return processed;
    }

    // Analyze and create optimal indexes
    async analyzeIndexes() {
        const indexes = await this.model.collection.indexes();
        const queryPatterns = Array.from(this.queryStats.entries())
            .sort((a, b) => b[1].avgTime - a[1].avgTime)
            .slice(0, 10);

        return this.indexAnalyzer.suggestIndexImprovements(
            indexes,
            queryPatterns
        );
    }

    // Monitor query performance
    recordQueryStats(queryHash, executionTime) {
        const stats = this.queryStats.get(queryHash) || {
            count: 0,
            totalTime: 0,
            avgTime: 0
        };

        stats.count++;
        stats.totalTime += executionTime;
        stats.avgTime = stats.totalTime / stats.count;

        this.queryStats.set(queryHash, stats);
    }
}

// Index Analysis and Optimization
class IndexAnalyzer {
    constructor(model) {
        this.model = model;
    }

    // Suggest optimal indexes based on query patterns
    suggestIndex(query) {
        const fields = this.extractQueryFields(query);
        return this.createCompoundIndex(fields);
    }

    // Analyze existing indexes and suggest improvements
    async suggestIndexImprovements(existingIndexes, queryPatterns) {
        const suggestions = [];

        for (const [queryHash, stats] of queryPatterns) {
            if (stats.avgTime > 100) { // Slow queries
                const fields = this.extractQueryFields(
                    this.parseQueryHash(queryHash)
                );
                
                if (!this.isIndexCovered(fields, existingIndexes)) {
                    suggestions.push({
                        fields,
                        avgQueryTime: stats.avgTime,
                        queryCount: stats.count
                    });
                }
            }
        }

        return suggestions;
    }

    // Create optimal compound indexes
    async createCompoundIndex(fields) {
        const index = {};
        fields.forEach(field => {
            index[field] = 1;
        });

        try {
            await this.model.collection.createIndex(index);
            console.log('Created compound index:', index);
        } catch (error) {
            console.error('Index creation failed:', error);
        }
    }
}

// Example usage
const userOptimizer = new DatabaseOptimizer(User);

// Optimize complex queries
const results = await userOptimizer.optimizedQuery({
    age: { $gte: 18 },
    status: 'active',
    $or: [
        { email: /gmail\.com$/ },
        { email: /yahoo\.com$/ }
    ]
});

// Batch process large datasets
await userOptimizer.batchProcess(
    { status: 'pending' },
    1000,
    async (batch) => {
        for (const user of batch) {
            await processUser(user);
        }
    }
);

// Analyze and optimize indexes
const indexSuggestions = await userOptimizer.analyzeIndexes();
console.log('Index Improvement Suggestions:', indexSuggestions);
```

### Connection Pool Management
```javascript
// Advanced connection pool management for optimal database performance
class ConnectionPoolManager {
    constructor(config = {}) {
        this.config = {
            maxPoolSize: 10,
            minPoolSize: 2,
            maxIdleTimeMS: 30000,
            ...config
        };

        this.stats = {
            activeConnections: 0,
            idleConnections: 0,
            waitingRequests: 0
        };
    }

    async initialize() {
        await mongoose.connect(process.env.MONGODB_URI, {
            maxPoolSize: this.config.maxPoolSize,
            minPoolSize: this.config.minPoolSize,
            maxIdleTimeMS: this.config.maxIdleTimeMS,
            serverSelectionTimeoutMS: 5000,
            socketTimeoutMS: 45000,
            family: 4
        });

        this.monitorPool();
    }

    // Monitor connection pool health
    monitorPool() {
        setInterval(() => {
            const pool = mongoose.connection.client.topology.s.pool;
            
            this.stats = {
                activeConnections: pool.totalConnectionCount - pool.availableConnectionCount,
                idleConnections: pool.availableConnectionCount,
                waitingRequests: pool.waitingRequests
            };

            this.adjustPool();
            this.logPoolStats();
        }, 5000);
    }

    // Dynamically adjust pool size based on load
    adjustPool() {
        const currentLoad = this.stats.activeConnections / this.config.maxPoolSize;
        
        if (currentLoad > 0.8 && this.config.maxPoolSize < 20) {
            // Increase pool size under high load
            this.config.maxPoolSize += 2;
            this.updatePoolSize();
        } else if (currentLoad < 0.3 && this.config.maxPoolSize > 10) {
            // Decrease pool size under light load
            this.config.maxPoolSize -= 2;
            this.updatePoolSize();
        }
    }

    // Update MongoDB pool size
    async updatePoolSize() {
        try {
            await mongoose.connection.client.topology.s.pool
                .setMaxPoolSize(this.config.maxPoolSize);
            
            console.log(`Pool size updated to ${this.config.maxPoolSize}`);
        } catch (error) {
            console.error('Failed to update pool size:', error);
        }
    }

    // Log pool statistics
    logPoolStats() {
        console.log('Connection Pool Stats:', {
            ...this.stats,
            maxPoolSize: this.config.maxPoolSize,
            utilizationRate: (
                this.stats.activeConnections / this.config.maxPoolSize * 100
            ).toFixed(2) + '%'
        });
    }
}

// Initialize pool manager
const poolManager = new ConnectionPoolManager({
    maxPoolSize: 10,
    minPoolSize: 2,
    maxIdleTimeMS: 30000
});

poolManager.initialize();
```

## Production Deployment

### Advanced Deployment Pipeline
```javascript
// Implement a sophisticated deployment pipeline with zero-downtime updates
const pm2 = require('pm2');
const Docker = require('dockerode');
const k8s = require('@kubernetes/client-node');

class DeploymentManager {
    constructor(config = {}) {
        this.docker = new Docker();
        this.k8sClient = new k8s.KubeConfig();
        this.k8sClient.loadFromDefault();
        
        this.config = {
            appName: 'node-service',
            version: process.env.VERSION,
            replicas: 3,
            ...config
        };
    }

    // Orchestrate the deployment process
    async deploy() {
        console.log(`Starting deployment of ${this.config.appName} v${this.config.version}`);

        try {
            // Pre-deployment checks
            await this.runPreflightChecks();
            
            // Build and push new container
            const imageTag = await this.buildContainer();
            
            // Deploy to Kubernetes
            await this.k8sDeploy(imageTag);
            
            // Run post-deployment verification
            await this.verifyDeployment();

            console.log('Deployment completed successfully');
        } catch (error) {
            console.error('Deployment failed:', error);
            await this.rollback();
        }
    }

    async runPreflightChecks() {
        const checks = [
            this.checkDiskSpace(),
            this.checkMemory(),
            this.checkDependencies(),
            this.runTests()
        ];

        await Promise.all(checks);
        console.log('Preflight checks passed');
    }

    async buildContainer() {
        const imageTag = `${this.config.appName}:${this.config.version}`;
        
        console.log('Building container image...');
        await this.docker.buildImage({
            context: process.cwd(),
            src: ['Dockerfile', 'package.json', 'src/']
        }, { t: imageTag });

        return imageTag;
    }

    async k8sDeploy(imageTag) {
        const k8sApi = this.k8sClient.makeApiClient(k8s.AppsV1Api);
        
        // Create deployment configuration
        const deployment = {
            apiVersion: 'apps/v1',
            kind: 'Deployment',
            metadata: { name: this.config.appName },
            spec: {
                replicas: this.config.replicas,
                selector: {
                    matchLabels: { app: this.config.appName }
                },
                template: {
                    metadata: {
                        labels: { app: this.config.appName }
                    },
                    spec: {
                        containers: [{
                            name: this.config.appName,
                            image: imageTag,
                            ports: [{ containerPort: 3000 }],
                            readinessProbe: {
                                httpGet: {
                                    path: '/health',
                                    port: 3000
                                }
                            }
                        }]
                    }
                }
            }
        };

        // Apply deployment with rolling update
        await k8sApi.createNamespacedDeployment(
            'default',
            deployment
        );
    }

    async verifyDeployment() {
        // Monitor deployment status
        return new Promise((resolve, reject) => {
            const checkInterval = setInterval(async () => {
                const status = await this.getDeploymentStatus();
                
                if (status.readyReplicas === this.config.replicas) {
                    clearInterval(checkInterval);
                    resolve();
                }
                
                if (status.failedReplicas > 0) {
                    clearInterval(checkInterval);
                    reject(new Error('Deployment failed'));
                }
            }, 5000);
        });
    }
}

### Production Monitoring
```javascript
// Implement comprehensive production monitoring
class ProductionMonitor {
    constructor() {
        this.metrics = new Map();
        this.alerts = new AlertManager();
        this.startTime = Date.now();
    }

    // Track application metrics
    trackMetric(name, value, tags = {}) {
        const metric = {
            value,
            timestamp: Date.now(),
            tags
        };

        if (!this.metrics.has(name)) {
            this.metrics.set(name, []);
        }

        this.metrics.get(name).push(metric);
        this.checkThresholds(name, value, tags);
    }

    // Monitor system health
    async checkHealth() {
        const health = {
            uptime: Date.now() - this.startTime,
            memory: process.memoryUsage(),
            cpu: process.cpuUsage(),
            connections: await this.getConnectionCount(),
            lastError: this.lastError
        };

        this.trackMetric('system.health', health);
        return health;
    }

    // Set up alert thresholds
    setThreshold(metricName, threshold, callback) {
        this.alerts.addRule(metricName, threshold, callback);
    }

    // Check metric thresholds
    checkThresholds(name, value, tags) {
        this.alerts.check(name, value, tags);
    }

    // Export metrics for monitoring systems
    async exportMetrics() {
        const metrics = Array.from(this.metrics.entries())
            .map(([name, values]) => ({
                name,
                values: values.slice(-100) // Keep last 100 values
            }));

        await Promise.all([
            this.exportToPrometheus(metrics),
            this.exportToCloudWatch(metrics)
        ]);
    }
}

// Alert management system
class AlertManager {
    constructor() {
        this.rules = new Map();
        this.notifications = new NotificationService();
    }

    addRule(metricName, threshold, callback) {
        this.rules.set(metricName, { threshold, callback });
    }

    async check(metricName, value, tags) {
        const rule = this.rules.get(metricName);
        if (!rule) return;

        if (value > rule.threshold) {
            await this.notifications.send({
                level: 'warning',
                message: `Metric ${metricName} exceeded threshold`,
                value,
                threshold: rule.threshold,
                tags
            });

            await rule.callback(value, tags);
        }
    }
}

// Initialize monitoring
const monitor = new ProductionMonitor();

// Set up monitoring rules
monitor.setThreshold('memory.heapUsed', 1024 * 1024 * 1024, async (value) => {
    await restartService();
});

monitor.setThreshold('http.responseTime', 1000, async (value, tags) => {
    await scaleService(tags.service);
});

// Start monitoring loop
setInterval(() => {
    monitor.checkHealth();
    monitor.exportMetrics();
}, 60000);

