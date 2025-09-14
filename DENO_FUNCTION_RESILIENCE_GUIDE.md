# 🛡️ **Resilient Deno Function Design Patterns**

## 📋 **Current State Analysis**

Your Deno functions already implement several resilience patterns, but there are opportunities to enhance them further. Here's a comprehensive guide for ensuring continuous availability and health.

---

## 🔧 **1. Circuit Breaker Pattern**

### **Current Implementation:**
```javascript
// src/api/asyncOptimizer.js - Basic circuit breaker
class CircuitBreaker {
  constructor(threshold = 5, timeout = 60000) {
    this.failureThreshold = threshold;
    this.timeout = timeout;
    this.failureCount = 0;
    this.lastFailureTime = null;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
  }
}
```

### **Enhanced Circuit Breaker:**
```javascript
// Enhanced circuit breaker with health monitoring
class EnhancedCircuitBreaker {
  constructor(serviceName, options = {}) {
    this.serviceName = serviceName;
    this.failureThreshold = options.failureThreshold || 5;
    this.timeout = options.timeout || 60000;
    this.successThreshold = options.successThreshold || 3;
    this.failureCount = 0;
    this.successCount = 0;
    this.lastFailureTime = null;
    this.state = 'CLOSED';
    this.metrics = {
      totalRequests: 0,
      successfulRequests: 0,
      failedRequests: 0,
      averageResponseTime: 0
    };
  }

  async execute(operation) {
    this.metrics.totalRequests++;
    const startTime = Date.now();

    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.timeout) {
        this.state = 'HALF_OPEN';
        this.successCount = 0;
      } else {
        throw new Error(`Circuit breaker for ${this.serviceName} is OPEN`);
      }
    }

    try {
      const result = await operation();
      this.onSuccess(Date.now() - startTime);
      return result;
    } catch (error) {
      this.onFailure(Date.now() - startTime);
      throw error;
    }
  }

  onSuccess(responseTime) {
    this.metrics.successfulRequests++;
    this.updateAverageResponseTime(responseTime);
    
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.successThreshold) {
        this.state = 'CLOSED';
        this.failureCount = 0;
      }
    } else {
      this.failureCount = Math.max(0, this.failureCount - 1);
    }
  }

  onFailure(responseTime) {
    this.metrics.failedRequests++;
    this.updateAverageResponseTime(responseTime);
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
    }
  }

  updateAverageResponseTime(responseTime) {
    const total = this.metrics.successfulRequests + this.metrics.failedRequests;
    this.metrics.averageResponseTime = 
      (this.metrics.averageResponseTime * (total - 1) + responseTime) / total;
  }

  getHealthStatus() {
    return {
      service: this.serviceName,
      state: this.state,
      failureCount: this.failureCount,
      successCount: this.successCount,
      metrics: this.metrics,
      lastFailureTime: this.lastFailureTime
    };
  }
}
```

---

## 🔄 **2. Retry Mechanisms**

### **Current Implementation:**
```javascript
// Basic retry with exponential backoff
const retryWithBackoff = async (operation, maxRetries = 3, baseDelay = 1000) => {
  // ... existing implementation
};
```

### **Enhanced Retry Strategy:**
```javascript
// Advanced retry with jitter and adaptive delays
class RetryStrategy {
  constructor(options = {}) {
    this.maxRetries = options.maxRetries || 3;
    this.baseDelay = options.baseDelay || 1000;
    this.maxDelay = options.maxDelay || 30000;
    this.jitter = options.jitter || true;
    this.backoffMultiplier = options.backoffMultiplier || 2;
  }

  async execute(operation, context = {}) {
    let lastError;
    let attempt = 0;

    while (attempt <= this.maxRetries) {
      try {
        const result = await operation();
        
        // Log successful retry if it wasn't the first attempt
        if (attempt > 0) {
          console.log(`[RETRY] Operation succeeded on attempt ${attempt + 1}`);
        }
        
        return result;
      } catch (error) {
        lastError = error;
        attempt++;

        // Don't retry if it's the last attempt or error is not retryable
        if (attempt > this.maxRetries || !this.isRetryableError(error)) {
          break;
        }

        const delay = this.calculateDelay(attempt);
        console.log(`[RETRY] Attempt ${attempt}/${this.maxRetries + 1} failed, retrying in ${delay}ms`, {
          error: error.message,
          context
        });

        await this.sleep(delay);
      }
    }

    throw lastError;
  }

  calculateDelay(attempt) {
    let delay = this.baseDelay * Math.pow(this.backoffMultiplier, attempt - 1);
    
    // Cap the delay
    delay = Math.min(delay, this.maxDelay);
    
    // Add jitter to prevent thundering herd
    if (this.jitter) {
      delay = delay * (0.5 + Math.random() * 0.5);
    }
    
    return Math.floor(delay);
  }

  isRetryableError(error) {
    const message = error.message?.toLowerCase() || '';
    const status = error.status || error.statusCode;

    // Retry on network errors, timeouts, and rate limits
    return (
      message.includes('network') ||
      message.includes('timeout') ||
      message.includes('rate limit') ||
      message.includes('temporary') ||
      status === 429 || // Too Many Requests
      status === 503 || // Service Unavailable
      status === 502 || // Bad Gateway
      status === 504    // Gateway Timeout
    );
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

---

## ⏱️ **3. Timeout Management**

### **Enhanced Timeout Wrapper:**
```javascript
// Advanced timeout with cleanup and monitoring
class TimeoutManager {
  constructor(defaultTimeout = 30000) {
    this.defaultTimeout = defaultTimeout;
    this.activeTimeouts = new Map();
  }

  async withTimeout(operation, timeoutMs = null, context = {}) {
    const timeout = timeoutMs || this.defaultTimeout;
    const timeoutId = `timeout_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    
    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        this.activeTimeouts.delete(timeoutId);
        reject(new Error(`Operation timeout after ${timeout}ms`));
      }, timeout);

      this.activeTimeouts.set(timeoutId, {
        timer,
        startTime: Date.now(),
        timeout,
        context
      });

      operation()
        .then(result => {
          clearTimeout(timer);
          this.activeTimeouts.delete(timeoutId);
          resolve(result);
        })
        .catch(error => {
          clearTimeout(timer);
          this.activeTimeouts.delete(timeoutId);
          reject(error);
        });
    });
  }

  getActiveTimeouts() {
    return Array.from(this.activeTimeouts.entries()).map(([id, data]) => ({
      id,
      elapsed: Date.now() - data.startTime,
      timeout: data.timeout,
      context: data.context
    }));
  }

  clearAllTimeouts() {
    for (const [id, data] of this.activeTimeouts) {
      clearTimeout(data.timer);
    }
    this.activeTimeouts.clear();
  }
}
```

---

## 🔒 **4. Concurrency Control**

### **Enhanced Concurrency Limiter:**
```javascript
// Advanced concurrency control with priority queuing
class ConcurrencyManager {
  constructor(options = {}) {
    this.maxConcurrent = options.maxConcurrent || 5;
    this.running = 0;
    this.queue = [];
    this.priorityQueue = [];
    this.metrics = {
      totalProcessed: 0,
      averageWaitTime: 0,
      averageExecutionTime: 0
    };
  }

  async execute(operation, priority = 0, context = {}) {
    return new Promise((resolve, reject) => {
      const task = {
        operation,
        priority,
        context,
        resolve,
        reject,
        queuedAt: Date.now()
      };

      if (priority > 0) {
        this.priorityQueue.push(task);
        this.priorityQueue.sort((a, b) => b.priority - a.priority);
      } else {
        this.queue.push(task);
      }

      this.process();
    });
  }

  async process() {
    if (this.running >= this.maxConcurrent) return;

    const task = this.priorityQueue.shift() || this.queue.shift();
    if (!task) return;

    this.running++;
    const startTime = Date.now();
    const waitTime = startTime - task.queuedAt;

    try {
      const result = await task.operation();
      const executionTime = Date.now() - startTime;
      
      this.updateMetrics(waitTime, executionTime);
      task.resolve(result);
    } catch (error) {
      task.reject(error);
    } finally {
      this.running--;
      this.process();
    }
  }

  updateMetrics(waitTime, executionTime) {
    this.metrics.totalProcessed++;
    this.metrics.averageWaitTime = 
      (this.metrics.averageWaitTime * (this.metrics.totalProcessed - 1) + waitTime) / 
      this.metrics.totalProcessed;
    this.metrics.averageExecutionTime = 
      (this.metrics.averageExecutionTime * (this.metrics.totalProcessed - 1) + executionTime) / 
      this.metrics.totalProcessed;
  }

  getStatus() {
    return {
      running: this.running,
      queued: this.queue.length,
      priorityQueued: this.priorityQueue.length,
      metrics: this.metrics
    };
  }
}
```

---

## 📊 **5. Health Monitoring & Metrics**

### **Function Health Monitor:**
```javascript
// Comprehensive health monitoring for Deno functions
class FunctionHealthMonitor {
  constructor() {
    this.metrics = new Map();
    this.alerts = [];
    this.thresholds = {
      errorRate: 0.05, // 5% error rate threshold
      responseTime: 5000, // 5 second response time threshold
      availability: 0.99 // 99% availability threshold
    };
  }

  recordFunctionCall(functionName, startTime, endTime, success, error = null) {
    const duration = endTime - startTime;
    
    if (!this.metrics.has(functionName)) {
      this.metrics.set(functionName, {
        totalCalls: 0,
        successfulCalls: 0,
        failedCalls: 0,
        totalDuration: 0,
        averageDuration: 0,
        errorRate: 0,
        lastCall: null,
        errors: []
      });
    }

    const metric = this.metrics.get(functionName);
    metric.totalCalls++;
    metric.totalDuration += duration;
    metric.averageDuration = metric.totalDuration / metric.totalCalls;
    metric.lastCall = endTime;

    if (success) {
      metric.successfulCalls++;
    } else {
      metric.failedCalls++;
      metric.errors.push({
        timestamp: endTime,
        error: error?.message || 'Unknown error',
        duration
      });
      
      // Keep only last 100 errors
      if (metric.errors.length > 100) {
        metric.errors = metric.errors.slice(-100);
      }
    }

    metric.errorRate = metric.failedCalls / metric.totalCalls;
    
    // Check thresholds and generate alerts
    this.checkThresholds(functionName, metric);
  }

  checkThresholds(functionName, metric) {
    const now = Date.now();
    
    // Error rate threshold
    if (metric.errorRate > this.thresholds.errorRate) {
      this.addAlert({
        type: 'error_rate',
        function: functionName,
        value: metric.errorRate,
        threshold: this.thresholds.errorRate,
        timestamp: now,
        severity: 'high'
      });
    }

    // Response time threshold
    if (metric.averageDuration > this.thresholds.responseTime) {
      this.addAlert({
        type: 'response_time',
        function: functionName,
        value: metric.averageDuration,
        threshold: this.thresholds.responseTime,
        timestamp: now,
        severity: 'medium'
      });
    }

    // Availability threshold (last 100 calls)
    const recentCalls = Math.min(100, metric.totalCalls);
    const recentSuccessRate = metric.successfulCalls / recentCalls;
    if (recentSuccessRate < this.thresholds.availability) {
      this.addAlert({
        type: 'availability',
        function: functionName,
        value: recentSuccessRate,
        threshold: this.thresholds.availability,
        timestamp: now,
        severity: 'critical'
      });
    }
  }

  addAlert(alert) {
    this.alerts.push(alert);
    
    // Keep only last 1000 alerts
    if (this.alerts.length > 1000) {
      this.alerts = this.alerts.slice(-1000);
    }

    // Log critical alerts
    if (alert.severity === 'critical') {
      console.error(`[HEALTH] Critical alert: ${alert.type} for ${alert.function}`, alert);
    }
  }

  getHealthReport() {
    const report = {
      timestamp: new Date().toISOString(),
      functions: {},
      overallHealth: 'healthy',
      activeAlerts: this.alerts.filter(alert => 
        Date.now() - alert.timestamp < 300000 // Last 5 minutes
      )
    };

    let totalCalls = 0;
    let totalErrors = 0;
    let totalDuration = 0;

    for (const [functionName, metric] of this.metrics) {
      report.functions[functionName] = {
        ...metric,
        availability: metric.successfulCalls / metric.totalCalls,
        lastCall: metric.lastCall ? new Date(metric.lastCall).toISOString() : null
      };

      totalCalls += metric.totalCalls;
      totalErrors += metric.failedCalls;
      totalDuration += metric.totalDuration;
    }

    // Calculate overall health
    const overallErrorRate = totalCalls > 0 ? totalErrors / totalCalls : 0;
    const overallAvailability = totalCalls > 0 ? (totalCalls - totalErrors) / totalCalls : 1;

    if (overallErrorRate > 0.1 || overallAvailability < 0.95) {
      report.overallHealth = 'critical';
    } else if (overallErrorRate > 0.05 || overallAvailability < 0.99) {
      report.overallHealth = 'warning';
    }

    return report;
  }
}
```

---

## 🛠️ **6. Resilient Function Wrapper**

### **Complete Function Wrapper:**
```javascript
// Comprehensive wrapper for resilient Deno functions
class ResilientFunctionWrapper {
  constructor(functionName, options = {}) {
    this.functionName = functionName;
    this.circuitBreaker = new EnhancedCircuitBreaker(functionName, options.circuitBreaker);
    this.retryStrategy = new RetryStrategy(options.retry);
    this.timeoutManager = new TimeoutManager(options.timeout);
    this.concurrencyManager = new ConcurrencyManager(options.concurrency);
    this.healthMonitor = new FunctionHealthMonitor();
    this.options = options;
  }

  async execute(operation, context = {}) {
    const startTime = Date.now();
    let result;
    let error;

    try {
      result = await this.concurrencyManager.execute(
        () => this.circuitBreaker.execute(
          () => this.timeoutManager.withTimeout(
            () => this.retryStrategy.execute(operation, context),
            this.options.timeout,
            context
          )
        ),
        context.priority || 0,
        context
      );

      this.healthMonitor.recordFunctionCall(
        this.functionName,
        startTime,
        Date.now(),
        true
      );

      return result;
    } catch (err) {
      error = err;
      this.healthMonitor.recordFunctionCall(
        this.functionName,
        startTime,
        Date.now(),
        false,
        err
      );
      throw err;
    }
  }

  getHealthStatus() {
    return {
      function: this.functionName,
      circuitBreaker: this.circuitBreaker.getHealthStatus(),
      concurrency: this.concurrencyManager.getStatus(),
      healthReport: this.healthMonitor.getHealthReport()
    };
  }
}

// Usage example for AI functions
const groqWrapper = new ResilientFunctionWrapper('invokeGroq', {
  circuitBreaker: { failureThreshold: 5, timeout: 60000 },
  retry: { maxRetries: 3, baseDelay: 1000 },
  timeout: 30000,
  concurrency: { maxConcurrent: 3 }
});

export const invokeGroqResilient = async (params) => {
  return groqWrapper.execute(async () => {
    return await base44.functions.invokeGroq(params);
  }, { userId: params.userId, promptLength: params.prompt?.length });
};
```

---

## 🚨 **7. Alerting & Monitoring**

### **Alert System:**
```javascript
// Alert system for function health monitoring
class FunctionAlertSystem {
  constructor() {
    this.alerts = [];
    this.notificationChannels = [];
  }

  addNotificationChannel(channel) {
    this.notificationChannels.push(channel);
  }

  async sendAlert(alert) {
    // Log alert
    console.error(`[ALERT] ${alert.severity.toUpperCase()}: ${alert.type}`, alert);
    
    // Send to notification channels
    for (const channel of this.notificationChannels) {
      try {
        await channel.send(alert);
      } catch (error) {
        console.error('Failed to send alert via channel:', error);
      }
    }
  }

  // Slack notification channel example
  createSlackChannel(webhookUrl) {
    return {
      send: async (alert) => {
        const message = {
          text: `🚨 Function Alert: ${alert.function}`,
          attachments: [{
            color: alert.severity === 'critical' ? 'danger' : 'warning',
            fields: [
              { title: 'Type', value: alert.type, short: true },
              { title: 'Value', value: alert.value.toString(), short: true },
              { title: 'Threshold', value: alert.threshold.toString(), short: true },
              { title: 'Severity', value: alert.severity, short: true }
            ],
            timestamp: Math.floor(alert.timestamp / 1000)
          }]
        };

        await fetch(webhookUrl, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(message)
        });
      }
    };
  }
}
```

---

## 📈 **8. Performance Optimization**

### **Function Performance Optimizer:**
```javascript
// Performance optimization for Deno functions
class FunctionPerformanceOptimizer {
  constructor() {
    this.cache = new Map();
    this.batchQueue = new Map();
    this.batchTimers = new Map();
  }

  // Request batching for similar operations
  batchRequests(key, operation, delay = 100) {
    return new Promise((resolve, reject) => {
      if (!this.batchQueue.has(key)) {
        this.batchQueue.set(key, []);
      }

      this.batchQueue.get(key).push({ resolve, reject, operation });

      if (!this.batchTimers.has(key)) {
        this.batchTimers.set(key, setTimeout(() => {
          this.processBatch(key);
        }, delay));
      }
    });
  }

  async processBatch(key) {
    const batch = this.batchQueue.get(key) || [];
    this.batchQueue.delete(key);
    this.batchTimers.delete(key);

    if (batch.length === 0) return;

    try {
      // Process batch operations
      const results = await Promise.all(
        batch.map(item => item.operation())
      );

      // Resolve all promises
      batch.forEach((item, index) => {
        item.resolve(results[index]);
      });
    } catch (error) {
      // Reject all promises
      batch.forEach(item => {
        item.reject(error);
      });
    }
  }

  // Connection pooling for external APIs
  createConnectionPool(maxConnections = 10) {
    const pool = {
      connections: [],
      available: [],
      inUse: new Set(),
      maxConnections
    };

    return {
      async getConnection() {
        if (pool.available.length > 0) {
          const connection = pool.available.pop();
          pool.inUse.add(connection);
          return connection;
        }

        if (pool.connections.length < pool.maxConnections) {
          const connection = await this.createConnection();
          pool.connections.push(connection);
          pool.inUse.add(connection);
          return connection;
        }

        // Wait for available connection
        return new Promise((resolve) => {
          const checkAvailable = () => {
            if (pool.available.length > 0) {
              const connection = pool.available.pop();
              pool.inUse.add(connection);
              resolve(connection);
            } else {
              setTimeout(checkAvailable, 100);
            }
          };
          checkAvailable();
        });
      },

      releaseConnection(connection) {
        pool.inUse.delete(connection);
        pool.available.push(connection);
      }
    };
  }
}
```

---

## 🎯 **Implementation Recommendations**

### **1. Immediate Actions:**
1. **Implement enhanced circuit breakers** for all external API calls
2. **Add comprehensive health monitoring** to all functions
3. **Set up alerting** for critical function failures
4. **Implement request batching** for similar operations

### **2. Medium-term Improvements:**
1. **Add performance metrics** collection and analysis
2. **Implement connection pooling** for external APIs
3. **Add automated scaling** based on load
4. **Create function health dashboard**

### **3. Long-term Enhancements:**
1. **Implement chaos engineering** for resilience testing
2. **Add predictive scaling** based on usage patterns
3. **Create automated recovery** mechanisms
4. **Implement distributed tracing** for complex operations

---

## ✅ **Benefits of These Patterns**

- **🛡️ Fault Tolerance**: Functions continue working even when dependencies fail
- **📊 Observability**: Complete visibility into function health and performance
- **⚡ Performance**: Optimized resource usage and response times
- **🔧 Maintainability**: Easy to debug and optimize function behavior
- **📈 Scalability**: Functions can handle increased load gracefully

These patterns will ensure your Deno functions are resilient, observable, and performant in production environments! 🚀
