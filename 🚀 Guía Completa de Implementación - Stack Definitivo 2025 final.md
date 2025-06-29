# 🚀 GUÍA COMPLETA DE IMPLEMENTACIÓN - STACK DEFINITIVO 2025

## 📋 Tabla de Contenidos
1. [Arquitectura de Microservicios](#1-arquitectura-de-microservicios)
2. [Frontend PWA Impresionante](#2-frontend-pwa-impresionante)
3. [IA y Embeddings](#3-ia-y-embeddings)
4. [Monitoreo Enterprise](#4-monitoreo-enterprise)
5. [Calendario de Reservas "Chingón"](#5-calendario-de-reservas-chingón)
6. [Seguridad y Mejores Prácticas](#6-seguridad-y-mejores-prácticas)

---

## 1. ARQUITECTURA DE MICROSERVICIOS

### 1.1 n8n como Gateway Central con Rate Limiting

#### Configuración Completa de n8n Gateway:

```javascript
// n8n-workflows/master-gateway.json
{
  "name": "Master API Gateway with Rate Limiting",
  "nodes": [
    {
      "parameters": {
        "path": "gateway/{{service}}/{{action}}",
        "responseMode": "onReceived",
        "responseData": "allEntries",
        "options": {
          "responseHeaders": {
            "values": {
              "X-Gateway-Version": "1.0",
              "X-Request-ID": "={{$uuid}}"
            }
          }
        }
      },
      "name": "Gateway Entry",
      "type": "n8n-nodes-base.webhook",
      "position": [250, 300]
    },
    {
      "parameters": {
        "functionCode": `
          // Rate Limiting Implementation
          const Redis = require('ioredis');
          const redis = new Redis({
            host: 'redis',
            password: process.env.REDIS_PASSWORD,
            retryStrategy: (times) => Math.min(times * 50, 2000)
          });

          // Extract client identifier
          const clientId = $input.item.headers['x-api-key'] || 
                          $input.item.headers['authorization'] || 
                          $input.item.headers['x-forwarded-for'] || 
                          'anonymous';
          
          // Rate limit configuration per service
          const rateLimits = {
            'llama': { requests: 10, window: 60 }, // 10 req/min
            'embeddings': { requests: 100, window: 60 }, // 100 req/min
            'calendar': { requests: 300, window: 60 }, // 300 req/min
            'ecommerce': { requests: 500, window: 60 }, // 500 req/min
            'default': { requests: 100, window: 60 }
          };
          
          const service = $input.item.params.service;
          const limit = rateLimits[service] || rateLimits.default;
          
          // Sliding window rate limiting
          const now = Date.now();
          const windowStart = now - (limit.window * 1000);
          const key = \`rate_limit:\${service}:\${clientId}\`;
          
          // Remove old entries
          await redis.zremrangebyscore(key, '-inf', windowStart);
          
          // Count current requests in window
          const currentCount = await redis.zcard(key);
          
          if (currentCount >= limit.requests) {
            throw new Error(\`Rate limit exceeded: \${currentCount}/\${limit.requests} requests in \${limit.window}s\`);
          }
          
          // Add current request
          await redis.zadd(key, now, \`\${now}:\${$uuid}\`);
          await redis.expire(key, limit.window + 1);
          
          // Add rate limit headers
          $input.item.rateLimitInfo = {
            limit: limit.requests,
            remaining: limit.requests - currentCount - 1,
            reset: new Date(now + (limit.window * 1000)).toISOString()
          };
          
          return $input.item;
        `
      },
      "name": "Rate Limiter",
      "type": "n8n-nodes-base.function",
      "position": [450, 300]
    },
    {
      "parameters": {
        "functionCode": `
          // Request validation and sanitization
          const validator = require('validator');
          const DOMPurify = require('isomorphic-dompurify');
          
          // Validate request structure
          const requiredFields = {
            'llama': ['prompt', 'model'],
            'embeddings': ['text'],
            'calendar': ['action', 'data'],
            'ecommerce': ['action', 'items']
          };
          
          const service = $input.item.params.service;
          const required = requiredFields[service] || [];
          const body = $input.item.body || {};
          
          // Check required fields
          for (const field of required) {
            if (!body[field]) {
              throw new Error(\`Missing required field: \${field}\`);
            }
          }
          
          // Sanitize string inputs
          const sanitizeInput = (input) => {
            if (typeof input === 'string') {
              // Remove any HTML/script tags
              input = DOMPurify.sanitize(input, { ALLOWED_TAGS: [] });
              // Escape SQL characters
              input = input.replace(/['";\\]/g, '\\\\$&');
              // Limit length
              input = input.substring(0, 10000);
            }
            return input;
          };
          
          // Recursively sanitize all inputs
          const sanitizeObject = (obj) => {
            if (Array.isArray(obj)) {
              return obj.map(sanitizeObject);
            } else if (obj && typeof obj === 'object') {
              const sanitized = {};
              for (const [key, value] of Object.entries(obj)) {
                sanitized[key] = sanitizeObject(value);
              }
              return sanitized;
            }
            return sanitizeInput(obj);
          };
          
          $input.item.body = sanitizeObject(body);
          
          return $input.item;
        `
      },
      "name": "Request Validator",
      "type": "n8n-nodes-base.function",
      "position": [650, 300]
    },
    {
      "parameters": {
        "dataPropertyName": "params.service",
        "rules": {
          "rules": [
            {
              "value": "llama",
              "output": 0
            },
            {
              "value": "embeddings",
              "output": 1
            },
            {
              "value": "calendar",
              "output": 2
            },
            {
              "value": "ecommerce",
              "output": 3
            },
            {
              "value": "baserow",
              "output": 4
            }
          ]
        },
        "fallbackOutput": 5
      },
      "name": "Service Router",
      "type": "n8n-nodes-base.switch",
      "position": [850, 300]
    },
    {
      "parameters": {
        "url": "http://llama:11434/api/generate",
        "method": "POST",
        "jsonParameters": true,
        "options": {
          "timeout": 30000,
          "retry": {
            "maxTries": 3,
            "onFailedAttempt": "={{$attempt < 3 ? 'retry' : 'error'}}"
          }
        },
        "bodyParametersJson": "={{JSON.stringify($input.item.body)}}",
        "headerParametersJson": {
          "X-Request-ID": "={{$input.item.headers['X-Request-ID']}}",
          "X-Gateway-Forwarded": "true"
        }
      },
      "name": "Llama Service",
      "type": "n8n-nodes-base.httpRequest",
      "position": [1050, 100]
    },
    {
      "parameters": {
        "functionCode": `
          // Circuit Breaker Implementation
          const CircuitBreaker = require('opossum');
          
          // Circuit breaker options
          const options = {
            timeout: 3000, // 3 seconds
            errorThresholdPercentage: 50,
            resetTimeout: 30000 // 30 seconds
          };
          
          // Create circuit breaker for each service
          const breakers = {
            llama: new CircuitBreaker(callLlamaService, options),
            embeddings: new CircuitBreaker(callEmbeddingsService, options),
            calendar: new CircuitBreaker(callCalendarService, options),
            ecommerce: new CircuitBreaker(callEcommerceService, options)
          };
          
          // Monitor circuit breaker events
          Object.entries(breakers).forEach(([service, breaker]) => {
            breaker.on('open', () => {
              console.log(\`Circuit breaker is OPEN for \${service}\`);
              // Send alert to monitoring
            });
            
            breaker.on('halfOpen', () => {
              console.log(\`Circuit breaker is HALF-OPEN for \${service}\`);
            });
            
            breaker.fallback(() => {
              return {
                error: true,
                message: \`Service \${service} is temporarily unavailable\`,
                fallback: true
              };
            });
          });
          
          // Execute with circuit breaker
          const service = $input.item.params.service;
          const breaker = breakers[service];
          
          if (breaker) {
            try {
              const result = await breaker.fire($input.item);
              return { json: result };
            } catch (error) {
              console.error(\`Circuit breaker error for \${service}:\`, error);
              throw error;
            }
          }
          
          return $input.item;
        `
      },
      "name": "Circuit Breaker",
      "type": "n8n-nodes-base.function",
      "position": [1250, 300]
    },
    {
      "parameters": {
        "functionCode": `
          // Response formatting and rate limit headers
          const response = $input.item;
          
          // Add rate limit headers to response
          if (response.rateLimitInfo) {
            response.headers = {
              ...response.headers,
              'X-RateLimit-Limit': response.rateLimitInfo.limit,
              'X-RateLimit-Remaining': response.rateLimitInfo.remaining,
              'X-RateLimit-Reset': response.rateLimitInfo.reset
            };
          }
          
          // Log metrics for Prometheus
          const prometheus = require('prom-client');
          
          // Request counter
          const requestCounter = new prometheus.Counter({
            name: 'gateway_requests_total',
            help: 'Total number of requests',
            labelNames: ['service', 'method', 'status']
          });
          
          // Request duration histogram
          const requestDuration = new prometheus.Histogram({
            name: 'gateway_request_duration_seconds',
            help: 'Request duration in seconds',
            labelNames: ['service', 'method'],
            buckets: [0.1, 0.5, 1, 2, 5]
          });
          
          // Update metrics
          const labels = {
            service: response.params.service,
            method: response.method || 'unknown',
            status: response.error ? 'error' : 'success'
          };
          
          requestCounter.inc(labels);
          requestDuration.observe(labels, response.duration || 0);
          
          return response;
        `
      },
      "name": "Response Handler",
      "type": "n8n-nodes-base.function",
      "position": [1450, 300]
    }
  ]
}
```

### 1.2 Circuit Breakers para Resiliencia

#### Implementación Completa de Circuit Breakers:

```typescript
// lib/circuit-breaker/resilient-service.ts
import CircuitBreaker from 'opossum';
import { EventEmitter } from 'events';

interface ServiceConfig {
  name: string;
  timeout?: number;
  errorThresholdPercentage?: number;
  resetTimeout?: number;
  rollingCountTimeout?: number;
  rollingCountBuckets?: number;
  fallbackFunction?: Function;
}

export class ResilientService extends EventEmitter {
  private breakers: Map<string, CircuitBreaker> = new Map();
  private metrics: Map<string, any> = new Map();

  constructor(private config: ServiceConfig[]) {
    super();
    this.initializeBreakers();
  }

  private initializeBreakers() {
    this.config.forEach(service => {
      const options = {
        timeout: service.timeout || 3000,
        errorThresholdPercentage: service.errorThresholdPercentage || 50,
        resetTimeout: service.resetTimeout || 30000,
        rollingCountTimeout: service.rollingCountTimeout || 10000,
        rollingCountBuckets: service.rollingCountBuckets || 10,
        name: service.name
      };

      const breaker = new CircuitBreaker(
        this.createServiceFunction(service.name),
        options
      );

      // Attach event listeners
      this.attachEventListeners(breaker, service.name);

      // Set fallback if provided
      if (service.fallbackFunction) {
        breaker.fallback(service.fallbackFunction);
      } else {
        breaker.fallback(this.defaultFallback(service.name));
      }

      this.breakers.set(service.name, breaker);
    });
  }

  private createServiceFunction(serviceName: string) {
    return async (request: any) => {
      // This will be overridden by actual service implementations
      throw new Error(`Service ${serviceName} not implemented`);
    };
  }

  private attachEventListeners(breaker: CircuitBreaker, serviceName: string) {
    breaker.on('open', () => {
      console.error(`Circuit breaker OPENED for ${serviceName}`);
      this.emit('circuit:open', { service: serviceName, timestamp: new Date() });
      
      // Send alert to monitoring
      this.sendAlert({
        level: 'critical',
        service: serviceName,
        message: `Circuit breaker opened for ${serviceName}`,
        action: 'investigate_immediately'
      });
    });

    breaker.on('halfOpen', () => {
      console.warn(`Circuit breaker HALF-OPEN for ${serviceName}`);
      this.emit('circuit:halfOpen', { service: serviceName });
    });

    breaker.on('close', () => {
      console.info(`Circuit breaker CLOSED for ${serviceName}`);
      this.emit('circuit:close', { service: serviceName });
    });

    breaker.on('success', (result) => {
      this.updateMetrics(serviceName, 'success');
    });

    breaker.on('failure', (error) => {
      this.updateMetrics(serviceName, 'failure');
      console.error(`Service ${serviceName} failed:`, error);
    });

    breaker.on('timeout', () => {
      this.updateMetrics(serviceName, 'timeout');
      console.error(`Service ${serviceName} timed out`);
    });

    breaker.on('reject', () => {
      this.updateMetrics(serviceName, 'rejected');
    });
  }

  private defaultFallback(serviceName: string) {
    return async (error: Error) => {
      console.warn(`Fallback activated for ${serviceName}:`, error.message);
      
      // Return cached data if available
      const cachedData = await this.getCachedData(serviceName);
      if (cachedData) {
        return {
          data: cachedData,
          fallback: true,
          cached: true,
          service: serviceName
        };
      }

      // Return degraded response
      return {
        error: true,
        message: `Service ${serviceName} is temporarily unavailable`,
        fallback: true,
        service: serviceName,
        retryAfter: 30
      };
    };
  }

  private updateMetrics(serviceName: string, event: string) {
    if (!this.metrics.has(serviceName)) {
      this.metrics.set(serviceName, {
        success: 0,
        failure: 0,
        timeout: 0,
        rejected: 0
      });
    }

    const serviceMetrics = this.metrics.get(serviceName);
    serviceMetrics[event]++;
    
    // Emit metrics for monitoring
    this.emit('metrics:update', {
      service: serviceName,
      metrics: serviceMetrics
    });
  }

  async callService(serviceName: string, request: any): Promise<any> {
    const breaker = this.breakers.get(serviceName);
    if (!breaker) {
      throw new Error(`Service ${serviceName} not configured`);
    }

    try {
      const startTime = Date.now();
      const result = await breaker.fire(request);
      const duration = Date.now() - startTime;

      // Log successful call
      console.info(`Service ${serviceName} responded in ${duration}ms`);
      
      return result;
    } catch (error) {
      console.error(`Service ${serviceName} call failed:`, error);
      throw error;
    }
  }

  private async getCachedData(serviceName: string): Promise<any> {
    // Implement Redis cache lookup
    try {
      const redis = this.getRedisClient();
      const cached = await redis.get(`cache:${serviceName}:fallback`);
      return cached ? JSON.parse(cached) : null;
    } catch (error) {
      console.error('Cache lookup failed:', error);
      return null;
    }
  }

  private async sendAlert(alert: any) {
    // Implement alert sending logic (Slack, PagerDuty, etc.)
    console.error('ALERT:', alert);
  }

  getStats(serviceName?: string) {
    if (serviceName) {
      const breaker = this.breakers.get(serviceName);
      return breaker ? breaker.stats : null;
    }

    const allStats: any = {};
    this.breakers.forEach((breaker, name) => {
      allStats[name] = breaker.stats;
    });
    return allStats;
  }

  private getRedisClient() {
    // Return Redis client instance
    // This should be injected or configured elsewhere
    return require('redis').createClient({
      host: 'redis',
      password: process.env.REDIS_PASSWORD
    });
  }
}

// Usage Example:
const services = new ResilientService([
  {
    name: 'llama',
    timeout: 30000, // 30 seconds for AI responses
    errorThresholdPercentage: 30, // More tolerant for AI
    resetTimeout: 60000 // 1 minute
  },
  {
    name: 'embeddings',
    timeout: 5000,
    errorThresholdPercentage: 50,
    resetTimeout: 30000
  },
  {
    name: 'calendar',
    timeout: 3000,
    errorThresholdPercentage: 50,
    resetTimeout: 30000
  },
  {
    name: 'payment',
    timeout: 5000,
    errorThresholdPercentage: 20, // Very strict for payments
    resetTimeout: 60000,
    fallbackFunction: async () => ({
      error: true,
      message: 'Payment service unavailable. Please try again later.',
      code: 'PAYMENT_SERVICE_DOWN'
    })
  }
]);
```

### 1.3 Prometheus Exporters Específicos

#### Configuración de Exporters para cada Servicio:

```yaml
# docker-compose.yml - Sección de Exporters
services:
  # Node Exporter para métricas del sistema
  node-exporter:
    image: prom/node-exporter:latest
    command:
      - '--path.rootfs=/host'
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    ports:
      - "9100:9100"
    networks:
      - microservices-net

  # PostgreSQL Exporter
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    environment:
      DATA_SOURCE_NAME: "postgresql://monitoring:${POSTGRES_MONITORING_PASSWORD}@postgres:5432/postgres?sslmode=disable"
      PG_EXPORTER_EXTEND_QUERY_PATH: "/etc/postgres_exporter/queries.yaml"
    volumes:
      - ./monitoring/postgres-queries.yaml:/etc/postgres_exporter/queries.yaml:ro
    ports:
      - "9187:9187"
    networks:
      - microservices-net

  # Redis Exporter
  redis-exporter:
    image: oliver006/redis_exporter:latest
    environment:
      REDIS_ADDR: "redis:6379"
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    ports:
      - "9121:9121"
    networks:
      - microservices-net

  # n8n Metrics Exporter (Custom)
  n8n-exporter:
    build: ./exporters/n8n
    environment:
      N8N_API_URL: "http://n8n:5678"
      N8N_API_KEY: ${N8N_API_KEY}
    ports:
      - "9200:9200"
    networks:
      - microservices-net

  # Container Exporter (cAdvisor)
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    devices:
      - /dev/kmsg
    ports:
      - "8080:8080"
    networks:
      - microservices-net
```

#### Custom n8n Exporter:

```typescript
// exporters/n8n/index.ts
import express from 'express';
import { register, Gauge, Counter, Histogram } from 'prom-client';
import axios from 'axios';

const app = express();
const port = 9200;

// Metrics definitions
const workflowExecutions = new Counter({
  name: 'n8n_workflow_executions_total',
  help: 'Total number of workflow executions',
  labelNames: ['workflow_name', 'status']
});

const workflowDuration = new Histogram({
  name: 'n8n_workflow_duration_seconds',
  help: 'Workflow execution duration in seconds',
  labelNames: ['workflow_name'],
  buckets: [0.1, 0.5, 1, 5, 10, 30, 60]
});

const activeWorkflows = new Gauge({
  name: 'n8n_active_workflows',
  help: 'Number of active workflows'
});

const queuedExecutions = new Gauge({
  name: 'n8n_queued_executions',
  help: 'Number of queued executions'
});

const nodeExecutions = new Counter({
  name: 'n8n_node_executions_total',
  help: 'Total number of node executions',
  labelNames: ['node_type', 'status']
});

// Collect metrics from n8n
async function collectMetrics() {
  try {
    const headers = {
      'X-N8N-API-KEY': process.env.N8N_API_KEY
    };

    // Get workflow executions
    const executions = await axios.get(
      `${process.env.N8N_API_URL}/api/v1/executions`,
      { headers }
    );

    // Process execution data
    executions.data.data.forEach((execution: any) => {
      workflowExecutions.inc({
        workflow_name: execution.workflowData.name,
        status: execution.finished ? 'completed' : 'running'
      });

      if (execution.startedAt && execution.stoppedAt) {
        const duration = (new Date(execution.stoppedAt).getTime() - 
                         new Date(execution.startedAt).getTime()) / 1000;
        
        workflowDuration.observe(
          { workflow_name: execution.workflowData.name },
          duration
        );
      }
    });

    // Get active workflows
    const workflows = await axios.get(
      `${process.env.N8N_API_URL}/api/v1/workflows`,
      { headers }
    );

    const activeCount = workflows.data.data.filter((w: any) => w.active).length;
    activeWorkflows.set(activeCount);

    // Get queue status
    const queue = await axios.get(
      `${process.env.N8N_API_URL}/api/v1/queue`,
      { headers }
    );

    queuedExecutions.set(queue.data.waiting || 0);

  } catch (error) {
    console.error('Error collecting n8n metrics:', error);
  }
}

// Collect metrics every 15 seconds
setInterval(collectMetrics, 15000);

// Metrics endpoint
app.get('/metrics', (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(register.metrics());
});

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

app.listen(port, () => {
  console.log(`n8n exporter listening on port ${port}`);
});
```

#### PostgreSQL Custom Queries para Monitoring:

```yaml
# monitoring/postgres-queries.yaml
pg_database:
  query: |
    SELECT 
      datname as database_name,
      pg_database_size(datname) as size_bytes,
      numbackends as connections,
      xact_commit as transactions_committed,
      xact_rollback as transactions_rolled_back,
      blks_read as blocks_read,
      blks_hit as blocks_hit,
      tup_returned as tuples_returned,
      tup_fetched as tuples_fetched,
      tup_inserted as tuples_inserted,
      tup_updated as tuples_updated,
      tup_deleted as tuples_deleted
    FROM pg_stat_database
    WHERE datname NOT IN ('template0', 'template1', 'postgres')
  metrics:
    - database_name:
        usage: "LABEL"
        description: "Name of the database"
    - size_bytes:
        usage: "GAUGE"
        description: "Database size in bytes"
    - connections:
        usage: "GAUGE"
        description: "Number of active connections"

pg_vector_indexes:
  query: |
    SELECT 
      schemaname,
      tablename,
      indexname,
      pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
      idx_scan as index_scans,
      idx_tup_read as tuples_read,
      idx_tup_fetch as tuples_fetched
    FROM pg_stat_user_indexes
    WHERE indexname LIKE '%vector%'
  metrics:
    - schemaname:
        usage: "LABEL"
    - tablename:
        usage: "LABEL"
    - indexname:
        usage: "LABEL"
    - index_size:
        usage: "GAUGE"
        description: "Size of vector index"

pg_locks:
  query: |
    SELECT 
      mode,
      count(*) as count
    FROM pg_locks
    GROUP BY mode
  metrics:
    - mode:
        usage: "LABEL"
        description: "Lock mode"
    - count:
        usage: "GAUGE"
        description: "Number of locks"
```

---

## 2. FRONTEND PWA IMPRESIONANTE

### 2.1 Web App Manifest Completo

```json
// public/manifest.json
{
  "name": "Tu Experiencia Premium - Reservas y Más",
  "short_name": "ExperienciasPremium",
  "description": "La mejor plataforma para reservar experiencias únicas y exclusivas",
  "start_url": "/",
  "display": "standalone",
  "orientation": "portrait",
  "theme_color": "#1a1a1a",
  "background_color": "#000000",
  "prefer_related_applications": false,
  "scope": "/",
  "lang": "es-MX",
  "dir": "ltr",
  "categories": ["travel", "lifestyle", "business"],
  
  "icons": [
    {
      "src": "/icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-96x96.png",
      "sizes": "96x96",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-128x128.png",
      "sizes": "128x128",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-144x144.png",
      "sizes": "144x144",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-152x152.png",
      "sizes": "152x152",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-384x384.png",
      "sizes": "384x384",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-maskable-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "maskable"
    },
    {
      "src": "/icons/icon-maskable-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ],
  
  "screenshots": [
    {
      "src": "/screenshots/home-mobile.png",
      "sizes": "412x915",
      "type": "image/png",
      "platform": "narrow",
      "label": "Pantalla de inicio"
    },
    {
      "src": "/screenshots/booking-mobile.png",
      "sizes": "412x915",
      "type": "image/png",
      "platform": "narrow",
      "label": "Sistema de reservas"
    },
    {
      "src": "/screenshots/home-desktop.png",
      "sizes": "1920x1080",
      "type": "image/png",
      "platform": "wide",
      "label": "Vista de escritorio"
    }
  ],
  
  "shortcuts": [
    {
      "name": "Nueva Reserva",
      "short_name": "Reservar",
      "description": "Crear una nueva reserva rápidamente",
      "url": "/booking/new?utm_source=pwa_shortcut",
      "icons": [
        {
          "src": "/icons/shortcut-booking.png",
          "sizes": "96x96"
        }
      ]
    },
    {
      "name": "Mis Reservas",
      "short_name": "Mis Reservas",
      "description": "Ver todas tus reservas activas",
      "url": "/my-bookings?utm_source=pwa_shortcut",
      "icons": [
        {
          "src": "/icons/shortcut-list.png",
          "sizes": "96x96"
        }
      ]
    },
    {
      "name": "Ofertas Especiales",
      "short_name": "Ofertas",
      "description": "Ver las ofertas del día",
      "url": "/deals?utm_source=pwa_shortcut",
      "icons": [
        {
          "src": "/icons/shortcut-deals.png",
          "sizes": "96x96"
        }
      ]
    }
  ],
  
  "share_target": {
    "action": "/share",
    "method": "GET",
    "params": {
      "title": "title",
      "text": "text",
      "url": "url"
    }
  },
  
  "protocol_handlers": [
    {
      "protocol": "web+booking",
      "url": "/booking?id=%s"
    }
  ],
  
  "related_applications": [],
  
  "features": [
    "Cross Platform",
    "Fast",
    "Offline Capable",
    "Installable"
  ],
  
  "display_override": [
    "window-controls-overlay",
    "standalone",
    "minimal-ui",
    "browser"
  ],
  
  "launch_handler": {
    "client_mode": "navigate-existing"
  }
}
```

### 2.2 Service Workers Avanzados con Estrategias de Caché

```typescript
// public/sw.ts
/// <reference lib="webworker" />
import { precacheAndRoute, cleanupOutdatedCaches } from 'workbox-precaching';
import { registerRoute, NavigationRoute } from 'workbox-routing';
import { 
  NetworkFirst, 
  StaleWhileRevalidate, 
  CacheFirst,
  NetworkOnly 
} from 'workbox-strategies';
import { ExpirationPlugin } from 'workbox-expiration';
import { CacheableResponsePlugin } from 'workbox-cacheable-response';
import { BackgroundSyncPlugin } from 'workbox-background-sync';
import { Queue } from 'workbox-background-sync';

declare const self: ServiceWorkerGlobalScope;

// Precache all static assets
precacheAndRoute(self.__WB_MANIFEST || []);
cleanupOutdatedCaches();

// Cache names
const CACHE_NAMES = {
  static: 'static-cache-v1',
  dynamic: 'dynamic-cache-v1',
  images: 'images-cache-v1',
  api: 'api-cache-v1',
  offline: 'offline-cache-v1'
};

// 1. Static Assets Strategy (CSS, JS) - Cache First
registerRoute(
  ({ request }) => 
    request.destination === 'script' || 
    request.destination === 'style',
  new CacheFirst({
    cacheName: CACHE_NAMES.static,
    plugins: [
      new CacheableResponsePlugin({
        statuses: [0, 200]
      }),
      new ExpirationPlugin({
        maxEntries: 100,
        maxAgeSeconds: 30 * 24 * 60 * 60, // 30 days
        purgeOnQuotaError: true
      })
    ]
  })
);

// 2. Image Caching Strategy - Cache First with Network Fallback
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: CACHE_NAMES.images,
    plugins: [
      new CacheableResponsePlugin({
        statuses: [0, 200]
      }),
      new ExpirationPlugin({
        maxEntries: 200,
        maxAgeSeconds: 60 * 24 * 60 * 60, // 60 days
        purgeOnQuotaError: true
      })
    ]
  })
);

// 3. API Routes - Network First with Intelligent Caching
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({
    cacheName: CACHE_NAMES.api,
    networkTimeoutSeconds: 3,
    plugins: [
      new CacheableResponsePlugin({
        statuses: [0, 200]
      }),
      new ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 5 * 60, // 5 minutes
        purgeOnQuotaError: true
      })
    ]
  })
);

// 4. Calendar Data - Stale While Revalidate
registerRoute(
  ({ url }) => url.pathname.includes('/calendar/'),
  new StaleWhileRevalidate({
    cacheName: 'calendar-cache',
    plugins: [
      new CacheableResponsePlugin({
        statuses: [0, 200]
      })
    ]
  })
);

// 5. Booking API - Network Only with Background Sync
const bookingQueue = new Queue('booking-queue', {
  onSync: async ({ queue }) => {
    let entry;
    while ((entry = await queue.shiftRequest())) {
      try {
        await fetch(entry.request);
        console.log('Booking synced successfully');
      } catch (error) {
        console.error('Error syncing booking:', error);
        await queue.unshiftRequest(entry);
        throw error;
      }
    }
  }
});

registerRoute(
  ({ url }) => url.pathname.includes('/api/bookings'),
  new NetworkOnly({
    plugins: [
      new BackgroundSyncPlugin('booking-sync', {
        maxRetentionTime: 24 * 60 // 24 hours
      })
    ]
  }),
  'POST'
);

// 6. Offline Page Handler
const offlinePages = [
  '/offline',
  '/booking/offline',
  '/my-bookings/offline'
];

// Cache offline pages on install
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAMES.offline).then((cache) => {
      return cache.addAll(offlinePages);
    })
  );
});

// 7. Navigation Route with Offline Fallback
const navigationHandler = new NetworkFirst({
  cacheName: 'navigation-cache',
  plugins: [
    new CacheableResponsePlugin({
      statuses: [0, 200]
    })
  ]
});

const navigationRoute = new NavigationRoute(navigationHandler, {
  denylist: [/^\/api/, /^\/admin/]
});

registerRoute(navigationRoute);

// 8. Advanced Offline Fallback
self.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => {
        return caches.match('/offline');
      })
    );
  }
});

// 9. Push Notifications Handler
self.addEventListener('push', (event) => {
  if (!event.data) return;

  const data = event.data.json();
  const options = {
    body: data.body,
    icon: '/icons/icon-192x192.png',
    badge: '/icons/badge-72x72.png',
    vibrate: [100, 50, 100],
    data: {
      dateOfArrival: Date.now(),
      primaryKey: data.id,
      url: data.url || '/'
    },
    actions: [
      {
        action: 'view',
        title: 'Ver',
        icon: '/icons/checkmark.png'
      },
      {
        action: 'close',
        title: 'Cerrar',
        icon: '/icons/xmark.png'
      }
    ],
    tag: data.tag || 'notification',
    requireInteraction: data.requireInteraction || false
  };

  event.waitUntil(
    self.registration.showNotification(data.title, options)
  );
});

// 10. Notification Click Handler
self.addEventListener('notificationclick', (event) => {
  event.notification.close();

  if (event.action === 'view') {
    event.waitUntil(
      clients.openWindow(event.notification.data.url)
    );
  }
});

// 11. Background Sync for Failed Requests
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-bookings') {
    event.waitUntil(syncBookings());
  }
});

async function syncBookings() {
  try {
    const cache = await caches.open('failed-bookings');
    const failedRequests = await cache.keys();

    for (const request of failedRequests) {
      try {
        const response = await fetch(request);
        if (response.ok) {
          await cache.delete(request);
        }
      } catch (error) {
        console.error('Failed to sync booking:', error);
      }
    }
  } catch (error) {
    console.error('Error in sync:', error);
  }
}

// 12. Periodic Background Sync (if supported)
self.addEventListener('periodicsync', (event) => {
  if (event.tag === 'update-calendar') {
    event.waitUntil(updateCalendarData());
  }
});

async function updateCalendarData() {
  try {
    const response = await fetch('/api/calendar/sync');
    const data = await response.json();
    
    // Update cache with fresh data
    const cache = await caches.open('calendar-cache');
    await cache.put('/api/calendar/data', new Response(JSON.stringify(data)));
  } catch (error) {
    console.error('Failed to update calendar:', error);
  }
}

// 13. Skip Waiting and Claim Clients
self.addEventListener('message', (event) => {
  if (event.data && event.data.type === 'SKIP_WAITING') {
    self.skipWaiting();
  }
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    (async () => {
      // Clean up old caches
      const cacheNames = await caches.keys();
      await Promise.all(
        cacheNames
          .filter((cacheName) => !Object.values(CACHE_NAMES).includes(cacheName))
          .map((cacheName) => caches.delete(cacheName))
      );

      // Claim clients
      await self.clients.claim();
    })()
  );
});
```

### 2.3 Mobile-First Design con Animaciones GPU-Accelerated

```scss
// styles/mobile-first.scss
// Base mobile styles (Mobile First)
:root {
  // Design tokens
  --spacing-xs: 0.25rem;
  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;
  --spacing-lg: 1.5rem;
  --spacing-xl: 2rem;
  --spacing-2xl: 3rem;

  // Animation timings
  --transition-fast: 150ms;
  --transition-normal: 300ms;
  --transition-slow: 500ms;

  // Easing functions
  --ease-out-expo: cubic-bezier(0.19, 1, 0.22, 1);
  --ease-in-out-expo: cubic-bezier(0.87, 0, 0.13, 1);
  --ease-spring: cubic-bezier(0.43, 0.195, 0.02, 0.96);

  // Shadows (optimized for performance)
  --shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
  --shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
  --shadow-xl: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
}

// GPU-Accelerated animations mixin
@mixin gpu-accelerated {
  transform: translateZ(0);
  backface-visibility: hidden;
  perspective: 1000px;
  will-change: transform;
}

// Smooth scroll for mobile
html {
  scroll-behavior: smooth;
  -webkit-overflow-scrolling: touch;
  
  @media (prefers-reduced-motion: reduce) {
    scroll-behavior: auto;
  }
}

// Base mobile layout
.container {
  width: 100%;
  padding: var(--spacing-md);
  margin: 0 auto;

  @media (min-width: 640px) {
    max-width: 640px;
    padding: var(--spacing-lg);
  }

  @media (min-width: 768px) {
    max-width: 768px;
  }

  @media (min-width: 1024px) {
    max-width: 1024px;
    padding: var(--spacing-xl);
  }

  @media (min-width: 1280px) {
    max-width: 1280px;
  }
}

// Touch-optimized buttons
.btn {
  @include gpu-accelerated;
  
  // Minimum touch target size (48x48px)
  min-height: 48px;
  min-width: 48px;
  padding: var(--spacing-sm) var(--spacing-lg);
  
  display: inline-flex;
  align-items: center;
  justify-content: center;
  
  font-weight: 600;
  text-decoration: none;
  border-radius: 8px;
  
  transition: all var(--transition-fast) var(--ease-out-expo);
  
  // Visual feedback for touch
  &:active {
    transform: translateZ(0) scale(0.98);
  }
  
  // Hover effects for desktop
  @media (hover: hover) {
    &:hover {
      transform: translateZ(0) translateY(-2px);
      box-shadow: var(--shadow-lg);
    }
  }
}

// Animated card component
.card {
  @include gpu-accelerated;
  
  background: var(--card-bg);
  border-radius: 12px;
  padding: var(--spacing-lg);
  box-shadow: var(--shadow-md);
  
  transition: all var(--transition-normal) var(--ease-out-expo);
  
  &.card--interactive {
    cursor: pointer;
    
    &:active {
      transform: translateZ(0) scale(0.98);
    }
    
    @media (hover: hover) {
      &:hover {
        transform: translateZ(0) translateY(-4px);
        box-shadow: var(--shadow-xl);
      }
    }
  }
}

// Skeleton loading animation
@keyframes skeleton-pulse {
  0% {
    opacity: 0.6;
  }
  50% {
    opacity: 1;
  }
  100% {
    opacity: 0.6;
  }
}

.skeleton {
  @include gpu-accelerated;
  
  background: linear-gradient(
    90deg,
    rgba(255, 255, 255, 0.1) 0%,
    rgba(255, 255, 255, 0.3) 50%,
    rgba(255, 255, 255, 0.1) 100%
  );
  background-size: 200% 100%;
  animation: skeleton-pulse 1.5s ease-in-out infinite;
  border-radius: 4px;
}

// Slide-in animation
@keyframes slide-in-bottom {
  from {
    transform: translateZ(0) translateY(100%);
    opacity: 0;
  }
  to {
    transform: translateZ(0) translateY(0);
    opacity: 1;
  }
}

.animate-slide-in {
  @include gpu-accelerated;
  animation: slide-in-bottom var(--transition-normal) var(--ease-out-expo) both;
}

// Fade and scale animation
@keyframes fade-scale-in {
  from {
    transform: translateZ(0) scale(0.9);
    opacity: 0;
  }
  to {
    transform: translateZ(0) scale(1);
    opacity: 1;
  }
}

.animate-fade-scale {
  @include gpu-accelerated;
  animation: fade-scale-in var(--transition-normal) var(--ease-spring) both;
}

// Optimized image loading
.image-container {
  position: relative;
  overflow: hidden;
  background-color: var(--skeleton-bg);
  
  img {
    @include gpu-accelerated;
    
    width: 100%;
    height: 100%;
    object-fit: cover;
    opacity: 0;
    transition: opacity var(--transition-normal) ease-in-out;
    
    &.loaded {
      opacity: 1;
    }
  }
  
  // Blur-up effect
  .image-placeholder {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    filter: blur(20px);
    transform: scale(1.1);
    opacity: 1;
    transition: opacity var(--transition-slow) ease-out;
    
    &.hidden {
      opacity: 0;
    }
  }
}

// Mobile navigation with gestures
.mobile-nav {
  @include gpu-accelerated;
  
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  
  background: var(--nav-bg);
  box-shadow: var(--shadow-xl);
  
  transform: translateZ(0) translateY(0);
  transition: transform var(--transition-normal) var(--ease-out-expo);
  
  &.hidden {
    transform: translateZ(0) translateY(100%);
  }
  
  // Safe area for iPhone X+
  padding-bottom: env(safe-area-inset-bottom);
}

// Gesture indicators
.gesture-indicator {
  @include gpu-accelerated;
  
  width: 40px;
  height: 4px;
  background: rgba(0, 0, 0, 0.3);
  border-radius: 2px;
  margin: var(--spacing-sm) auto;
  
  transition: all var(--transition-fast) ease-out;
  
  &:hover {
    background: rgba(0, 0, 0, 0.5);
    transform: translateZ(0) scaleX(1.5);
  }
}
```

```tsx
// components/animations/gpu-accelerated.tsx
import { motion, useMotionValue, useTransform, useSpring } from 'framer-motion';
import { useRef, useEffect } from 'react';

// GPU-Accelerated Card Component
export const AnimatedCard = ({ children, delay = 0 }) => {
  const ref = useRef(null);
  const y = useMotionValue(0);
  const opacity = useTransform(y, [100, 0], [0, 1]);
  const scale = useTransform(y, [100, 0], [0.9, 1]);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          y.set(0);
        }
      },
      { threshold: 0.1 }
    );

    if (ref.current) {
      observer.observe(ref.current);
    }

    return () => observer.disconnect();
  }, [y]);

  return (
    <motion.div
      ref={ref}
      initial={{ y: 100, opacity: 0, scale: 0.9 }}
      style={{ y, opacity, scale }}
      transition={{
        type: "spring",
        stiffness: 100,
        damping: 15,
        delay
      }}
      whileHover={{
        scale: 1.02,
        transition: { duration: 0.2 }
      }}
      whileTap={{ scale: 0.98 }}
    >
      {children}
    </motion.div>
  );
};

// 3D Flip Card Animation
export const FlipCard = ({ front, back }) => {
  const [isFlipped, setIsFlipped] = useState(false);

  return (
    <motion.div
      className="flip-card-container"
      onClick={() => setIsFlipped(!isFlipped)}
      style={{
        transformStyle: "preserve-3d",
        width: "100%",
        height: "100%"
      }}
      animate={{ rotateY: isFlipped ? 180 : 0 }}
      transition={{
        type: "spring",
        stiffness: 100,
        damping: 20
      }}
    >
      <motion.div
        className="flip-card-front"
        style={{
          backfaceVisibility: "hidden",
          position: "absolute",
          width: "100%",
          height: "100%"
        }}
      >
        {front}
      </motion.div>
      
      <motion.div
        className="flip-card-back"
        style={{
          backfaceVisibility: "hidden",
          position: "absolute",
          width: "100%",
          height: "100%",
          rotateY: 180
        }}
      >
        {back}
      </motion.div>
    </motion.div>
  );
};

// Parallax Scroll Effect
export const ParallaxSection = ({ children, offset = 50 }) => {
  const ref = useRef(null);
  const { scrollY } = useScroll({
    target: ref,
    offset: ["start end", "end start"]
  });

  const y = useTransform(scrollY, [0, 1], [offset, -offset]);

  return (
    <motion.div
      ref={ref}
      style={{ y }}
      transition={{ type: "spring", stiffness: 100 }}
    >
      {children}
    </motion.div>
  );
};

// Gesture-Enabled Drawer
export const SwipeableDrawer = ({ children, onClose }) => {
  const y = useMotionValue(0);
  const input = [-200, 0, 200];
  const output = [0, 1, 0];
  const opacity = useTransform(y, input, output);

  return (
    <motion.div
      drag="y"
      dragConstraints={{ top: 0 }}
      dragElastic={0.2}
      onDragEnd={(e, { velocity, offset }) => {
        if (velocity.y > 500 || offset.y > 100) {
          onClose();
        } else {
          y.set(0);
        }
      }}
      style={{ y, opacity }}
      className="swipeable-drawer"
    >
      <div className="gesture-indicator" />
      {children}
    </motion.div>
  );
};

// Stagger Animation for Lists
export const StaggerList = ({ children }) => {
  const containerVariants = {
    hidden: { opacity: 0 },
    visible: {
      opacity: 1,
      transition: {
        staggerChildren: 0.1
      }
    }
  };

  const itemVariants = {
    hidden: { y: 20, opacity: 0 },
    visible: {
      y: 0,
      opacity: 1,
      transition: {
        type: "spring",
        stiffness: 100
      }
    }
  };

  return (
    <motion.div
      variants={containerVariants}
      initial="hidden"
      animate="visible"
    >
      {React.Children.map(children, (child, index) => (
        <motion.div key={index} variants={itemVariants}>
          {child}
        </motion.div>
      ))}
    </motion.div>
  );
};
```

### 2.4 Cross-Device Sync Implementation

```typescript
// lib/sync/cross-device-sync.ts
import { EventEmitter } from 'events';

interface SyncData {
  deviceId: string;
  timestamp: number;
  action: string;
  data: any;
}

export class CrossDeviceSync extends EventEmitter {
  private ws: WebSocket | null = null;
  private deviceId: string;
  private syncQueue: SyncData[] = [];
  private isOnline: boolean = navigator.onLine;
  private retryCount: number = 0;
  private maxRetries: number = 5;

  constructor(private userId: string, private wsUrl: string) {
    super();
    this.deviceId = this.generateDeviceId();
    this.initializeSync();
    this.setupEventListeners();
  }

  private generateDeviceId(): string {
    const stored = localStorage.getItem('device-id');
    if (stored) return stored;

    const id = `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    localStorage.setItem('device-id', id);
    return id;
  }

  private initializeSync() {
    // Initialize WebSocket connection
    this.connectWebSocket();

    // Setup Broadcast Channel for same-origin sync
    if ('BroadcastChannel' in window) {
      this.setupBroadcastChannel();
    }

    // Setup Service Worker sync
    if ('serviceWorker' in navigator && 'SyncManager' in window) {
      this.setupBackgroundSync();
    }

    // Setup visibility change handler
    document.addEventListener('visibilitychange', () => {
      if (!document.hidden) {
        this.syncPendingData();
      }
    });
  }

  private connectWebSocket() {
    try {
      this.ws = new WebSocket(`${this.wsUrl}?userId=${this.userId}&deviceId=${this.deviceId}`);

      this.ws.onopen = () => {
        console.log('WebSocket connected for cross-device sync');
        this.retryCount = 0;
        this.emit('connected');
        this.syncPendingData();
      };

      this.ws.onmessage = (event) => {
        try {
          const data = JSON.parse(event.data);
          this.handleIncomingSync(data);
        } catch (error) {
          console.error('Error parsing sync data:', error);
        }
      };

      this.ws.onclose = () => {
        console.log('WebSocket disconnected');
        this.emit('disconnected');
        this.scheduleReconnect();
      };

      this.ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        this.emit('error', error);
      };
    } catch (error) {
      console.error('Failed to connect WebSocket:', error);
      this.scheduleReconnect();
    }
  }

  private scheduleReconnect() {
    if (this.retryCount >= this.maxRetries) {
      console.error('Max reconnection attempts reached');
      return;
    }

    const delay = Math.min(1000 * Math.pow(2, this.retryCount), 30000);
    this.retryCount++;

    setTimeout(() => {
      if (this.isOnline) {
        this.connectWebSocket();
      }
    }, delay);
  }

  private setupBroadcastChannel() {
    const channel = new BroadcastChannel(`sync-${this.userId}`);
    
    channel.onmessage = (event) => {
      if (event.data.deviceId !== this.deviceId) {
        this.handleIncomingSync(event.data);
      }
    };

    // Override emit to also broadcast locally
    const originalEmit = this.emit.bind(this);
    this.emit = (event: string, data?: any) => {
      originalEmit(event, data);
      
      if (event === 'sync') {
        channel.postMessage({
          deviceId: this.deviceId,
          timestamp: Date.now(),
          action: data.action,
          data: data.data
        });
      }
    };
  }

  private setupBackgroundSync() {
    navigator.serviceWorker.ready.then((registration) => {
      registration.sync.register('sync-data').catch((error) => {
        console.error('Background sync registration failed:', error);
      });
    });

    // Listen for sync complete events from service worker
    navigator.serviceWorker.addEventListener('message', (event) => {
      if (event.data.type === 'sync-complete') {
        this.handleSyncComplete(event.data.results);
      }
    });
  }

  private setupEventListeners() {
    // Online/Offline detection
    window.addEventListener('online', () => {
      this.isOnline = true;
      this.emit('online');
      this.connectWebSocket();
      this.syncPendingData();
    });

    window.addEventListener('offline', () => {
      this.isOnline = false;
      this.emit('offline');
    });

    // Page visibility for battery optimization
    document.addEventListener('visibilitychange', () => {
      if (document.hidden && this.ws) {
        // Reduce activity when page is hidden
        this.ws.close();
      } else if (!document.hidden && this.isOnline) {
        this.connectWebSocket();
      }
    });
  }

  // Public API

  public sync(action: string, data: any) {
    const syncData: SyncData = {
      deviceId: this.deviceId,
      timestamp: Date.now(),
      action,
      data
    };

    // Add to queue
    this.syncQueue.push(syncData);

    // Try to sync immediately if online
    if (this.isOnline && this.ws?.readyState === WebSocket.OPEN) {
      this.sendSyncData(syncData);
    } else {
      // Store for later sync
      this.storePendingSync(syncData);
    }

    // Emit local event
    this.emit('sync', syncData);
  }

  private sendSyncData(data: SyncData) {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify({
        type: 'sync',
        ...data
      }));

      // Remove from queue
      this.syncQueue = this.syncQueue.filter(item => 
        item.timestamp !== data.timestamp
      );
    }
  }

  private async syncPendingData() {
    // Get stored pending syncs
    const pending = await this.getPendingSync();
    
    for (const data of pending) {
      this.sendSyncData(data);
    }
  }

  private storePendingSync(data: SyncData) {
    const stored = localStorage.getItem('pending-sync') || '[]';
    const pending = JSON.parse(stored);
    pending.push(data);
    
    // Keep only last 100 items
    if (pending.length > 100) {
      pending.splice(0, pending.length - 100);
    }
    
    localStorage.setItem('pending-sync', JSON.stringify(pending));
  }

  private async getPendingSync(): Promise<SyncData[]> {
    const stored = localStorage.getItem('pending-sync') || '[]';
    return JSON.parse(stored);
  }

  private handleIncomingSync(data: SyncData) {
    if (data.deviceId === this.deviceId) return;

    this.emit('remote-sync', data);

    // Handle specific sync actions
    switch (data.action) {
      case 'booking-created':
        this.emit('booking-created', data.data);
        break;
      
      case 'booking-updated':
        this.emit('booking-updated', data.data);
        break;
      
      case 'preference-changed':
        this.emit('preference-changed', data.data);
        break;
      
      case 'cart-updated':
        this.emit('cart-updated', data.data);
        break;
      
      default:
        this.emit('unknown-sync', data);
    }
  }

  private handleSyncComplete(results: any[]) {
    console.log('Background sync completed:', results);
    this.emit('background-sync-complete', results);
    
    // Clear pending sync data
    localStorage.removeItem('pending-sync');
  }

  public getDeviceInfo() {
    return {
      deviceId: this.deviceId,
      isOnline: this.isOnline,
      pendingSync: this.syncQueue.length,
      connected: this.ws?.readyState === WebSocket.OPEN
    };
  }

  public destroy() {
    if (this.ws) {
      this.ws.close();
    }
    this.removeAllListeners();
  }
}

// React Hook for Cross-Device Sync
export function useCrossDeviceSync(userId: string) {
  const [sync, setSync] = useState<CrossDeviceSync | null>(null);
  const [isConnected, setIsConnected] = useState(false);
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const syncInstance = new CrossDeviceSync(
      userId,
      process.env.NEXT_PUBLIC_WS_URL || 'wss://api.tu-dominio.com/sync'
    );

    syncInstance.on('connected', () => setIsConnected(true));
    syncInstance.on('disconnected', () => setIsConnected(false));
    syncInstance.on('online', () => setIsOnline(true));
    syncInstance.on('offline', () => setIsOnline(false));

    setSync(syncInstance);

    return () => {
      syncInstance.destroy();
    };
  }, [userId]);

  return {
    sync,
    isConnected,
    isOnline,
    syncData: (action: string, data: any) => {
      sync?.sync(action, data);
    }
  };
}
```

---

## 3. IA Y EMBEDDINGS

### 3.1 Configuración de Llama 3.1 con Temperatura 0.7

```python
# config/llama_config.py
import os
from typing import Dict, Any, Optional
from dataclasses import dataclass
import json

@dataclass
class LlamaConfig:
    """Configuration for Llama 3.1 model"""
    model_name: str = "llama3.1:70b"
    temperature: float = 0.7
    top_p: float = 0.9
    top_k: int = 40
    num_predict: int = 2048
    num_ctx: int = 4096
    repeat_penalty: float = 1.1
    seed: Optional[int] = None
    stop_sequences: list = None
    
    def __post_init__(self):
        if self.stop_sequences is None:
            self.stop_sequences = ["<|eot_id|>", "<|end|>", "Human:", "Assistant:"]
    
    def to_dict(self) -> Dict[str, Any]:
        return {
            "model": self.model_name,
            "options": {
                "temperature": self.temperature,
                "top_p": self.top_p,
                "top_k": self.top_k,
                "num_predict": self.num_predict,
                "num_ctx": self.num_ctx,
                "repeat_penalty": self.repeat_penalty,
                "seed": self.seed,
                "stop": self.stop_sequences
            }
        }

# Preset configurations for different use cases
LLAMA_PRESETS = {
    "balanced": LlamaConfig(
        temperature=0.7,
        top_p=0.9,
        top_k=40
    ),
    "creative": LlamaConfig(
        temperature=0.9,
        top_p=0.95,
        top_k=100,
        repeat_penalty=1.0
    ),
    "precise": LlamaConfig(
        temperature=0.3,
        top_p=0.85,
        top_k=20,
        repeat_penalty=1.2
    ),
    "customer_service": LlamaConfig(
        temperature=0.5,
        top_p=0.9,
        top_k=30,
        num_predict=512,
        stop_sequences=["Human:", "Customer:", "Agent:"]
    ),
    "analysis": LlamaConfig(
        temperature=0.4,
        top_p=0.88,
        top_k=25,
        num_predict=4096
    )
}
```

```python
# services/llama_service.py
import aiohttp
import asyncio
from typing import Dict, Any, AsyncGenerator, Optional
import json
import time
from functools import lru_cache
import redis.asyncio as redis
import hashlib

class LlamaService:
    def __init__(self, base_url: str = "http://llama:11434", redis_url: str = "redis://redis:6379"):
        self.base_url = base_url
        self.redis_client = None
        self.redis_url = redis_url
        self.session = None
        
    async def __aenter__(self):
        self.session = aiohttp.ClientSession()
        self.redis_client = await redis.from_url(self.redis_url)
        return self
        
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.session:
            await self.session.close()
        if self.redis_client:
            await self.redis_client.close()
    
    def _generate_cache_key(self, prompt: str, config: Dict[str, Any]) -> str:
        """Generate a unique cache key based on prompt and config"""
        content = f"{prompt}{json.dumps(config, sort_keys=True)}"
        return f"llama:cache:{hashlib.md5(content.encode()).hexdigest()}"
    
    async def generate(
        self, 
        prompt: str, 
        preset: str = "balanced",
        context: Optional[str] = None,
        use_cache: bool = True,
        stream: bool = False
    ) -> Any:
        """Generate response from Llama model"""
        
        # Get configuration
        config = LLAMA_PRESETS.get(preset, LLAMA_PRESETS["balanced"]).to_dict()
        
        # Build the full prompt
        full_prompt = self._build_prompt(prompt, context)
        
        # Check cache if enabled
        if use_cache and not stream:
            cache_key = self._generate_cache_key(full_prompt, config)
            cached = await self._get_cached_response(cache_key)
            if cached:
                return cached
        
        # Prepare request
        payload = {
            **config,
            "prompt": full_prompt,
            "stream": stream
        }
        
        # Make request
        if stream:
            return self._stream_response(payload)
        else:
            response = await self._generate_response(payload)
            
            # Cache response if enabled
            if use_cache and response:
                await self._cache_response(cache_key, response)
            
            return response
    
    def _build_prompt(self, prompt: str, context: Optional[str] = None) -> str:
        """Build a properly formatted prompt"""
        system_prompt = """You are a helpful AI assistant for a premium booking and experiences platform. 
You provide accurate, helpful, and friendly responses while maintaining professionalism."""
        
        if context:
            full_prompt = f"""<|begin_of_text|><|start_header_id|>system<|end_header_id|>
{system_prompt}

Context: {context}<|eot_id|>
<|start_header_id|>user<|end_header_id|>
{prompt}<|eot_id|>
<|start_header_id|>assistant<|end_header_id|>"""
        else:
            full_prompt = f"""<|begin_of_text|><|start_header_id|>system<|end_header_id|>
{system_prompt}<|eot_id|>
<|start_header_id|>user<|end_header_id|>
{prompt}<|eot_id|>
<|start_header_id|>assistant<|end_header_id|>"""
        
        return full_prompt
    
    async def _generate_response(self, payload: Dict[str, Any]) -> Dict[str, Any]:
        """Generate a non-streaming response"""
        try:
            async with self.session.post(
                f"{self.base_url}/api/generate",
                json=payload,
                timeout=aiohttp.ClientTimeout(total=60)
            ) as response:
                if response.status == 200:
                    data = await response.json()
                    return {
                        "response": data.get("response", ""),
                        "model": data.get("model"),
                        "created_at": data.get("created_at"),
                        "done": data.get("done", True),
                        "context": data.get("context", []),
                        "total_duration": data.get("total_duration"),
                        "eval_count": data.get("eval_count"),
                        "eval_duration": data.get("eval_duration")
                    }
                else:
                    error_text = await response.text()
                    raise Exception(f"Llama API error: {response.status} - {error_text}")
        except asyncio.TimeoutError:
            raise Exception("Llama API request timed out")
        except Exception as e:
            raise Exception(f"Error calling Llama API: {str(e)}")
    
    async def _stream_response(self, payload: Dict[str, Any]) -> AsyncGenerator[str, None]:
        """Stream response from Llama"""
        try:
            async with self.session.post(
                f"{self.base_url}/api/generate",
                json=payload
            ) as response:
                async for line in response.content:
                    if line:
                        try:
                            data = json.loads(line)
                            if "response" in data:
                                yield data["response"]
                        except json.JSONDecodeError:
                            continue
        except Exception as e:
            yield f"Error: {str(e)}"
    
    async def _get_cached_response(self, key: str) -> Optional[Dict[str, Any]]:
        """Get cached response from Redis"""
        try:
            cached = await self.redis_client.get(key)
            if cached:
                return json.loads(cached)
        except Exception as e:
            print(f"Cache retrieval error: {e}")
        return None
    
    async def _cache_response(self, key: str, response: Dict[str, Any], ttl: int = 3600):
        """Cache response in Redis"""
        try:
            await self.redis_client.setex(
                key, 
                ttl, 
                json.dumps(response)
            )
        except Exception as e:
            print(f"Cache storage error: {e}")
    
    async def analyze_sentiment(self, text: str) -> Dict[str, Any]:
        """Analyze sentiment of text"""
        prompt = f"""Analyze the sentiment of the following text and provide:
1. Overall sentiment (positive, negative, neutral)
2. Confidence score (0-1)
3. Key emotions detected
4. Brief explanation

Text: {text}

Provide response in JSON format."""
        
        response = await self.generate(prompt, preset="analysis")
        
        try:
            # Parse JSON from response
            result = json.loads(response["response"])
            return result
        except:
            # Fallback parsing
            return {
                "sentiment": "neutral",
                "confidence": 0.5,
                "emotions": [],
                "explanation": response["response"]
            }
    
    async def generate_booking_recommendation(
        self, 
        customer_profile: Dict[str, Any],
        available_experiences: list
    ) -> Dict[str, Any]:
        """Generate personalized booking recommendations"""
        
        context = f"""Customer Profile:
- Name: {customer_profile.get('name')}
- Past Bookings: {', '.join(customer_profile.get('past_bookings', []))}
- Preferences: {', '.join(customer_profile.get('preferences', []))}
- Budget Range: {customer_profile.get('budget_range')}

Available Experiences:
{json.dumps(available_experiences, indent=2)}"""
        
        prompt = """Based on the customer profile and available experiences, provide:
1. Top 3 recommended experiences with reasoning
2. Personalized message for the customer
3. Alternative suggestions if budget is a concern

Format as structured JSON."""
        
        response = await self.generate(
            prompt, 
            preset="customer_service",
            context=context
        )
        
        return response
```

### 3.2 mxbai-embed-large con 1024 Dimensiones

```python
# services/embedding_service.py
import numpy as np
import aiohttp
import asyncio
from typing import List, Dict, Any, Optional, Tuple
import json
import hashlib
from sklearn.metrics.pairwise import cosine_similarity
import redis.asyncio as redis

class EmbeddingService:
    def __init__(
        self, 
        base_url: str = "http://embeddings:8080",
        model_name: str = "mxbai-embed-large",
        dimensions: int = 1024,
        redis_url: str = "redis://redis:6379"
    ):
        self.base_url = base_url
        self.model_name = model_name
        self.dimensions = dimensions
        self.redis_url = redis_url
        self.session = None
        self.redis_client = None
        
    async def __aenter__(self):
        self.session = aiohttp.ClientSession()
        self.redis_client = await redis.from_url(self.redis_url)
        return self
        
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.session:
            await self.session.close()
        if self.redis_client:
            await self.redis_client.close()
    
    def _generate_cache_key(self, text: str) -> str:
        """Generate cache key for embedding"""
        return f"embedding:{self.model_name}:{hashlib.md5(text.encode()).hexdigest()}"
    
    async def generate_embedding(
        self, 
        text: str,
        use_cache: bool = True
    ) -> np.ndarray:
        """Generate embedding for a single text"""
        
        # Check cache
        if use_cache:
            cached = await self._get_cached_embedding(text)
            if cached is not None:
                return cached
        
        # Generate embedding
        embedding = await self._call_embedding_api(text)
        
        # Cache result
        if use_cache and embedding is not None:
            await self._cache_embedding(text, embedding)
        
        return embedding
    
    async def generate_embeddings_batch(
        self,
        texts: List[str],
        batch_size: int = 32,
        use_cache: bool = True
    ) -> List[np.ndarray]:
        """Generate embeddings for multiple texts efficiently"""
        
        embeddings = []
        uncached_texts = []
        uncached_indices = []
        
        # Check cache for each text
        if use_cache:
            for i, text in enumerate(texts):
                cached = await self._get_cached_embedding(text)
                if cached is not None:
                    embeddings.append(cached)
                else:
                    uncached_texts.append(text)
                    uncached_indices.append(i)
                    embeddings.append(None)
        else:
            uncached_texts = texts
            uncached_indices = list(range(len(texts)))
            embeddings = [None] * len(texts)
        
        # Process uncached texts in batches
        for i in range(0, len(uncached_texts), batch_size):
            batch = uncached_texts[i:i + batch_size]
            batch_embeddings = await self._call_embedding_api_batch(batch)
            
            # Fill in the embeddings
            for j, embedding in enumerate(batch_embeddings):
                idx = uncached_indices[i + j]
                embeddings[idx] = embedding
                
                # Cache the embedding
                if use_cache:
                    await self._cache_embedding(uncached_texts[i + j], embedding)
        
        return embeddings
    
    async def _call_embedding_api(self, text: str) -> np.ndarray:
        """Call the embedding API for a single text"""
        try:
            async with self.session.post(
                f"{self.base_url}/embed",
                json={
                    "text": text,
                    "model": self.model_name
                },
                timeout=aiohttp.ClientTimeout(total=30)
            ) as response:
                if response.status == 200:
                    data = await response.json()
                    embedding = np.array(data["embedding"], dtype=np.float32)
                    
                    # Validate dimensions
                    if len(embedding) != self.dimensions:
                        raise ValueError(
                            f"Expected {self.dimensions} dimensions, "
                            f"got {len(embedding)}"
                        )
                    
                    # Normalize embedding
                    embedding = embedding / np.linalg.norm(embedding)
                    
                    return embedding
                else:
                    raise Exception(f"Embedding API error: {response.status}")
        except Exception as e:
            print(f"Error generating embedding: {e}")
            return None
    
    async def _call_embedding_api_batch(self, texts: List[str]) -> List[np.ndarray]:
        """Call the embedding API for multiple texts"""
        try:
            async with self.session.post(
                f"{self.base_url}/embed/batch",
                json={
                    "texts": texts,
                    "model": self.model_name
                },
                timeout=aiohttp.ClientTimeout(total=60)
            ) as response:
                if response.status == 200:
                    data = await response.json()
                    embeddings = []
                    
                    for emb_data in data["embeddings"]:
                        embedding = np.array(emb_data, dtype=np.float32)
                        # Normalize
                        embedding = embedding / np.linalg.norm(embedding)
                        embeddings.append(embedding)
                    
                    return embeddings
                else:
                    raise Exception(f"Batch embedding API error: {response.status}")
        except Exception as e:
            print(f"Error generating batch embeddings: {e}")
            # Fallback to individual embeddings
            embeddings = []
            for text in texts:
                emb = await self._call_embedding_api(text)
                embeddings.append(emb)
            return embeddings
    
    async def _get_cached_embedding(self, text: str) -> Optional[np.ndarray]:
        """Get cached embedding from Redis"""
        try:
            key = self._generate_cache_key(text)
            cached = await self.redis_client.get(key)
            
            if cached:
                # Deserialize numpy array
                embedding = np.frombuffer(cached, dtype=np.float32)
                return embedding.reshape(-1)
        except Exception as e:
            print(f"Cache retrieval error: {e}")
        return None
    
    async def _cache_embedding(self, text: str, embedding: np.ndarray, ttl: int = 86400):
        """Cache embedding in Redis"""
        try:
            key = self._generate_cache_key(text)
            # Serialize numpy array
            serialized = embedding.astype(np.float32).tobytes()
            await self.redis_client.setex(key, ttl, serialized)
        except Exception as e:
            print(f"Cache storage error: {e}")
    
    def calculate_similarity(
        self, 
        embedding1: np.ndarray, 
        embedding2: np.ndarray
    ) -> float:
        """Calculate cosine similarity between two embeddings"""
        return float(cosine_similarity(
            embedding1.reshape(1, -1),
            embedding2.reshape(1, -1)
        )[0][0])
    
    def find_most_similar(
        self,
        query_embedding: np.ndarray,
        embeddings: List[np.ndarray],
        top_k: int = 5,
        threshold: float = 0.0
    ) -> List[Tuple[int, float]]:
        """Find most similar embeddings to query"""
        
        similarities = []
        for i, embedding in enumerate(embeddings):
            sim = self.calculate_similarity(query_embedding, embedding)
            if sim >= threshold:
                similarities.append((i, sim))
        
        # Sort by similarity descending
        similarities.sort(key=lambda x: x[1], reverse=True)
        
        return similarities[:top_k]
```

### 3.3 Búsqueda Híbrida: Vectorial + Texto

```python
# services/hybrid_search_service.py
import asyncpg
import numpy as np
from typing import List, Dict, Any, Tuple, Optional
from datetime import datetime
import re
from fuzzywuzzy import fuzz
import asyncio

class HybridSearchService:
    def __init__(
        self,
        db_url: str,
        embedding_service: EmbeddingService,
        llama_service: LlamaService
    ):
        self.db_url = db_url
        self.embedding_service = embedding_service
        self.llama_service = llama_service
        self.pool = None
        
    async def __aenter__(self):
        self.pool = await asyncpg.create_pool(self.db_url)
        await self.embedding_service.__aenter__()
        await self.llama_service.__aenter__()
        return self
        
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.pool:
            await self.pool.close()
        await self.embedding_service.__aexit__(exc_type, exc_val, exc_tb)
        await self.llama_service.__aexit__(exc_type, exc_val, exc_tb)
    
    async def setup_database(self):
        """Setup database tables and indexes"""
        async with self.pool.acquire() as conn:
            # Create tables
            await conn.execute("""
                CREATE TABLE IF NOT EXISTS documents (
                    id SERIAL PRIMARY KEY,
                    title TEXT NOT NULL,
                    content TEXT NOT NULL,
                    metadata JSONB DEFAULT '{}',
                    embedding vector(1024),
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    
                    -- Full text search
                    search_vector tsvector GENERATED ALWAYS AS (
                        setweight(to_tsvector('spanish', coalesce(title, '')), 'A') ||
                        setweight(to_tsvector('spanish', coalesce(content, '')), 'B')
                    ) STORED
                );
                
                -- Vector index (HNSW)
                CREATE INDEX IF NOT EXISTS idx_documents_embedding 
                ON documents USING hnsw (embedding vector_cosine_ops)
                WITH (m = 16, ef_construction = 64);
                
                -- Full text search index
                CREATE INDEX IF NOT EXISTS idx_documents_search 
                ON documents USING GIN (search_vector);
                
                -- Metadata index
                CREATE INDEX IF NOT EXISTS idx_documents_metadata 
                ON documents USING GIN (metadata);
                
                -- Create experiences table
                CREATE TABLE IF NOT EXISTS experiences (
                    id SERIAL PRIMARY KEY,
                    name TEXT NOT NULL,
                    description TEXT NOT NULL,
                    category TEXT NOT NULL,
                    price DECIMAL(10, 2),
                    duration_minutes INTEGER,
                    location TEXT,
                    images JSONB DEFAULT '[]',
                    features JSONB DEFAULT '[]',
                    availability JSONB DEFAULT '{}',
                    embedding vector(1024),
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    
                    search_vector tsvector GENERATED ALWAYS AS (
                        setweight(to_tsvector('spanish', coalesce(name, '')), 'A') ||
                        setweight(to_tsvector('spanish', coalesce(description, '')), 'B') ||
                        setweight(to_tsvector('spanish', coalesce(category, '')), 'C')
                    ) STORED
                );
                
                -- Indexes for experiences
                CREATE INDEX IF NOT EXISTS idx_experiences_embedding 
                ON experiences USING hnsw (embedding vector_cosine_ops);
                
                CREATE INDEX IF NOT EXISTS idx_experiences_search 
                ON experiences USING GIN (search_vector);
                
                CREATE INDEX IF NOT EXISTS idx_experiences_category 
                ON experiences (category);
                
                CREATE INDEX IF NOT EXISTS idx_experiences_price 
                ON experiences (price);
            """)
    
    async def index_document(
        self,
        title: str,
        content: str,
        metadata: Dict[str, Any] = None
    ) -> int:
        """Index a document with embedding"""
        
        # Generate embedding
        full_text = f"{title} {content}"
        embedding = await self.embedding_service.generate_embedding(full_text)
        
        async with self.pool.acquire() as conn:
            doc_id = await conn.fetchval("""
                INSERT INTO documents (title, content, metadata, embedding)
                VALUES ($1, $2, $3, $4)
                RETURNING id
            """, title, content, metadata or {}, embedding.tolist())
            
        return doc_id
    
    async def hybrid_search(
        self,
        query: str,
        limit: int = 10,
        filters: Optional[Dict[str, Any]] = None,
        weights: Dict[str, float] = None
    ) -> List[Dict[str, Any]]:
        """Perform hybrid search combining vector and text search"""
        
        if weights is None:
            weights = {
                "vector": 0.6,
                "text": 0.3,
                "fuzzy": 0.1
            }
        
        # Generate query embedding
        query_embedding = await self.embedding_service.generate_embedding(query)
        
        # Prepare filter conditions
        filter_conditions = []
        filter_params = []
        param_count = 4  # Starting after the main parameters
        
        if filters:
            for key, value in filters.items():
                if key == "category":
                    filter_conditions.append(f"category = ${param_count}")
                    filter_params.append(value)
                    param_count += 1
                elif key == "price_range":
                    filter_conditions.append(
                        f"price BETWEEN ${param_count} AND ${param_count + 1}"
                    )
                    filter_params.extend(value)
                    param_count += 2
                elif key == "metadata":
                    for meta_key, meta_value in value.items():
                        filter_conditions.append(
                            f"metadata->>${param_count} = ${param_count + 1}"
                        )
                        filter_params.extend([meta_key, meta_value])
                        param_count += 2
        
        filter_clause = ""
        if filter_conditions:
            filter_clause = "AND " + " AND ".join(filter_conditions)
        
        # Execute hybrid search query
        async with self.pool.acquire() as conn:
            results = await conn.fetch(f"""
                WITH vector_search AS (
                    SELECT 
                        id,
                        title,
                        content,
                        metadata,
                        1 - (embedding <=> $1::vector) as vector_similarity
                    FROM documents
                    WHERE 1 - (embedding <=> $1::vector) > 0.3
                    {filter_clause}
                    ORDER BY vector_similarity DESC
                    LIMIT {limit * 2}
                ),
                text_search AS (
                    SELECT 
                        id,
                        title,
                        content,
                        metadata,
                        ts_rank(search_vector, plainto_tsquery('spanish', $2)) as text_rank
                    FROM documents
                    WHERE search_vector @@ plainto_tsquery('spanish', $2)
                    {filter_clause}
                    ORDER BY text_rank DESC
                    LIMIT {limit * 2}
                ),
                combined_results AS (
                    SELECT 
                        COALESCE(v.id, t.id) as id,
                        COALESCE(v.title, t.title) as title,
                        COALESCE(v.content, t.content) as content,
                        COALESCE(v.metadata, t.metadata) as metadata,
                        COALESCE(v.vector_similarity, 0) * $3 as vector_score,
                        COALESCE(t.text_rank, 0) * $4 as text_score
                    FROM vector_search v
                    FULL OUTER JOIN text_search t ON v.id = t.id
                )
                SELECT 
                    id,
                    title,
                    content,
                    metadata,
                    vector_score,
                    text_score,
                    (vector_score + text_score) as total_score
                FROM combined_results
                ORDER BY total_score DESC
                LIMIT {limit}
            """, 
                query_embedding.tolist(), 
                query,
                weights["vector"],
                weights["text"],
                *filter_params
            )
            
            # Add fuzzy matching score
            results_with_fuzzy = []
            for row in results:
                result = dict(row)
                
                # Calculate fuzzy match score
                title_fuzzy = fuzz.token_sort_ratio(query, result["title"]) / 100
                content_fuzzy = fuzz.token_sort_ratio(
                    query, 
                    result["content"][:200]  # First 200 chars
                ) / 100
                
                fuzzy_score = max(title_fuzzy, content_fuzzy) * weights["fuzzy"]
                
                result["fuzzy_score"] = fuzzy_score
                result["final_score"] = (
                    result["total_score"] + fuzzy_score
                )
                
                results_with_fuzzy.append(result)
            
            # Sort by final score
            results_with_fuzzy.sort(
                key=lambda x: x["final_score"], 
                reverse=True
            )
            
            return results_with_fuzzy[:limit]
    
    async def semantic_search_with_reranking(
        self,
        query: str,
        initial_results: int = 20,
        final_results: int = 10
    ) -> List[Dict[str, Any]]:
        """Perform semantic search with LLM-based reranking"""
        
        # Get initial results
        initial = await self.hybrid_search(query, limit=initial_results)
        
        # Prepare context for reranking
        documents = []
        for i, result in enumerate(initial):
            documents.append({
                "id": i,
                "title": result["title"],
                "content": result["content"][:500],  # Truncate for context
                "score": result["final_score"]
            })
        
        # Use LLM for reranking
        rerank_prompt = f"""Given the search query: "{query}"

And the following documents:
{json.dumps(documents, indent=2)}

Please rerank these documents based on their relevance to the query. 
Consider semantic meaning, context, and usefulness.

Return a JSON array of document IDs in order of relevance (most relevant first).
Only include the top {final_results} most relevant documents."""

        response = await self.llama_service.generate(
            rerank_prompt,
            preset="analysis"
        )
        
        try:
            reranked_ids = json.loads(response["response"])
            
            # Reorder results based on LLM ranking
            reranked_results = []
            for doc_id in reranked_ids[:final_results]:
                if doc_id < len(initial):
                    reranked_results.append(initial[doc_id])
            
            return reranked_results
        except:
            # Fallback to original ranking
            return initial[:final_results]
    
    async def find_similar_experiences(
        self,
        experience_id: int,
        limit: int = 5
    ) -> List[Dict[str, Any]]:
        """Find similar experiences based on embedding similarity"""
        
        async with self.pool.acquire() as conn:
            # Get the experience embedding
            source_embedding = await conn.fetchval("""
                SELECT embedding FROM experiences WHERE id = $1
            """, experience_id)
            
            if not source_embedding:
                return []
            
            # Find similar experiences
            results = await conn.fetch("""
                SELECT 
                    id,
                    name,
                    description,
                    category,
                    price,
                    1 - (embedding <=> $1::vector) as similarity
                FROM experiences
                WHERE id != $2
                    AND 1 - (embedding <=> $1::vector) > 0.5
                ORDER BY similarity DESC
                LIMIT $3
            """, source_embedding, experience_id, limit)
            
            return [dict(row) for row in results]
```

### 3.4 Cache de Embeddings en Redis

```python
# services/embedding_cache_service.py
import redis.asyncio as redis
import numpy as np
import json
import asyncio
from typing import List, Dict, Any, Optional, Set
from datetime import datetime, timedelta
import hashlib
import pickle
import lz4.frame

class EmbeddingCacheService:
    def __init__(
        self,
        redis_url: str = "redis://redis:6379",
        cache_prefix: str = "emb_cache",
        ttl_seconds: int = 86400 * 7,  # 7 days
        compression: bool = True
    ):
        self.redis_url = redis_url
        self.cache_prefix = cache_prefix
        self.ttl_seconds = ttl_seconds
        self.compression = compression
        self.redis_client = None
        self.stats = {
            "hits": 0,
            "misses": 0,
            "errors": 0
        }
        
    async def __aenter__(self):
        self.redis_client = await redis.from_url(
            self.redis_url,
            decode_responses=False  # We need binary for numpy arrays
        )
        return self
        
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.redis_client:
            await self.redis_client.close()
    
    def _generate_key(self, text: str, model: str = "default") -> str:
        """Generate cache key for text"""
        text_hash = hashlib.sha256(text.encode()).hexdigest()
        return f"{self.cache_prefix}:{model}:{text_hash}"
    
    def _serialize_embedding(self, embedding: np.ndarray) -> bytes:
        """Serialize numpy array with optional compression"""
        data = pickle.dumps(embedding)
        
        if self.compression:
            data = lz4.frame.compress(data)
            
        return data
    
    def _deserialize_embedding(self, data: bytes) -> np.ndarray:
        """Deserialize numpy array"""
        if self.compression:
            data = lz4.frame.decompress(data)
            
        return pickle.loads(data)
    
    async def get(
        self, 
        text: str, 
        model: str = "default"
    ) -> Optional[np.ndarray]:
        """Get cached embedding"""
        try:
            key = self._generate_key(text, model)
            data = await self.redis_client.get(key)
            
            if data:
                self.stats["hits"] += 1
                
                # Update access time for LRU
                await self.redis_client.expire(key, self.ttl_seconds)
                
                return self._deserialize_embedding(data)
            else:
                self.stats["misses"] += 1
                return None
                
        except Exception as e:
            self.stats["errors"] += 1
            print(f"Cache get error: {e}")
            return None
    
    async def set(
        self,
        text: str,
        embedding: np.ndarray,
        model: str = "default",
        metadata: Optional[Dict[str, Any]] = None
    ):
        """Cache embedding with metadata"""
        try:
            key = self._generate_key(text, model)
            
            # Serialize embedding
            data = self._serialize_embedding(embedding)
            
            # Store embedding
            await self.redis_client.setex(key, self.ttl_seconds, data)
            
            # Store metadata if provided
            if metadata:
                meta_key = f"{key}:meta"
                await self.redis_client.setex(
                    meta_key,
                    self.ttl_seconds,
                    json.dumps(metadata)
                )
            
            # Update index
            await self._update_index(text, model)
            
        except Exception as e:
            self.stats["errors"] += 1
            print(f"Cache set error: {e}")
    
    async def get_batch(
        self,
        texts: List[str],
        model: str = "default"
    ) -> Dict[str, Optional[np.ndarray]]:
        """Get multiple cached embeddings efficiently"""
        
        # Generate all keys
        keys = [self._generate_key(text, model) for text in texts]
        
        # Batch get from Redis
        try:
            values = await self.redis_client.mget(keys)
            
            results = {}
            for text, value in zip(texts, values):
                if value:
                    self.stats["hits"] += 1
                    results[text] = self._deserialize_embedding(value)
                else:
                    self.stats["misses"] += 1
                    results[text] = None
                    
            # Update expiration for hits
            hit_keys = [
                keys[i] for i, value in enumerate(values) if value
            ]
            if hit_keys:
                pipeline = self.redis_client.pipeline()
                for key in hit_keys:
                    pipeline.expire(key, self.ttl_seconds)
                await pipeline.execute()
                
            return results
            
        except Exception as e:
            self.stats["errors"] += 1
            print(f"Batch get error: {e}")
            return {text: None for text in texts}
    
    async def set_batch(
        self,
        embeddings: Dict[str, np.ndarray],
        model: str = "default"
    ):
        """Set multiple embeddings efficiently"""
        try:
            pipeline = self.redis_client.pipeline()
            
            for text, embedding in embeddings.items():
                key = self._generate_key(text, model)
                data = self._serialize_embedding(embedding)
                pipeline.setex(key, self.ttl_seconds, data)
            
            await pipeline.execute()
            
            # Update index
            for text in embeddings.keys():
                await self._update_index(text, model)
                
        except Exception as e:
            self.stats["errors"] += 1
            print(f"Batch set error: {e}")
    
    async def _update_index(self, text: str, model: str):
        """Update cache index for tracking"""
        index_key = f"{self.cache_prefix}:index:{model}"
        timestamp = datetime.now().timestamp()
        
        # Add to sorted set with timestamp as score
        await self.redis_client.zadd(
            index_key,
            {text: timestamp}
        )
        
        # Trim old entries (keep last 10000)
        await self.redis_client.zremrangebyrank(index_key, 0, -10001)
    
    async def warm_cache(
        self,
        texts: List[str],
        embedding_generator,
        model: str = "default",
        batch_size: int = 32
    ):
        """Warm cache with frequently used embeddings"""
        
        # Check which embeddings are missing
        cached = await self.get_batch(texts, model)
        missing_texts = [
            text for text, emb in cached.items() if emb is None
        ]
        
        if not missing_texts:
            print(f"Cache already warm, all {len(texts)} embeddings present")
            return
        
        print(f"Warming cache with {len(missing_texts)} embeddings...")
        
        # Generate missing embeddings in batches
        for i in range(0, len(missing_texts), batch_size):
            batch = missing_texts[i:i + batch_size]
            
            # Generate embeddings
            embeddings = await embedding_generator(batch)
            
            # Cache them
            embedding_dict = dict(zip(batch, embeddings))
            await self.set_batch(embedding_dict, model)
            
            print(f"Cached batch {i//batch_size + 1}/{len(missing_texts)//batch_size + 1}")
    
    async def get_stats(self) -> Dict[str, Any]:
        """Get cache statistics"""
        
        # Get Redis info
        info = await self.redis_client.info("memory")
        
        # Get cache size
        keys = await self.redis_client.keys(f"{self.cache_prefix}:*")
        
        return {
            "hits": self.stats["hits"],
            "misses": self.stats["misses"],
            "errors": self.stats["errors"],
            "hit_rate": (
                self.stats["hits"] / 
                (self.stats["hits"] + self.stats["misses"])
                if (self.stats["hits"] + self.stats["misses"]) > 0
                else 0
            ),
            "cache_entries": len([k for k in keys if not k.decode().endswith(":meta")]),
            "memory_used_mb": info.get("used_memory", 0) / 1024 / 1024,
            "compression_enabled": self.compression
        }
    
    async def clear_cache(self, model: Optional[str] = None):
        """Clear cache entries"""
        if model:
            pattern = f"{self.cache_prefix}:{model}:*"
        else:
            pattern = f"{self.cache_prefix}:*"
            
        keys = await self.redis_client.keys(pattern)
        
        if keys:
            await self.redis_client.delete(*keys)
            
        print(f"Cleared {len(keys)} cache entries")
    
    async def optimize_cache(self):
        """Optimize cache by removing least recently used entries"""
        
        # Get all keys with their TTL
        keys = await self.redis_client.keys(f"{self.cache_prefix}:*:*")
        
        key_ttls = []
        for key in keys:
            if not key.decode().endswith(":meta"):
                ttl = await self.redis_client.ttl(key)
                if ttl > 0:
                    key_ttls.append((key, ttl))
        
        # Sort by TTL (least recently accessed first)
        key_ttls.sort(key=lambda x: x[1])
        
        # Remove bottom 20% if cache is large
        if len(key_ttls) > 1000:
            to_remove = key_ttls[:len(key_ttls) // 5]
            
            pipeline = self.redis_client.pipeline()
            for key, _ in to_remove:
                pipeline.delete(key)
                # Also delete metadata
                pipeline.delete(f"{key.decode()}:meta")
            
            await pipeline.execute()
            
            print(f"Optimized cache, removed {len(to_remove)} old entries")
```

```python
# Usage example for the complete embedding cache system
async def example_usage():
    # Initialize services
    async with EmbeddingService() as embedding_service, \
               EmbeddingCacheService() as cache_service:
        
        # Example texts
        texts = [
            "Reserve una experiencia de buceo en cenotes",
            "Tour gastronómico por Mérida",
            "Clase de cocina yucateca tradicional",
            "Excursión a las ruinas mayas de Uxmal"
        ]
        
        # Check cache first
        cached_embeddings = await cache_service.get_batch(texts)
        
        # Generate missing embeddings
        missing_texts = [
            text for text, emb in cached_embeddings.items() 
            if emb is None
        ]
        
        if missing_texts:
            new_embeddings = await embedding_service.generate_embeddings_batch(
                missing_texts
            )
            
            # Cache the new embeddings
            embedding_dict = dict(zip(missing_texts, new_embeddings))
            await cache_service.set_batch(embedding_dict)
        
        # Combine cached and new embeddings
        all_embeddings = {}
        for text in texts:
            if cached_embeddings[text] is not None:
                all_embeddings[text] = cached_embeddings[text]
            else:
                idx = missing_texts.index(text)
                all_embeddings[text] = new_embeddings[idx]
        
        # Get cache statistics
        stats = await cache_service.get_stats()
        print(f"Cache stats: {stats}")
        
        return all_embeddings
```

---

## 4. MONITOREO ENTERPRISE

### 4.1 Alertas Inteligentes con Alertmanager

```yaml
# monitoring/alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_from: 'alerts@tu-dominio.com'
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_auth_username: 'alerts@tu-dominio.com'
  smtp_auth_password: '${SMTP_PASSWORD}'
  slack_api_url: '${SLACK_WEBHOOK_URL}'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

# Templates for notifications
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routing tree
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'default-receiver'
  
  routes:
    # Critical alerts - immediate notification
    - match:
        severity: critical
      receiver: 'critical-receiver'
      group_wait: 0s
      repeat_interval: 5m
      continue: true
    
    # Database alerts
    - match:
        service: database
      receiver: 'database-team'
      group_wait: 30s
      
    # AI/LLM alerts
    - match_re:
        service: (llama|embeddings)
      receiver: 'ai-team'
      group_wait: 1m
      
    # Business hours only
    - match:
        severity: warning
      receiver: 'business-hours'
      active_time_intervals:
        - business-hours
      
    # E-commerce specific
    - match:
        component: payment
      receiver: 'payment-critical'
      group_wait: 0s
      repeat_interval: 1m

# Receivers configuration
receivers:
  - name: 'default-receiver'
    slack_configs:
      - channel: '#alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
  
  - name: 'critical-receiver'
    pagerduty_configs:
      - service_key: '${PAGERDUTY_SERVICE_KEY}'
        description: '{{ .GroupLabels.alertname }}'
        details:
          severity: '{{ .CommonLabels.severity }}'
          service: '{{ .CommonLabels.service }}'
    slack_configs:
      - channel: '#critical-alerts'
        title: '🚨 CRITICAL: {{ .GroupLabels.alertname }}'
        color: 'danger'
        text: |
          *Alert:* {{ .GroupLabels.alertname }}
          *Service:* {{ .CommonLabels.service }}
          *Severity:* {{ .CommonLabels.severity }}
          *Description:* {{ range .Alerts }}{{ .Annotations.description }}{{ end }}
          *Graph:* <{{ .GeneratorURL }}|View in Prometheus>
    email_configs:
      - to: 'oncall@tu-dominio.com'
        headers:
          Subject: 'CRITICAL ALERT: {{ .GroupLabels.alertname }}'
        html: |
          <h2>Critical Alert Triggered</h2>
          <p><strong>Alert:</strong> {{ .GroupLabels.alertname }}</p>
          <p><strong>Service:</strong> {{ .CommonLabels.service }}</p>
          <p><strong>Description:</strong> {{ range .Alerts }}{{ .Annotations.description }}{{ end }}</p>
          <p><a href="{{ .GeneratorURL }}">View in Prometheus</a></p>
  
  - name: 'database-team'
    slack_configs:
      - channel: '#database-alerts'
        title: 'Database Alert: {{ .GroupLabels.alertname }}'
        fields:
          - title: 'Instance'
            value: '{{ .CommonLabels.instance }}'
            short: true
          - title: 'Database'
            value: '{{ .CommonLabels.database }}'
            short: true
    webhook_configs:
      - url: 'http://n8n:5678/webhook/database-alert'
        send_resolved: true
  
  - name: 'ai-team'
    slack_configs:
      - channel: '#ai-alerts'
        title: 'AI Service Alert: {{ .GroupLabels.alertname }}'
        text: |
          *Model:* {{ .CommonLabels.model }}
          *Performance:* {{ .CommonLabels.performance_metric }}
          *Description:* {{ .Annotations.description }}
  
  - name: 'payment-critical'
    pagerduty_configs:
      - service_key: '${PAGERDUTY_PAYMENT_KEY}'
        severity: 'critical'
    slack_configs:
      - channel: '#payment-critical'
        title: '💳 PAYMENT ALERT: {{ .GroupLabels.alertname }}'
        color: 'danger'
    sms_configs:
      - to: '+521234567890,+521234567891'
        text: 'PAYMENT ALERT: {{ .GroupLabels.alertname }} - {{ .Annotations.summary }}'
  
  - name: 'business-hours'
    email_configs:
      - to: 'team@tu-dominio.com'
        send_resolved: true

# Time intervals
time_intervals:
  - name: business-hours
    time_intervals:
      - weekdays: ['monday:friday']
        times:
          - start_time: '09:00'
            end_time: '18:00'
        location: 'America/Mexico_City'

# Inhibition rules
inhibit_rules:
  # Don't alert for pods if node is down
  - source_match:
      severity: 'critical'
      alertname: 'NodeDown'
    target_match:
      severity: 'warning'
    equal: ['instance']
  
  # Don't send warnings if critical alert exists
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'service']
```

```yaml
# monitoring/prometheus-rules.yml
groups:
  - name: microservices
    interval: 30s
    rules:
      # Service availability
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
          service: "{{ $labels.job }}"
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute."
          
      # High error rate
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
            /
            sum(rate(http_requests_total[5m])) by (service)
          ) > 0.05
        for: 5m
        labels:
          severity: warning
          service: "{{ $labels.service }}"
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"
      
      # Response time
      - alert: HighResponseTime
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le)
          ) > 1
        for: 10m
        labels:
          severity: warning
          service: "{{ $labels.service }}"
        annotations:
          summary: "High response time on {{ $labels.service }}"
          description: "95th percentile response time is {{ $value }}s"
  
  - name: database
    interval: 30s
    rules:
      # Connection pool exhaustion
      - alert: DatabaseConnectionPoolExhausted
        expr: |
          (pg_stat_database_numbackends / pg_settings_max_connections) > 0.8
        for: 5m
        labels:
          severity: warning
          service: database
        annotations:
          summary: "Database connection pool near exhaustion"
          description: "{{ $value | humanizePercentage }} of connections are in use"
      
      # Replication lag
      - alert: DatabaseReplicationLag
        expr: pg_replication_lag_seconds > 10
        for: 5m
        labels:
          severity: critical
          service: database
        annotations:
          summary: "Database replication lag detected"
          description: "Replication lag is {{ $value }}s"
      
      # Disk space
      - alert: DatabaseDiskSpaceLow
        expr: |
          (pg_database_size_bytes / node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"}) > 0.8
        for: 10m
        labels:
          severity: warning
          service: database
        annotations:
          summary: "Database disk space running low"
          description: "{{ $value | humanizePercentage }} of disk space used"
  
  - name: ai-services
    interval: 30s
    rules:
      # LLM response time
      - alert: LLMHighResponseTime
        expr: |
          histogram_quantile(0.95,
            sum(rate(llm_request_duration_seconds_bucket[5m])) by (model, le)
          ) > 30
        for: 5m
        labels:
          severity: warning
          service: llama
          model: "{{ $labels.model }}"
        annotations:
          summary: "LLM response time too high"
          description: "{{ $labels.model }} p95 response time is {{ $value }}s"
      
      # Embedding service errors
      - alert: EmbeddingServiceErrors
        expr: |
          sum(rate(embedding_errors_total[5m])) > 0.1
        for: 5m
        labels:
          severity: warning
          service: embeddings
        annotations:
          summary: "Embedding service experiencing errors"
          description: "Error rate: {{ $value }} errors/sec"
      
      # Model memory usage
      - alert: AIModelHighMemoryUsage
        expr: |
          container_memory_usage_bytes{name=~"llama|embeddings"} 
          / container_spec_memory_limit_bytes > 0.9
        for: 5m
        labels:
          severity: critical
          service: "{{ $labels.name }}"
        annotations:
          summary: "AI model high memory usage"
          description: "{{ $labels.name }} using {{ $value | humanizePercentage }} of memory limit"
  
  - name: business-metrics
    interval: 60s
    rules:
      # Booking conversion rate drop
      - alert: BookingConversionRateDropped
        expr: |
          (
            sum(rate(bookings_completed_total[1h]))
            /
            sum(rate(booking_attempts_total[1h]))
          ) < 0.1
        for: 15m
        labels:
          severity: critical
          component: business
        annotations:
          summary: "Booking conversion rate critically low"
          description: "Conversion rate is {{ $value | humanizePercentage }}"
      
      # Payment failures
      - alert: HighPaymentFailureRate
        expr: |
          sum(rate(payment_failures_total[5m])) > 5
        for: 5m
        labels:
          severity: critical
          component: payment
        annotations:
          summary: "High payment failure rate"
          description: "{{ $value }} payment failures per second"
      
      # Revenue tracking
      - alert: RevenueDropDetected
        expr: |
          (
            sum(rate(revenue_total[1h]))
            <
            sum(rate(revenue_total[1h] offset 1d)) * 0.5
          )
        for: 30m
        labels:
          severity: warning
          component: business
        annotations:
          summary: "Significant revenue drop detected"
          description: "Revenue is down more than 50% compared to same time yesterday"
```

### 4.2 Grafana Dashboards Pre-construidos

```json
// dashboards/microservices-overview.json
{
  "dashboard": {
    "title": "Microservices Overview - Production",
    "uid": "microservices-overview",
    "timezone": "browser",
    "panels": [
      {
        "title": "Service Health Status",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 0, "y": 0 },
        "targets": [
          {
            "expr": "sum(up) / count(up)",
            "legendFormat": "Health Score"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "mappings": [
              {
                "options": {
                  "from": 0.99,
                  "to": 1,
                  "result": { "text": "Healthy", "color": "green" }
                }
              },
              {
                "options": {
                  "from": 0.8,
                  "to": 0.99,
                  "result": { "text": "Degraded", "color": "yellow" }
                }
              },
              {
                "options": {
                  "from": 0,
                  "to": 0.8,
                  "result": { "text": "Critical", "color": "red" }
                }
              }
            ],
            "unit": "percentunit",
            "thresholds": {
              "mode": "absolute",
              "steps": [
                { "color": "red", "value": null },
                { "color": "yellow", "value": 0.8 },
                { "color": "green", "value": 0.99 }
              ]
            }
          }
        }
      },
      {
        "title": "Request Rate by Service",
        "type": "graph",
        "gridPos": { "h": 8, "w": 12, "x": 6, "y": 0 },
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{ service }}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "custom": {
              "drawStyle": "line",
              "lineInterpolation": "smooth",
              "lineWidth": 2,
              "fillOpacity": 0.1,
              "gradientMode": "opacity"
            }
          }
        }
      },
      {
        "title": "Error Rate",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 18, "y": 0 },
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (service)",
            "legendFormat": "{{ service }} - 5xx"
          },
          {
            "expr": "sum(rate(http_requests_total{status=~\"4..\"}[5m])) by (service)",
            "legendFormat": "{{ service }} - 4xx"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "custom": {
              "drawStyle": "bars",
              "lineWidth": 0,
              "fillOpacity": 0.8
            },
            "color": {
              "mode": "palette-classic"
            }
          }
        }
      },
      {
        "title": "Response Time Heatmap",
        "type": "heatmap",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 8 },
        "targets": [
          {
            "expr": "sum(rate(http_request_duration_seconds_bucket[5m])) by (le)",
            "format": "heatmap",
            "legendFormat": "{{ le }}"
          }
        ],
        "options": {
          "calculate": true,
          "calculation": {
            "xBuckets": {
              "mode": "count",
              "value": "30"
            },
            "yBuckets": {
              "mode": "count"
            }
          },
          "color": {
            "mode": "scheme",
            "scheme": "Spectral",
            "steps": 128
          }
        }
      },
      {
        "title": "Database Performance",
        "type": "row",
        "gridPos": { "h": 1, "w": 24, "x": 0, "y": 16 }
      },
      {
        "title": "Database Connections",
        "type": "graph",
        "gridPos": { "h": 8, "w": 8, "x": 0, "y": 17 },
        "targets": [
          {
            "expr": "pg_stat_database_numbackends{datname=\"main_db\"}",
            "legendFormat": "Active Connections"
          },
          {
            "expr": "pg_settings_max_connections",
            "legendFormat": "Max Connections"
          }
        ]
      },
      {
        "title": "Query Performance",
        "type": "table",
        "gridPos": { "h": 8, "w": 16, "x": 8, "y": 17 },
        "targets": [
          {
            "expr": "topk(10, pg_stat_statements_mean_exec_time_ms)",
            "format": "table",
            "instant": true
          }
        ],
        "transformations": [
          {
            "id": "organize",
            "options": {
              "excludeByName": {},
              "indexByName": {},
              "renameByName": {
                "query": "Query",
                "mean_exec_time_ms": "Avg Time (ms)",
                "calls": "Calls"
              }
            }
          }
        ]
      },
      {
        "title": "AI Services Performance",
        "type": "row",
        "gridPos": { "h": 1, "w": 24, "x": 0, "y": 25 }
      },
      {
        "title": "LLM Response Times",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 26 },
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(llm_request_duration_seconds_bucket[5m])) by (le, model))",
            "legendFormat": "{{ model }} - p95"
          },
          {
            "expr": "histogram_quantile(0.50, sum(rate(llm_request_duration_seconds_bucket[5m])) by (le, model))",
            "legendFormat": "{{ model }} - p50"
          }
        ]
      },
      {
        "title": "Embedding Cache Performance",
        "type": "stat",
        "gridPos": { "h": 8, "w": 6, "x": 12, "y": 26 },
        "targets": [
          {
            "expr": "embedding_cache_hits / (embedding_cache_hits + embedding_cache_misses)",
            "legendFormat": "Cache Hit Rate"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percentunit",
            "custom": {
              "text": {
                "valueSize": 48
              }
            }
          }
        }
      },
      {
        "title": "Business Metrics",
        "type": "row",
        "gridPos": { "h": 1, "w": 24, "x": 0, "y": 34 }
      },
      {
        "title": "Booking Funnel",
        "type": "bargauge",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 35 },
        "targets": [
          {
            "expr": "sum(booking_funnel_total{step=\"view\"})",
            "legendFormat": "Views"
          },
          {
            "expr": "sum(booking_funnel_total{step=\"start\"})",
            "legendFormat": "Started"
          },
          {
            "expr": "sum(booking_funnel_total{step=\"details\"})",
            "legendFormat": "Details Added"
          },
          {
            "expr": "sum(booking_funnel_total{step=\"payment\"})",
            "legendFormat": "Payment"
          },
          {
            "expr": "sum(booking_funnel_total{step=\"completed\"})",
            "legendFormat": "Completed"
          }
        ],
        "options": {
          "displayMode": "gradient",
          "orientation": "horizontal"
        }
      },
      {
        "title": "Revenue Metrics",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 12, "y": 35 },
        "targets": [
          {
            "expr": "sum(increase(revenue_total[1h]))",
            "legendFormat": "Revenue (Last Hour)"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "currencyMXN",
            "custom": {
              "text": {
                "valueSize": 32
              }
            }
          }
        }
      },
      {
        "title": "Active Users",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 18, "y": 35 },
        "targets": [
          {
            "expr": "count(count by (user_id) (rate(user_activity_total[5m]) > 0))",
            "legendFormat": "Active Users"
          }
        ]
      }
    ],
    "time": {
      "from": "now-3h",
      "to": "now"
    },
    "refresh": "10s"
  }
}
```

### 4.3 Distributed Tracing con Jaeger

```yaml
# docker-compose-tracing.yml
services:
  # Jaeger All-in-One
  jaeger:
    image: jaegertracing/all-in-one:latest
    environment:
      - COLLECTOR_OTLP_ENABLED=true
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
    ports:
      - "5775:5775/udp"  # Zipkin/Thrift
      - "6831:6831/udp"  # Compact Thrift
      - "6832:6832/udp"  # Binary Thrift
      - "5778:5778"      # Config
      - "16686:16686"    # Web UI
      - "14268:14268"    # HTTP collector
      - "14250:14250"    # gRPC collector
      - "9411:9411"      # Zipkin
      - "4317:4317"      # OTLP gRPC
      - "4318:4318"      # OTLP HTTP
    networks:
      - microservices-net
    depends_on:
      - elasticsearch

  # OpenTelemetry Collector
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./monitoring/otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "1888:1888"   # pprof extension
      - "8888:8888"   # Prometheus metrics exposed by the collector
      - "8889:8889"   # Prometheus exporter metrics
      - "13133:13133" # health_check extension
      - "4319:4317"   # OTLP gRPC receiver
      - "4320:4318"   # OTLP HTTP receiver
      - "55679:55679" # zpages extension
    networks:
      - microservices-net

  # Elasticsearch for trace storage
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - microservices-net
```

```yaml
# monitoring/otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
        cors:
          allowed_origins:
            - http://*
            - https://*
          allowed_headers:
            - "*"

  prometheus:
    config:
      scrape_configs:
        - job_name: 'otel-collector'
          scrape_interval: 10s
          static_configs:
            - targets: ['0.0.0.0:8888']

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
    send_batch_max_size: 2048

  memory_limiter:
    check_interval: 1s
    limit_percentage: 75
    spike_limit_percentage: 25

  resource:
    attributes:
      - key: environment
        value: production
        action: upsert
      - key: service.namespace
        value: microservices
        action: upsert

  attributes:
    actions:
      - key: http.url
        action: delete
      - key: http.user_agent
        action: delete

  tail_sampling:
    decision_wait: 10s
    num_traces: 100
    expected_new_traces_per_sec: 10
    policies:
      [
        {
          name: errors-policy,
          type: status_code,
          status_code: {status_codes: [ERROR]}
        },
        {
          name: slow-traces-policy,
          type: latency,
          latency: {threshold_ms: 1000}
        },
        {
          name: probabilistic-policy,
          type: probabilistic,
          probabilistic: {sampling_percentage: 10}
        }
      ]

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: traces
    const_labels:
      exporter: otlp

  logging:
    loglevel: info

extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  pprof:
    endpoint: 0.0.0.0:1888
  zpages:
    endpoint: 0.0.0.0:55679

service:
  extensions: [health_check, pprof, zpages]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource, attributes, tail_sampling]
      exporters: [jaeger, logging]
    
    metrics:
      receivers: [prometheus, otlp]
      processors: [memory_limiter, batch, resource]
      exporters: [prometheus]
```

```typescript
// lib/tracing/tracer.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-grpc';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { 
  ConsoleSpanExporter, 
  BatchSpanProcessor,
  SimpleSpanProcessor 
} from '@opentelemetry/sdk-trace-base';
import { trace, context, SpanStatusCode, SpanKind } from '@opentelemetry/api';

// Initialize tracing
export function initializeTracing(serviceName: string) {
  const resource = new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: serviceName,
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.SERVICE_VERSION || '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'development',
  });

  const traceExporter = new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://otel-collector:4317',
  });

  const metricExporter = new OTLPMetricExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://otel-collector:4317',
  });

  const sdk = new NodeSDK({
    resource,
    traceExporter,
    metricReader: new PeriodicExportingMetricReader({
      exporter: metricExporter,
      exportIntervalMillis: 10000,
    }),
    instrumentations: [
      getNodeAutoInstrumentations({
        '@opentelemetry/instrumentation-fs': {
          enabled: false,
        },
        '@opentelemetry/instrumentation-http': {
          requestHook: (span, request) => {
            span.setAttribute('http.request.body', JSON.stringify(request.body));
          },
          responseHook: (span, response) => {
            span.setAttribute('http.response.size', response.headers['content-length'] || 0);
          },
        },
      }),
    ],
  });

  sdk.start();

  // Graceful shutdown
  process.on('SIGTERM', () => {
    sdk.shutdown()
      .then(() => console.log('Tracing terminated'))
      .catch((error) => console.log('Error terminating tracing', error))
      .finally(() => process.exit(0));
  });

  return sdk;
}

// Custom tracer wrapper for business logic
export class CustomTracer {
  private tracer;

  constructor(name: string) {
    this.tracer = trace.getTracer(name);
  }

  async traceAsync<T>(
    spanName: string,
    fn: () => Promise<T>,
    attributes?: Record<string, any>
  ): Promise<T> {
    const span = this.tracer.startSpan(spanName, {
      kind: SpanKind.INTERNAL,
      attributes,
    });

    try {
      const result = await fn();
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error.message,
      });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  }

  traceSync<T>(
    spanName: string,
    fn: () => T,
    attributes?: Record<string, any>
  ): T {
    const span = this.tracer.startSpan(spanName, {
      kind: SpanKind.INTERNAL,
      attributes,
    });

    try {
      const result = fn();
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error.message,
      });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  }

  // Trace database operations
  async traceDatabase<T>(
    operation: string,
    query: string,
    fn: () => Promise<T>
  ): Promise<T> {
    const span = this.tracer.startSpan(`db.${operation}`, {
      kind: SpanKind.CLIENT,
      attributes: {
        'db.system': 'postgresql',
        'db.operation': operation,
        'db.statement': query.substring(0, 1000), // Truncate long queries
      },
    });

    try {
      const start = Date.now();
      const result = await fn();
      const duration = Date.now() - start;

      span.setAttributes({
        'db.duration': duration,
        'db.rows_affected': (result as any)?.rowCount || 0,
      });

      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error.message,
      });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  }

  // Trace external API calls
  async traceExternalCall<T>(
    service: string,
    method: string,
    url: string,
    fn: () => Promise<T>
  ): Promise<T> {
    const span = this.tracer.startSpan(`external.${service}`, {
      kind: SpanKind.CLIENT,
      attributes: {
        'http.method': method,
        'http.url': url,
        'peer.service': service,
      },
    });

    try {
      const start = Date.now();
      const result = await fn();
      const duration = Date.now() - start;

      span.setAttributes({
        'http.duration': duration,
        'http.status_code': (result as any)?.status || 200,
      });

      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error.message,
      });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  }
}

// Usage example in Next.js API route
export function withTracing(handler: any) {
  const tracer = new CustomTracer('nextjs-api');

  return async (req: any, res: any) => {
    return tracer.traceAsync(
      `${req.method} ${req.url}`,
      async () => {
        return handler(req, res);
      },
      {
        'http.method': req.method,
        'http.route': req.url,
        'http.target': req.url,
        'user.id': req.session?.userId,
      }
    );
  };
}
```

### 4.4 Business Metrics Monitoring

```typescript
// lib/metrics/business-metrics.ts
import { Counter, Gauge, Histogram, Registry } from 'prom-client';

class BusinessMetrics {
  private registry: Registry;
  
  // Booking metrics
  private bookingAttempts: Counter;
  private bookingCompletions: Counter;
  private bookingRevenue: Counter;
  private bookingDuration: Histogram;
  private activeBookings: Gauge;
  
  // User engagement metrics
  private activeUsers: Gauge;
  private userSessions: Counter;
  private pageViews: Counter;
  private searchQueries: Counter;
  
  // Experience metrics
  private experienceViews: Counter;
  private experienceBookings: Counter;
  private experienceRatings: Histogram;
  
  // Business funnel metrics
  private funnelSteps: Counter;
  private conversionRate: Gauge;
  
  // Performance metrics
  private checkoutDuration: Histogram;
  private searchLatency: Histogram;

  constructor() {
    this.registry = new Registry();
    this.initializeMetrics();
  }

  private initializeMetrics() {
    // Booking metrics
    this.bookingAttempts = new Counter({
      name: 'booking_attempts_total',
      help: 'Total number of booking attempts',
      labelNames: ['experience_type', 'source'],
      registers: [this.registry]
    });

    this.bookingCompletions = new Counter({
      name: 'booking_completions_total',
      help: 'Total number of completed bookings',
      labelNames: ['experience_type', 'payment_method'],
      registers: [this.registry]
    });

    this.bookingRevenue = new Counter({
      name: 'booking_revenue_total',
      help: 'Total revenue from bookings',
      labelNames: ['experience_type', 'currency'],
      registers: [this.registry]
    });

    this.bookingDuration = new Histogram({
      name: 'booking_duration_seconds',
      help: 'Time taken to complete a booking',
      labelNames: ['experience_type'],
      buckets: [10, 30, 60, 120, 300, 600],
      registers: [this.registry]
    });

    this.activeBookings = new Gauge({
      name: 'active_bookings_current',
      help: 'Current number of active bookings',
      labelNames: ['status'],
      registers: [this.registry]
    });

    // User engagement
    this.activeUsers = new Gauge({
      name: 'active_users_current',
      help: 'Current number of active users',
      labelNames: ['user_type'],
      registers: [this.registry]
    });

    this.userSessions = new Counter({
      name: 'user_sessions_total',
      help: 'Total number of user sessions',
      labelNames: ['device_type', 'country'],
      registers: [this.registry]
    });

    this.pageViews = new Counter({
      name: 'page_views_total',
      help: 'Total page views',
      labelNames: ['page_type', 'device_type'],
      registers: [this.registry]
    });

    this.searchQueries = new Counter({
      name: 'search_queries_total',
      help: 'Total search queries',
      labelNames: ['search_type', 'results_found'],
      registers: [this.registry]
    });

    // Experience metrics
    this.experienceViews = new Counter({
      name: 'experience_views_total',
      help: 'Total experience views',
      labelNames: ['experience_id', 'category'],
      registers: [this.registry]
    });

    this.experienceBookings = new Counter({
      name: 'experience_bookings_total',
      help: 'Total bookings per experience',
      labelNames: ['experience_id', 'category'],
      registers: [this.registry]
    });

    this.experienceRatings = new Histogram({
      name: 'experience_ratings',
      help: 'Distribution of experience ratings',
      labelNames: ['experience_id', 'category'],
      buckets: [1, 2, 3, 4, 5],
      registers: [this.registry]
    });

    // Funnel metrics
    this.funnelSteps = new Counter({
      name: 'booking_funnel_total',
      help: 'Users at each step of booking funnel',
      labelNames: ['step', 'experience_type'],
      registers: [this.registry]
    });

    this.conversionRate = new Gauge({
      name: 'conversion_rate_current',
      help: 'Current conversion rate',
      labelNames: ['period', 'experience_type'],
      registers: [this.registry]
    });

    // Performance metrics
    this.checkoutDuration = new Histogram({
      name: 'checkout_duration_seconds',
      help: 'Time taken to complete checkout',
      labelNames: ['payment_method'],
      buckets: [1, 5, 10, 30, 60, 120],
      registers: [this.registry]
    });

    this.searchLatency = new Histogram({
      name: 'search_latency_seconds',
      help: 'Search query latency',
      labelNames: ['search_type'],
      buckets: [0.1, 0.25, 0.5, 1, 2.5, 5],
      registers: [this.registry]
    });
  }

  // Booking tracking methods
  trackBookingAttempt(experienceType: string, source: string) {
    this.bookingAttempts.inc({ experience_type: experienceType, source });
    this.funnelSteps.inc({ step: 'start', experience_type: experienceType });
  }

  trackBookingCompletion(
    experienceType: string, 
    paymentMethod: string, 
    revenue: number,
    duration: number
  ) {
    this.bookingCompletions.inc({ 
      experience_type: experienceType, 
      payment_method: paymentMethod 
    });
    
    this.bookingRevenue.inc(
      { experience_type: experienceType, currency: 'MXN' }, 
      revenue
    );
    
    this.bookingDuration.observe(
      { experience_type: experienceType }, 
      duration
    );
    
    this.funnelSteps.inc({ 
      step: 'completed', 
      experience_type: experienceType 
    });
    
    this.updateConversionRate();
  }

  trackFunnelStep(step: string, experienceType: string) {
    this.funnelSteps.inc({ step, experience_type: experienceType });
  }

  // User engagement tracking
  trackUserSession(deviceType: string, country: string) {
    this.userSessions.inc({ device_type: deviceType, country });
  }

  trackPageView(pageType: string, deviceType: string) {
    this.pageViews.inc({ page_type: pageType, device_type: deviceType });
  }

  trackSearch(searchType: string, resultsFound: boolean, latency: number) {
    this.searchQueries.inc({ 
      search_type: searchType, 
      results_found: resultsFound ? 'yes' : 'no' 
    });
    
    this.searchLatency.observe({ search_type: searchType }, latency);
  }

  // Experience tracking
  trackExperienceView(experienceId: string, category: string) {
    this.experienceViews.inc({ experience_id: experienceId, category });
  }

  trackExperienceBooking(experienceId: string, category: string) {
    this.experienceBookings.inc({ experience_id: experienceId, category });
  }

  trackExperienceRating(experienceId: string, category: string, rating: number) {
    this.experienceRatings.observe(
      { experience_id: experienceId, category }, 
      rating
    );
  }

  // Performance tracking
  trackCheckout(paymentMethod: string, duration: number) {
    this.checkoutDuration.observe({ payment_method: paymentMethod }, duration);
  }

  // Update methods
  updateActiveUsers(count: number, userType: string = 'all') {
    this.activeUsers.set({ user_type: userType }, count);
  }

  updateActiveBookings(counts: Record<string, number>) {
    Object.entries(counts).forEach(([status, count]) => {
      this.activeBookings.set({ status }, count);
    });
  }

  private updateConversionRate() {
    // This would be called periodically to calculate conversion rates
    // Implementation depends on your specific calculation needs
  }

  // Get metrics for Prometheus
  getMetrics() {
    return this.registry.metrics();
  }

  // Custom business reports
  async generateBusinessReport(period: string): Promise<any> {
    // Generate comprehensive business report
    const metrics = await this.registry.getMetricsAsJSON();
    
    const report = {
      period,
      timestamp: new Date().toISOString(),
      summary: {
        totalBookings: this.getMetricValue(metrics, 'booking_completions_total'),
        totalRevenue: this.getMetricValue(metrics, 'booking_revenue_total'),
        activeUsers: this.getMetricValue(metrics, 'active_users_current'),
        conversionRate: this.calculateConversionRate(metrics)
      },
      details: {
        bookingsByType: this.groupByLabel(metrics, 'booking_completions_total', 'experience_type'),
        revenueByType: this.groupByLabel(metrics, 'booking_revenue_total', 'experience_type'),
        funnelAnalysis: this.analyzeFunnel(metrics),
        performanceMetrics: this.getPerformanceMetrics(metrics)
      }
    };
    
    return report;
  }

  private getMetricValue(metrics: any[], name: string): number {
    const metric = metrics.find(m => m.name === name);
    if (!metric || !metric.values.length) return 0;
    
    return metric.values.reduce((sum: number, v: any) => sum + (v.value || 0), 0);
  }

  private groupByLabel(metrics: any[], name: string, label: string): Record<string, number> {
    const metric = metrics.find(m => m.name === name);
    if (!metric) return {};
    
    const grouped: Record<string, number> = {};
    metric.values.forEach((v: any) => {
      const key = v.labels[label] || 'unknown';
      grouped[key] = (grouped[key] || 0) + v.value;
    });
    
    return grouped;
  }

  private calculateConversionRate(metrics: any[]): number {
    const attempts = this.getMetricValue(metrics, 'booking_attempts_total');
    const completions = this.getMetricValue(metrics, 'booking_completions_total');
    
    return attempts > 0 ? (completions / attempts) * 100 : 0;
  }

  private analyzeFunnel(metrics: any[]): any {
    const funnelMetric = metrics.find(m => m.name === 'booking_funnel_total');
    if (!funnelMetric) return {};
    
    const funnel: Record<string, number> = {};
    funnelMetric.values.forEach((v: any) => {
      const step = v.labels.step;
      funnel[step] = (funnel[step] || 0) + v.value;
    });
    
    return funnel;
  }

  private getPerformanceMetrics(metrics: any[]): any {
    return {
      avgCheckoutTime: this.getHistogramAverage(metrics, 'checkout_duration_seconds'),
      avgSearchLatency: this.getHistogramAverage(metrics, 'search_latency_seconds'),
      avgBookingDuration: this.getHistogramAverage(metrics, 'booking_duration_seconds')
    };
  }

  private getHistogramAverage(metrics: any[], name: string): number {
    const metric = metrics.find(m => m.name === name);
    if (!metric) return 0;
    
    // Calculate weighted average from histogram buckets
    // This is a simplified calculation
    let sum = 0;
    let count = 0;
    
    metric.values.forEach((v: any) => {
      if (v.metricName === `${name}_sum`) {
        sum += v.value;
      } else if (v.metricName === `${name}_count`) {
        count += v.value;
      }
    });
    
    return count > 0 ? sum / count : 0;
  }
}

// Export singleton instance
export const businessMetrics = new BusinessMetrics();

// Next.js API endpoint for metrics
export async function metricsHandler(req: any, res: any) {
  res.setHeader('Content-Type', businessMetrics.getMetrics());
  res.status(200).send(await businessMetrics.getMetrics());
}
```

---

## 5. CALENDARIO DE RESERVAS "CHINGÓN"

### 5.1 FullCalendar con React y Máxima Flexibilidad

```tsx
// components/calendar/BookingCalendar.tsx
import React, { useState, useEffect, useRef, useCallback } from 'react';
import FullCalendar from '@fullcalendar/react';
import dayGridPlugin from '@fullcalendar/daygrid';
import timeGridPlugin from '@fullcalendar/timegrid';
import interactionPlugin from '@fullcalendar/interaction';
import listPlugin from '@fullcalendar/list';
import multiMonthPlugin from '@fullcalendar/multimonth';
import resourceTimelinePlugin from '@fullcalendar/resource-timeline';
import adaptivePlugin from '@fullcalendar/adaptive';
import { motion, AnimatePresence, useAnimation } from 'framer-motion';
import { Canvas } from '@react-three/fiber';
import { CalendarEvent, Resource, BookingSlot } from '@/types/calendar';
import { useWebSocket } from '@/hooks/useWebSocket';
import { useCalendarSync } from '@/hooks/useCalendarSync';

interface BookingCalendarProps {
  experienceId?: string;
  onSlotSelect: (slot: BookingSlot) => void;
  view?: 'month' | 'week' | 'day' | 'timeline' | '3d';
  showResources?: boolean;
}

export const BookingCalendar: React.FC<BookingCalendarProps> = ({
  experienceId,
  onSlotSelect,
  view = 'month',
  showResources = false
}) => {
  const calendarRef = useRef<FullCalendar>(null);
  const [events, setEvents] = useState<CalendarEvent[]>([]);
  const [resources, setResources] = useState<Resource[]>([]);
  const [selectedDate, setSelectedDate] = useState<Date | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [show3D, setShow3D] = useState(false);
  const controls = useAnimation();

  // WebSocket for real-time updates
  const { sendMessage, lastMessage } = useWebSocket('/calendar');

  // External calendar sync
  const { syncStatus, syncCalendar } = useCalendarSync();

  // Fetch initial data
  useEffect(() => {
    fetchCalendarData();
  }, [experienceId]);

  // Handle real-time updates
  useEffect(() => {
    if (lastMessage) {
      handleRealtimeUpdate(lastMessage);
    }
  }, [lastMessage]);

  const fetchCalendarData = async () => {
    setIsLoading(true);
    try {
      const [eventsData, resourcesData] = await Promise.all([
        fetch(`/api/calendar/events${experienceId ? `?experienceId=${experienceId}` : ''}`),
        showResources ? fetch('/api/calendar/resources') : Promise.resolve(null)
      ]);

      const events = await eventsData.json();
      setEvents(events.data);

      if (resourcesData) {
        const resources = await resourcesData.json();
        setResources(resources.data);
      }
    } catch (error) {
      console.error('Error fetching calendar data:', error);
    } finally {
      setIsLoading(false);
    }
  };

  const handleRealtimeUpdate = (message: any) => {
    switch (message.type) {
      case 'event-created':
        setEvents(prev => [...prev, message.event]);
        break;
      case 'event-updated':
        setEvents(prev => prev.map(e => 
          e.id === message.event.id ? message.event : e
        ));
        break;
      case 'event-deleted':
        setEvents(prev => prev.filter(e => e.id !== message.eventId));
        break;
      case 'availability-changed':
        fetchCalendarData(); // Refetch for complex changes
        break;
    }
  };

  const handleDateSelect = useCallback((selectInfo: any) => {
    const slot: BookingSlot = {
      start: selectInfo.start,
      end: selectInfo.end,
      allDay: selectInfo.allDay,
      resource: selectInfo.resource
    };

    // Animate selection
    controls.start({
      scale: [1, 1.05, 1],
      transition: { duration: 0.3 }
    });

    setSelectedDate(selectInfo.start);
    onSlotSelect(slot);

    // Send selection to other connected users
    sendMessage({
      type: 'slot-selected',
      slot,
      userId: 'current-user-id' // Replace with actual user ID
    });
  }, [onSlotSelect, sendMessage, controls]);

  const handleEventClick = useCallback((clickInfo: any) => {
    // Implement event details modal or action
    console.log('Event clicked:', clickInfo.event);
  }, []);

  const handleEventDrop = useCallback(async (dropInfo: any) => {
    try {
      const response = await fetch(`/api/calendar/events/${dropInfo.event.id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          start: dropInfo.event.start,
          end: dropInfo.event.end,
          resourceId: dropInfo.newResource?.id
        })
      });

      if (!response.ok) {
        dropInfo.revert();
      }
    } catch (error) {
      console.error('Error updating event:', error);
      dropInfo.revert();
    }
  }, []);

  const handleEventResize = useCallback(async (resizeInfo: any) => {
    try {
      const response = await fetch(`/api/calendar/events/${resizeInfo.event.id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          start: resizeInfo.event.start,
          end: resizeInfo.event.end
        })
      });

      if (!response.ok) {
        resizeInfo.revert();
      }
    } catch (error) {
      console.error('Error resizing event:', error);
      resizeInfo.revert();
    }
  }, []);

  // Custom event content with animations
  const renderEventContent = (eventInfo: any) => {
    return (
      <motion.div
        initial={{ opacity: 0, scale: 0.8 }}
        animate={{ opacity: 1, scale: 1 }}
        whileHover={{ scale: 1.05 }}
        className="calendar-event"
        style={{
          backgroundColor: eventInfo.event.backgroundColor,
          color: eventInfo.event.textColor,
          padding: '2px 4px',
          borderRadius: '4px',
          overflow: 'hidden',
          cursor: 'pointer'
        }}
      >
        <div className="event-time">
          {eventInfo.timeText}
        </div>
        <div className="event-title">
          {eventInfo.event.title}
        </div>
        {eventInfo.event.extendedProps.attendees && (
          <div className="event-attendees">
            <span className="attendee-count">
              {eventInfo.event.extendedProps.attendees.length} asistentes
            </span>
          </div>
        )}
      </motion.div>
    );
  };

  // Toggle 3D view
  const toggle3DView = () => {
    setShow3D(!show3D);
    controls.start({
      rotateY: show3D ? 0 : 180,
      transition: { duration: 0.6 }
    });
  };

  if (isLoading) {
    return <CalendarSkeleton />;
  }

  return (
    <div className="booking-calendar-container">
      {/* Calendar Header Controls */}
      <div className="calendar-controls">
        <div className="view-selector">
          <button
            className={`view-btn ${view === 'month' ? 'active' : ''}`}
            onClick={() => calendarRef.current?.getApi().changeView('dayGridMonth')}
          >
            Mes
          </button>
          <button
            className={`view-btn ${view === 'week' ? 'active' : ''}`}
            onClick={() => calendarRef.current?.getApi().changeView('timeGridWeek')}
          >
            Semana
          </button>
          <button
            className={`view-btn ${view === 'day' ? 'active' : ''}`}
            onClick={() => calendarRef.current?.getApi().changeView('timeGridDay')}
          >
            Día
          </button>
          {showResources && (
            <button
              className={`view-btn ${view === 'timeline' ? 'active' : ''}`}
              onClick={() => calendarRef.current?.getApi().changeView('resourceTimeline')}
            >
              Timeline
            </button>
          )}
          <button
            className="view-btn special"
            onClick={toggle3DView}
          >
            Vista 3D
          </button>
        </div>

        <div className="sync-controls">
          <button
            className="sync-btn"
            onClick={() => syncCalendar('google')}
            disabled={syncStatus === 'syncing'}
          >
            {syncStatus === 'syncing' ? 'Sincronizando...' : 'Sync Google'}
          </button>
          <button
            className="sync-btn"
            onClick={() => syncCalendar('outlook')}
            disabled={syncStatus === 'syncing'}
          >
            Sync Outlook
          </button>
        </div>
      </div>

      {/* Main Calendar View */}
      <AnimatePresence>
        {!show3D ? (
          <motion.div
            key="calendar-2d"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1, ...controls }}
            exit={{ opacity: 0 }}
            className="calendar-wrapper"
          >
            <FullCalendar
              ref={calendarRef}
              plugins={[
                dayGridPlugin,
                timeGridPlugin,
                interactionPlugin,
                listPlugin,
                multiMonthPlugin,
                resourceTimelinePlugin,
                adaptivePlugin
              ]}
              initialView={view === 'timeline' ? 'resourceTimelineMonth' : `${view}Grid`}
              headerToolbar={{
                left: 'prev,next today',
                center: 'title',
                right: 'dayGridMonth,timeGridWeek,timeGridDay,listWeek'
              }}
              events={events}
              resources={resources}
              editable={true}
              droppable={true}
              selectable={true}
              selectMirror={true}
              dayMaxEvents={true}
              weekends={true}
              nowIndicator={true}
              height="auto"
              locale="es"
              timeZone="America/Mexico_City"
              businessHours={{
                daysOfWeek: [1, 2, 3, 4, 5, 6],
                startTime: '09:00',
                endTime: '20:00'
              }}
              select={handleDateSelect}
              eventClick={handleEventClick}
              eventDrop={handleEventDrop}
              eventResize={handleEventResize}
              eventContent={renderEventContent}
              dayCellDidMount={(info) => {
                // Add custom styling to specific dates
                if (info.date.getDay() === 0) {
                  info.el.style.backgroundColor = '#f0f0f0';
                }
              }}
              slotLabelFormat={{
                hour: 'numeric',
                minute: '2-digit',
                omitZeroMinute: false,
                meridiem: 'short'
              }}
              eventTimeFormat={{
                hour: 'numeric',
                minute: '2-digit',
                meridiem: 'short'
              }}
            />
          </motion.div>
        ) : (
          <motion.div
            key="calendar-3d"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            className="calendar-3d-wrapper"
          >
            <Calendar3D events={events} onDateSelect={handleDateSelect} />
          </motion.div>
        )}
      </AnimatePresence>

      {/* Selected Date Details */}
      <AnimatePresence>
        {selectedDate && (
          <motion.div
            className="selected-date-details"
            initial={{ y: 50, opacity: 0 }}
            animate={{ y: 0, opacity: 1 }}
            exit={{ y: 50, opacity: 0 }}
          >
            <h3>Fecha seleccionada</h3>
            <p>{selectedDate.toLocaleDateString('es-MX', {
              weekday: 'long',
              year: 'numeric',
              month: 'long',
              day: 'numeric'
            })}</p>
            <button onClick={() => setSelectedDate(null)}>Cerrar</button>
          </motion.div>
        )}
      </AnimatePresence>
    </div>
  );
};

// 3D Calendar Component
const Calendar3D: React.FC<{
  events: CalendarEvent[];
  onDateSelect: (date: Date) => void;
}> = ({ events, onDateSelect }) => {
  // Implement 3D calendar visualization using Three.js
  // This is a placeholder for the actual 3D implementation
  return (
    <Canvas camera={{ position: [0, 0, 5] }}>
      <ambientLight intensity={0.5} />
      <pointLight position={[10, 10, 10]} />
      {/* 3D calendar implementation */}
      <mesh>
        <boxGeometry args={[1, 1, 1]} />
        <meshStandardMaterial color="orange" />
      </mesh>
    </Canvas>
  );
};

// Calendar Skeleton Loader
const CalendarSkeleton: React.FC = () => {
  return (
    <div className="calendar-skeleton">
      <div className="skeleton-header" />
      <div className="skeleton-grid">
        {Array.from({ length: 35 }).map((_, i) => (
          <div key={i} className="skeleton-cell" />
        ))}
      </div>
    </div>
  );
};
```

### 5.2 Drag-and-Drop con Animaciones Fluidas

```tsx
// components/calendar/DraggableBooking.tsx
import React, { useState } from 'react';
import { useDrag, useDrop, DndProvider } from 'react-dnd';
import { HTML5Backend } from 'react-dnd-html5-backend';
import { TouchBackend } from 'react-dnd-touch-backend';
import { motion, useMotionValue, useTransform, PanInfo } from 'framer-motion';
import { useSpring, animated } from '@react-spring/web';
import { useMeasure } from 'react-use';

interface DraggableBookingProps {
  booking: {
    id: string;
    title: string;
    duration: number;
    color: string;
    icon?: string;
  };
  onDrop: (bookingId: string, targetDate: Date, targetTime?: string) => void;
}

export const DraggableBooking: React.FC<DraggableBookingProps> = ({
  booking,
  onDrop
}) => {
  const [isDragging, setIsDragging] = useState(false);
  const [{ opacity }, dragRef] = useDrag({
    type: 'booking',
    item: { ...booking },
    collect: (monitor) => ({
      opacity: monitor.isDragging() ? 0.5 : 1
    }),
    end: (item, monitor) => {
      const dropResult = monitor.getDropResult();
      if (dropResult) {
        onDrop(item.id, dropResult.date, dropResult.time);
      }
      setIsDragging(false);
    },
    begin: () => setIsDragging(true)
  });

  const springProps = useSpring({
    scale: isDragging ? 1.1 : 1,
    shadow: isDragging ? 15 : 5,
    config: { tension: 300, friction: 20 }
  });

  return (
    <animated.div
      ref={dragRef}
      style={{
        opacity,
        transform: springProps.scale.to(s => `scale(${s})`),
        boxShadow: springProps.shadow.to(s => `0 ${s}px ${s * 2}px rgba(0,0,0,0.2)`),
        cursor: 'grab'
      }}
      className="draggable-booking"
    >
      <motion.div
        whileHover={{ scale: 1.05 }}
        whileTap={{ scale: 0.95 }}
        style={{
          backgroundColor: booking.color,
          padding: '12px',
          borderRadius: '8px',
          color: 'white'
        }}
      >
        {booking.icon && <span className="booking-icon">{booking.icon}</span>}
        <h4>{booking.title}</h4>
        <p>{booking.duration} min</p>
      </motion.div>
    </animated.div>
  );
};

interface DroppableTimeSlotProps {
  date: Date;
  time?: string;
  isAvailable: boolean;
  children: React.ReactNode;
}

export const DroppableTimeSlot: React.FC<DroppableTimeSlotProps> = ({
  date,
  time,
  isAvailable,
  children
}) => {
  const [isOver, setIsOver] = useState(false);
  const [ref, bounds] = useMeasure<HTMLDivElement>();
  
  const [{ canDrop }, dropRef] = useDrop({
    accept: 'booking',
    drop: () => ({ date, time }),
    collect: (monitor) => ({
      canDrop: monitor.canDrop() && isAvailable
    }),
    hover: (item, monitor) => {
      setIsOver(monitor.isOver());
    }
  });

  const springProps = useSpring({
    backgroundColor: isOver && canDrop ? '#4ade80' : isOver ? '#f87171' : 'transparent',
    scale: isOver ? 1.02 : 1,
    config: { duration: 200 }
  });

  return (
    <animated.div
      ref={(el) => {
        dropRef(el);
        if (ref) ref.current = el;
      }}
      style={{
        ...springProps,
        transition: 'all 0.2s ease'
      }}
      className={`droppable-slot ${canDrop ? 'can-drop' : ''} ${!isAvailable ? 'unavailable' : ''}`}
    >
      <motion.div
        animate={{
          borderColor: isOver ? (canDrop ? '#10b981' : '#ef4444') : '#e5e7eb',
          borderWidth: isOver ? '2px' : '1px'
        }}
        transition={{ duration: 0.2 }}
        style={{
          border: '1px solid #e5e7eb',
          borderRadius: '4px',
          padding: '8px',
          minHeight: '60px',
          position: 'relative'
        }}
      >
        {children}
        {isOver && (
          <motion.div
            initial={{ opacity: 0, scale: 0.8 }}
            animate={{ opacity: 1, scale: 1 }}
            exit={{ opacity: 0, scale: 0.8 }}
            className="drop-indicator"
            style={{
              position: 'absolute',
              top: '50%',
              left: '50%',
              transform: 'translate(-50%, -50%)',
              pointerEvents: 'none'
            }}
          >
            {canDrop ? (
              <div className="text-green-600">✓ Soltar aquí</div>
            ) : (
              <div className="text-red-600">✗ No disponible</div>
            )}
          </motion.div>
        )}
      </motion.div>
    </animated.div>
  );
};

// Advanced Gesture-based Booking
export const GestureBooking: React.FC<{
  onSwipe: (direction: 'left' | 'right') => void;
  onTap: () => void;
  children: React.ReactNode;
}> = ({ onSwipe, onTap, children }) => {
  const x = useMotionValue(0);
  const background = useTransform(
    x,
    [-100, 0, 100],
    ['#ef4444', '#3b82f6', '#10b981']
  );

  const handleDragEnd = (event: any, info: PanInfo) => {
    if (Math.abs(info.offset.x) > 100) {
      onSwipe(info.offset.x > 0 ? 'right' : 'left');
    }
  };

  return (
    <motion.div
      drag="x"
      dragConstraints={{ left: -200, right: 200 }}
      dragElastic={0.2}
      onDragEnd={handleDragEnd}
      onTap={onTap}
      style={{ x, background }}
      whileHover={{ scale: 1.02 }}
      whileTap={{ scale: 0.98 }}
      className="gesture-booking"
    >
      {children}
      <motion.div
        className="swipe-indicators"
        style={{
          position: 'absolute',
          top: 0,
          left: 0,
          right: 0,
          bottom: 0,
          display: 'flex',
          alignItems: 'center',
          justifyContent: 'space-between',
          padding: '0 16px',
          pointerEvents: 'none'
        }}
      >
        <motion.div
          animate={{ opacity: x.get() < -50 ? 1 : 0 }}
          className="cancel-indicator"
        >
          ❌ Cancelar
        </motion.div>
        <motion.div
          animate={{ opacity: x.get() > 50 ? 1 : 0 }}
          className="confirm-indicator"
        >
          ✅ Confirmar
        </motion.div>
      </motion.div>
    </motion.div>
  );
};
```

### 5.3 Vista 3D Opcional

```tsx
// components/calendar/Calendar3DView.tsx
import React, { useRef, useState, useEffect } from 'react';
import { Canvas, useFrame, useThree } from '@react-three/fiber';
import { 
  OrbitControls, 
  Text, 
  Box, 
  Plane,
  Float,
  MeshReflectorMaterial,
  Environment,
  PerspectiveCamera
} from '@react-three/drei';
import * as THREE from 'three';
import { motion } from 'framer-motion-3d';

interface Calendar3DViewProps {
  events: any[];
  currentDate: Date;
  onDateSelect: (date: Date) => void;
}

export const Calendar3DView: React.FC<Calendar3DViewProps> = ({
  events,
  currentDate,
  onDateSelect
}) => {
  return (
    <div style={{ width: '100%', height: '600px' }}>
      <Canvas shadows>
        <PerspectiveCamera makeDefault position={[0, 15, 20]} fov={60} />
        <Environment preset="city" />
        <ambientLight intensity={0.5} />
        <spotLight
          position={[10, 20, 10]}
          angle={0.3}
          penumbra={1}
          intensity={1}
          castShadow
        />
        
        <CalendarGrid 
          currentDate={currentDate}
          events={events}
          onDateSelect={onDateSelect}
        />
        
        <OrbitControls 
          enablePan={true}
          enableZoom={true}
          enableRotate={true}
          minDistance={10}
          maxDistance={50}
          maxPolarAngle={Math.PI / 2.5}
        />
        
        {/* Reflective floor */}
        <Plane
          args={[50, 50]}
          rotation={[-Math.PI / 2, 0, 0]}
          position={[0, -0.5, 0]}
        >
          <MeshReflectorMaterial
            blur={[300, 100]}
            resolution={2048}
            mixBlur={1}
            mixStrength={50}
            roughness={1}
            depthScale={1.2}
            minDepthThreshold={0.4}
            maxDepthThreshold={1.4}
            color="#101010"
            metalness={0.5}
          />
        </Plane>
      </Canvas>
    </div>
  );
};

const CalendarGrid: React.FC<{
  currentDate: Date;
  events: any[];
  onDateSelect: (date: Date) => void;
}> = ({ currentDate, events, onDateSelect }) => {
  const [hoveredDate, setHoveredDate] = useState<number | null>(null);
  const groupRef = useRef<THREE.Group>(null);

  useFrame((state) => {
    if (groupRef.current) {
      groupRef.current.rotation.y = Math.sin(state.clock.elapsedTime * 0.1) * 0.02;
    }
  });

  const getDaysInMonth = (date: Date) => {
    return new Date(date.getFullYear(), date.getMonth() + 1, 0).getDate();
  };

  const getFirstDayOfMonth = (date: Date) => {
    return new Date(date.getFullYear(), date.getMonth(), 1).getDay();
  };

  const daysInMonth = getDaysInMonth(currentDate);
  const firstDay = getFirstDayOfMonth(currentDate);
  const weeks = Math.ceil((daysInMonth + firstDay) / 7);

  const getEventsForDate = (day: number) => {
    return events.filter(event => {
      const eventDate = new Date(event.start);
      return eventDate.getDate() === day && 
             eventDate.getMonth() === currentDate.getMonth() &&
             eventDate.getFullYear() === currentDate.getFullYear();
    });
  };

  return (
    <group ref={groupRef}>
      {/* Month title */}
      <Float speed={2} rotationIntensity={0.5} floatIntensity={0.5}>
        <Text
          position={[0, 8, 0]}
          fontSize={2}
          color="#ffffff"
          anchorX="center"
          anchorY="middle"
          font="/fonts/bold.woff"
        >
          {currentDate.toLocaleDateString('es-MX', { 
            month: 'long', 
            year: 'numeric' 
          }).toUpperCase()}
        </Text>
      </Float>

      {/* Day labels */}
      {['Dom', 'Lun', 'Mar', 'Mié', 'Jue', 'Vie', 'Sáb'].map((day, i) => (
        <Text
          key={day}
          position={[(i - 3) * 2.5, 5, 0]}
          fontSize={0.8}
          color="#888888"
          anchorX="center"
          anchorY="middle"
        >
          {day}
        </Text>
      ))}

      {/* Calendar days */}
      {Array.from({ length: weeks * 7 }).map((_, index) => {
        const dayNumber = index - firstDay + 1;
        const isValidDay = dayNumber > 0 && dayNumber <= daysInMonth;
        const x = ((index % 7) - 3) * 2.5;
        const z = Math.floor(index / 7) * 2.5 - 2;
        const dayEvents = isValidDay ? getEventsForDate(dayNumber) : [];

        if (!isValidDay) return null;

        return (
          <DayBox
            key={index}
            position={[x, 0, z]}
            day={dayNumber}
            events={dayEvents}
            isToday={dayNumber === new Date().getDate() && 
                     currentDate.getMonth() === new Date().getMonth()}
            isHovered={hoveredDate === dayNumber}
            onHover={() => setHoveredDate(dayNumber)}
            onUnhover={() => setHoveredDate(null)}
            onClick={() => {
              const selectedDate = new Date(
                currentDate.getFullYear(),
                currentDate.getMonth(),
                dayNumber
              );
              onDateSelect(selectedDate);
            }}
          />
        );
      })}
    </group>
  );
};

const DayBox: React.FC<{
  position: [number, number, number];
  day: number;
  events: any[];
  isToday: boolean;
  isHovered: boolean;
  onHover: () => void;
  onUnhover: () => void;
  onClick: () => void;
}> = ({ position, day, events, isToday, isHovered, onHover, onUnhover, onClick }) => {
  const meshRef = useRef<THREE.Mesh>(null);
  const [hovered, setHovered] = useState(false);

  useFrame((state) => {
    if (meshRef.current) {
      meshRef.current.position.y = 
        (hovered ? 0.5 : 0) + 
        Math.sin(state.clock.elapsedTime * 2 + day) * 0.05;
    }
  });

  const baseColor = isToday ? '#3b82f6' : '#1f2937';
  const hoverColor = '#10b981';
  const eventColors = ['#ef4444', '#f59e0b', '#10b981', '#3b82f6'];

  return (
    <group position={position}>
      <motion.mesh
        ref={meshRef}
        onPointerOver={(e) => {
          e.stopPropagation();
          setHovered(true);
          onHover();
        }}
        onPointerOut={() => {
          setHovered(false);
          onUnhover();
        }}
        onClick={(e) => {
          e.stopPropagation();
          onClick();
        }}
        castShadow
        receiveShadow
        animate={{
          scale: hovered ? 1.1 : 1,
          rotateY: hovered ? 0.1 : 0
        }}
      >
        <Box args={[2, 0.5, 2]}>
          <meshStandardMaterial 
            color={hovered ? hoverColor : baseColor}
            emissive={hovered ? hoverColor : baseColor}
            emissiveIntensity={hovered ? 0.2 : 0.05}
          />
        </Box>
      </motion.mesh>

      {/* Day number */}
      <Text
        position={[0, 0.3, 0]}
        fontSize={0.6}
        color="#ffffff"
        anchorX="center"
        anchorY="middle"
      >
        {day}
      </Text>

      {/* Event indicators */}
      {events.slice(0, 3).map((event, i) => (
        <Float key={i} speed={3} floatIntensity={0.5}>
          <Box
            position={[-0.6 + i * 0.6, 0.8 + i * 0.1, 0]}
            args={[0.4, 0.1, 0.4]}
            castShadow
          >
            <meshStandardMaterial 
              color={eventColors[i % eventColors.length]}
              emissive={eventColors[i % eventColors.length]}
              emissiveIntensity={0.5}
            />
          </Box>
        </Float>
      ))}

      {events.length > 3 && (
        <Text
          position={[0.7, 0.8, 0]}
          fontSize={0.3}
          color="#ffffff"
        >
          +{events.length - 3}
        </Text>
      )}
    </group>
  );
};

// Mini calendar for month navigation
export const MiniCalendar3D: React.FC<{
  onMonthChange: (date: Date) => void;
}> = ({ onMonthChange }) => {
  const [currentMonth, setCurrentMonth] = useState(new Date());

  const navigateMonth = (direction: number) => {
    const newDate = new Date(currentMonth);
    newDate.setMonth(newDate.getMonth() + direction);
    setCurrentMonth(newDate);
    onMonthChange(newDate);
  };

  return (
    <div style={{ width: '300px', height: '300px' }}>
      <Canvas>
        <PerspectiveCamera makeDefault position={[0, 5, 10]} fov={50} />
        <ambientLight intensity={0.8} />
        <pointLight position={[10, 10, 10]} />
        
        {/* Month navigation */}
        <group>
          <Text
            position={[0, 3, 0]}
            fontSize={1}
            color="#ffffff"
            anchorX="center"
          >
            {currentMonth.toLocaleDateString('es-MX', { 
              month: 'short', 
              year: 'numeric' 
            })}
          </Text>
          
          {/* Navigation arrows */}
          <mesh
            position={[-3, 3, 0]}
            onClick={() => navigateMonth(-1)}
          >
            <coneGeometry args={[0.5, 1, 3]} />
            <meshStandardMaterial color="#3b82f6" />
          </mesh>
          
          <mesh
            position={[3, 3, 0]}
            onClick={() => navigateMonth(1)}
            rotation={[0, 0, Math.PI]}
          >
            <coneGeometry args={[0.5, 1, 3]} />
            <meshStandardMaterial color="#3b82f6" />
          </mesh>
        </group>
        
        <OrbitControls enableZoom={false} enablePan={false} />
      </Canvas>
    </div>
  );
};
```

### 5.4 Sincronización en Tiempo Real con WebSockets

```typescript
// lib/calendar/realtime-sync.ts
import { io, Socket } from 'socket.io-client';
import { EventEmitter } from 'events';

interface CalendarSyncOptions {
  userId: string;
  experienceId?: string;
  onConnect?: () => void;
  onDisconnect?: () => void;
  onError?: (error: Error) => void;
}

export class CalendarRealtimeSync extends EventEmitter {
  private socket: Socket | null = null;
  private options: CalendarSyncOptions;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private syncQueue: any[] = [];
  private isOnline = navigator.onLine;

  constructor(options: CalendarSyncOptions) {
    super();
    this.options = options;
    this.initialize();
  }

  private initialize() {
    // Initialize WebSocket connection
    this.socket = io(process.env.NEXT_PUBLIC_WS_URL || 'ws://localhost:3001', {
      query: {
        userId: this.options.userId,
        experienceId: this.options.experienceId,
        type: 'calendar'
      },
      transports: ['websocket'],
      reconnection: true,
      reconnectionAttempts: this.maxReconnectAttempts,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 5000
    });

    this.setupEventHandlers();
    this.setupOfflineHandling();
  }

  private setupEventHandlers() {
    if (!this.socket) return;

    this.socket.on('connect', () => {
      console.log('Calendar sync connected');
      this.reconnectAttempts = 0;
      this.processSyncQueue();
      this.options.onConnect?.();
      this.emit('connected');
    });

    this.socket.on('disconnect', (reason) => {
      console.log('Calendar sync disconnected:', reason);
      this.options.onDisconnect?.();
      this.emit('disconnected', reason);
    });

    this.socket.on('error', (error) => {
      console.error('Calendar sync error:', error);
      this.options.onError?.(error);
      this.emit('error', error);
    });

    // Calendar-specific events
    this.socket.on('booking:created', (data) => {
      this.emit('booking-created', data);
    });

    this.socket.on('booking:updated', (data) => {
      this.emit('booking-updated', data);
    });

    this.socket.on('booking:cancelled', (data) => {
      this.emit('booking-cancelled', data);
    });

    this.socket.on('availability:changed', (data) => {
      this.emit('availability-changed', data);
    });

    this.socket.on('slot:locked', (data) => {
      this.emit('slot-locked', data);
    });

    this.socket.on('slot:released', (data) => {
      this.emit('slot-released', data);
    });

    this.socket.on('user:viewing', (data) => {
      this.emit('user-viewing', data);
    });

    this.socket.on('sync:conflict', (data) => {
      this.handleSyncConflict(data);
    });
  }

  private setupOfflineHandling() {
    window.addEventListener('online', () => {
      this.isOnline = true;
      this.emit('online');
      this.reconnect();
    });

    window.addEventListener('offline', () => {
      this.isOnline = false;
      this.emit('offline');
    });
  }

  // Public methods

  public createBooking(bookingData: any) {
    const event = {
      type: 'booking:create',
      data: {
        ...bookingData,
        timestamp: Date.now(),
        clientId: this.generateClientId()
      }
    };

    if (this.isConnected()) {
      this.socket?.emit('booking:create', event.data);
    } else {
      this.queueSync(event);
    }
  }

  public updateBooking(bookingId: string, updates: any) {
    const event = {
      type: 'booking:update',
      data: {
        bookingId,
        updates,
        timestamp: Date.now(),
        clientId: this.generateClientId()
      }
    };

    if (this.isConnected()) {
      this.socket?.emit('booking:update', event.data);
    } else {
      this.queueSync(event);
    }
  }

  public cancelBooking(bookingId: string, reason?: string) {
    const event = {
      type: 'booking:cancel',
      data: {
        bookingId,
        reason,
        timestamp: Date.now(),
        clientId: this.generateClientId()
      }
    };

    if (this.isConnected()) {
      this.socket?.emit('booking:cancel', event.data);
    } else {
      this.queueSync(event);
    }
  }

  public lockSlot(slotData: any) {
    if (this.isConnected()) {
      this.socket?.emit('slot:lock', {
        ...slotData,
        userId: this.options.userId,
        timestamp: Date.now()
      });
    }
  }

  public releaseSlot(slotId: string) {
    if (this.isConnected()) {
      this.socket?.emit('slot:release', {
        slotId,
        userId: this.options.userId,
        timestamp: Date.now()
      });
    }
  }

  public notifyViewing(date: Date, view: string) {
    if (this.isConnected()) {
      this.socket?.emit('user:viewing', {
        userId: this.options.userId,
        date: date.toISOString(),
        view,
        timestamp: Date.now()
      });
    }
  }

  public joinExperienceRoom(experienceId: string) {
    if (this.isConnected()) {
      this.socket?.emit('room:join', {
        room: `experience-${experienceId}`,
        userId: this.options.userId
      });
    }
  }

  public leaveExperienceRoom(experienceId: string) {
    if (this.isConnected()) {
      this.socket?.emit('room:leave', {
        room: `experience-${experienceId}`,
        userId: this.options.userId
      });
    }
  }

  // Sync queue management

  private queueSync(event: any) {
    this.syncQueue.push(event);
    this.emit('queued', event);
    
    // Store in localStorage for persistence
    this.persistSyncQueue();
  }

  private async processSyncQueue() {
    if (this.syncQueue.length === 0) return;

    console.log(`Processing ${this.syncQueue.length} queued events`);

    for (const event of this.syncQueue) {
      try {
        await this.processSyncEvent(event);
      } catch (error) {
        console.error('Error processing sync event:', error);
      }
    }

    this.syncQueue = [];
    this.clearPersistedQueue();
  }

  private async processSyncEvent(event: any) {
    return new Promise((resolve, reject) => {
      if (!this.socket) {
        reject(new Error('Socket not initialized'));
        return;
      }

      const timeout = setTimeout(() => {
        reject(new Error('Sync timeout'));
      }, 5000);

      this.socket.emit(event.type, event.data, (response: any) => {
        clearTimeout(timeout);
        if (response.error) {
          reject(new Error(response.error));
        } else {
          resolve(response);
        }
      });
    });
  }

  private persistSyncQueue() {
    try {
      localStorage.setItem(
        `calendar-sync-queue-${this.options.userId}`,
        JSON.stringify(this.syncQueue)
      );
    } catch (error) {
      console.error('Error persisting sync queue:', error);
    }
  }

  private loadPersistedQueue() {
    try {
      const stored = localStorage.getItem(`calendar-sync-queue-${this.options.userId}`);
      if (stored) {
        this.syncQueue = JSON.parse(stored);
      }
    } catch (error) {
      console.error('Error loading persisted queue:', error);
    }
  }

  private clearPersistedQueue() {
    try {
      localStorage.removeItem(`calendar-sync-queue-${this.options.userId}`);
    } catch (error) {
      console.error('Error clearing persisted queue:', error);
    }
  }

  // Conflict resolution

  private handleSyncConflict(conflict: any) {
    console.warn('Sync conflict detected:', conflict);
    
    // Emit conflict event for UI handling
    this.emit('conflict', conflict);
    
    // Auto-resolve based on strategy
    switch (conflict.type) {
      case 'booking_overlap':
        this.resolveBookingOverlap(conflict);
        break;
      
      case 'availability_mismatch':
        this.resolveAvailabilityMismatch(conflict);
        break;
      
      default:
        // Let UI handle unknown conflicts
        break;
    }
  }

  private resolveBookingOverlap(conflict: any) {
    // Implement last-write-wins or other strategy
    const resolution = {
      conflictId: conflict.id,
      resolution: 'last_write_wins',
      winner: conflict.server,
      timestamp: Date.now()
    };

    this.socket?.emit('conflict:resolve', resolution);
  }

  private resolveAvailabilityMismatch(conflict: any) {
    // Server availability always wins
    this.emit('availability-update-required', conflict.server);
  }

  // Utility methods

  private isConnected(): boolean {
    return this.socket?.connected || false;
  }

  private generateClientId(): string {
    return `${this.options.userId}-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  public reconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      this.socket?.connect();
    }
  }

  public disconnect() {
    this.socket?.disconnect();
    this.socket = null;
  }

  public getConnectionStatus() {
    return {
      connected: this.isConnected(),
      online: this.isOnline,
      queuedEvents: this.syncQueue.length,
      reconnectAttempts: this.reconnectAttempts
    };
  }
}

// React Hook for calendar sync
export function useCalendarSync(userId: string, experienceId?: string) {
  const [sync, setSync] = useState<CalendarRealtimeSync | null>(null);
  const [status, setStatus] = useState<'disconnected' | 'connected' | 'syncing'>('disconnected');
  const [queueSize, setQueueSize] = useState(0);

  useEffect(() => {
    const calendarSync = new CalendarRealtimeSync({
      userId,
      experienceId,
      onConnect: () => setStatus('connected'),
      onDisconnect: () => setStatus('disconnected'),
      onError: (error) => console.error('Calendar sync error:', error)
    });

    calendarSync.on('queued', () => {
      const status = calendarSync.getConnectionStatus();
      setQueueSize(status.queuedEvents);
    });

    calendarSync.on('connected', () => {
      setStatus('connected');
      setQueueSize(0);
    });

    setSync(calendarSync);

    return () => {
      calendarSync.disconnect();
    };
  }, [userId, experienceId]);

  return {
    sync,
    status,
    queueSize,
    createBooking: (data: any) => sync?.createBooking(data),
    updateBooking: (id: string, updates: any) => sync?.updateBooking(id, updates),
    cancelBooking: (id: string, reason?: string) => sync?.cancelBooking(id, reason),
    lockSlot: (slot: any) => sync?.lockSlot(slot),
    releaseSlot: (slotId: string) => sync?.releaseSlot(slotId)
  };
}
```

### 5.5 Integración con Calendarios Externos

```typescript
// lib/calendar/external-sync.ts
import { google } from 'googleapis';
import { OAuth2Client } from 'google-auth-library';
import ical from 'node-ical';
import { DateTime } from 'luxon';

export class ExternalCalendarSync {
  private googleOAuth2Client: OAuth2Client;
  
  constructor() {
    this.googleOAuth2Client = new google.auth.OAuth2(
      process.env.GOOGLE_CLIENT_ID,
      process.env.GOOGLE_CLIENT_SECRET,
      process.env.GOOGLE_REDIRECT_URI
    );
  }

  // Google Calendar Integration
  async syncWithGoogle(userId: string, accessToken: string) {
    this.googleOAuth2Client.setCredentials({ access_token: accessToken });
    
    const calendar = google.calendar({ 
      version: 'v3', 
      auth: this.googleOAuth2Client 
    });

    try {
      // Get events from Google Calendar
      const response = await calendar.events.list({
        calendarId: 'primary',
        timeMin: new Date().toISOString(),
        maxResults: 100,
        singleEvents: true,
        orderBy: 'startTime'
      });

      const googleEvents = response.data.items || [];
      
      // Convert Google events to our format
      const convertedEvents = googleEvents.map(event => ({
        id: event.id,
        title: event.summary,
        start: event.start?.dateTime || event.start?.date,
        end: event.end?.dateTime || event.end?.date,
        description: event.description,
        location: event.location,
        attendees: event.attendees?.map(a => ({
          email: a.email,
          displayName: a.displayName,
          responseStatus: a.responseStatus
        })),
        source: 'google',
        originalEvent: event
      }));

      // Sync our bookings to Google
      await this.pushBookingsToGoogle(userId, calendar);

      return {
        success: true,
        events: convertedEvents,
        syncedAt: new Date()
      };
    } catch (error) {
      console.error('Google Calendar sync error:', error);
      throw error;
    }
  }

  private async pushBookingsToGoogle(userId: string, calendar: any) {
    // Get user's bookings from our database
    const bookings = await this.getUserBookings(userId);

    for (const booking of bookings) {
      if (!booking.googleEventId) {
        // Create new event in Google Calendar
        const event = {
          summary: booking.title,
          description: booking.description,
          start: {
            dateTime: booking.startTime,
            timeZone: 'America/Mexico_City'
          },
          end: {
            dateTime: booking.endTime,
            timeZone: 'America/Mexico_City'
          },
          attendees: booking.attendees?.map(a => ({ email: a.email })),
          reminders: {
            useDefault: false,
            overrides: [
              { method: 'email', minutes: 24 * 60 },
              { method: 'popup', minutes: 30 }
            ]
          }
        };

        try {
          const response = await calendar.events.insert({
            calendarId: 'primary',
            resource: event
          });

          // Update our booking with Google event ID
          await this.updateBookingGoogleId(booking.id, response.data.id);
        } catch (error) {
          console.error('Error creating Google event:', error);
        }
      } else {
        // Update existing event
        try {
          await calendar.events.update({
            calendarId: 'primary',
            eventId: booking.googleEventId,
            resource: {
              summary: booking.title,
              description: booking.description,
              start: {
                dateTime: booking.startTime,
                timeZone: 'America/Mexico_City'
              },
              end: {
                dateTime: booking.endTime,
                timeZone: 'America/Mexico_City'
              }
            }
          });
        } catch (error) {
          console.error('Error updating Google event:', error);
        }
      }
    }
  }

  // Outlook/Exchange Calendar Integration
  async syncWithOutlook(userId: string, accessToken: string) {
    const graphClient = this.getGraphClient(accessToken);

    try {
      // Get events from Outlook
      const response = await graphClient
        .api('/me/calendar/events')
        .select('subject,start,end,location,attendees,body')
        .filter(`start/dateTime ge '${new Date().toISOString()}'`)
        .orderby('start/dateTime')
        .top(100)
        .get();

      const outlookEvents = response.value.map((event: any) => ({
        id: event.id,
        title: event.subject,
        start: event.start.dateTime,
        end: event.end.dateTime,
        description: event.body?.content,
        location: event.location?.displayName,
        attendees: event.attendees?.map((a: any) => ({
          email: a.emailAddress.address,
          displayName: a.emailAddress.name,
          responseStatus: a.status?.response
        })),
        source: 'outlook',
        originalEvent: event
      }));

      // Sync our bookings to Outlook
      await this.pushBookingsToOutlook(userId, graphClient);

      return {
        success: true,
        events: outlookEvents,
        syncedAt: new Date()
      };
    } catch (error) {
      console.error('Outlook sync error:', error);
      throw error;
    }
  }

  private async pushBookingsToOutlook(userId: string, graphClient: any) {
    const bookings = await this.getUserBookings(userId);

    for (const booking of bookings) {
      const event = {
        subject: booking.title,
        body: {
          contentType: 'HTML',
          content: booking.description
        },
        start: {
          dateTime: booking.startTime,
          timeZone: 'America/Mexico_City'
        },
        end: {
          dateTime: booking.endTime,
          timeZone: 'America/Mexico_City'
        },
        attendees: booking.attendees?.map(a => ({
          emailAddress: {
            address: a.email,
            name: a.name
          },
          type: 'required'
        }))
      };

      try {
        if (!booking.outlookEventId) {
          // Create new event
          const response = await graphClient
            .api('/me/calendar/events')
            .post(event);

          await this.updateBookingOutlookId(booking.id, response.id);
        } else {
          // Update existing event
          await graphClient
            .api(`/me/calendar/events/${booking.outlookEventId}`)
            .update(event);
        }
      } catch (error) {
        console.error('Error syncing to Outlook:', error);
      }
    }
  }

  // iCal Feed Integration
  async generateICalFeed(userId: string): Promise<string> {
    const bookings = await this.getUserBookings(userId);
    const icalendar = ical.createCalendar({
      domain: 'tu-dominio.com',
      prodId: '//Tu Dominio//Booking Calendar//ES',
      name: 'Mis Reservas',
      timezone: 'America/Mexico_City'
    });

    for (const booking of bookings) {
      icalendar.createEvent({
        start: new Date(booking.startTime),
        end: new Date(booking.endTime),
        summary: booking.title,
        description: booking.description,
        location: booking.location,
        url: `https://tu-dominio.com/bookings/${booking.id}`,
        organizer: {
          name: 'Tu Dominio',
          email: 'bookings@tu-dominio.com'
        },
        attendees: booking.attendees?.map(a => ({
          name: a.name,
          email: a.email,
          rsvp: true
        }))
      });
    }

    return icalendar.toString();
  }

  // CalDAV Integration
  async syncWithCalDAV(serverUrl: string, username: string, password: string) {
    // Implementation for CalDAV sync
    // This would use a CalDAV client library
    try {
      // Connect to CalDAV server
      // Fetch events
      // Convert to our format
      // Push our events
      
      return {
        success: true,
        events: [],
        syncedAt: new Date()
      };
    } catch (error) {
      console.error('CalDAV sync error:', error);
      throw error;
    }
  }

  // Helper methods
  private getGraphClient(accessToken: string) {
    // Initialize Microsoft Graph client
    // Implementation depends on your auth setup
    return null; // Placeholder
  }

  private async getUserBookings(userId: string) {
    // Fetch user bookings from database
    // Implementation depends on your database setup
    return []; // Placeholder
  }

  private async updateBookingGoogleId(bookingId: string, googleEventId: string) {
    // Update booking with Google event ID
    // Implementation depends on your database setup
  }

  private async updateBookingOutlookId(bookingId: string, outlookEventId: string) {
    // Update booking with Outlook event ID
    // Implementation depends on your database setup
  }

  // Two-way sync manager
  async performTwoWaySync(userId: string, providers: string[]) {
    const results = {
      google: null as any,
      outlook: null as any,
      conflicts: [] as any[],
      merged: [] as any[]
    };

    // Get all events from all providers
    if (providers.includes('google')) {
      try {
        const googleToken = await this.getUserGoogleToken(userId);
        results.google = await this.syncWithGoogle(userId, googleToken);
      } catch (error) {
        console.error('Google sync failed:', error);
      }
    }

    if (providers.includes('outlook')) {
      try {
        const outlookToken = await this.getUserOutlookToken(userId);
        results.outlook = await this.syncWithOutlook(userId, outlookToken);
      } catch (error) {
        console.error('Outlook sync failed:', error);
      }
    }

    // Merge and detect conflicts
    const allExternalEvents = [
      ...(results.google?.events || []),
      ...(results.outlook?.events || [])
    ];

    // Detect conflicts
    results.conflicts = this.detectConflicts(allExternalEvents);
    
    // Merge non-conflicting events
    results.merged = this.mergeEvents(allExternalEvents);

    return results;
  }

  private detectConflicts(events: any[]): any[] {
    const conflicts = [];
    
    for (let i = 0; i < events.length; i++) {
      for (let j = i + 1; j < events.length; j++) {
        if (this.eventsOverlap(events[i], events[j])) {
          conflicts.push({
            event1: events[i],
            event2: events[j],
            type: 'time_overlap'
          });
        }
      }
    }

    return conflicts;
  }

  private eventsOverlap(event1: any, event2: any): boolean {
    const start1 = new Date(event1.start).getTime();
    const end1 = new Date(event1.end).getTime();
    const start2 = new Date(event2.start).getTime();
    const end2 = new Date(event2.end).getTime();

    return (start1 < end2 && end1 > start2);
  }

  private mergeEvents(events: any[]): any[] {
    // Remove duplicates and merge similar events
    const merged = new Map();

    for (const event of events) {
      const key = `${event.title}-${event.start}`;
      
      if (!merged.has(key)) {
        merged.set(key, event);
      } else {
        // Merge additional data
        const existing = merged.get(key);
        existing.sources = [...(existing.sources || [existing.source]), event.source];
        existing.attendees = this.mergeAttendees(existing.attendees, event.attendees);
      }
    }

    return Array.from(merged.values());
  }

  private mergeAttendees(attendees1: any[], attendees2: any[]): any[] {
    const merged = new Map();
    
    for (const attendee of [...(attendees1 || []), ...(attendees2 || [])]) {
      if (!merged.has(attendee.email)) {
        merged.set(attendee.email, attendee);
      }
    }

    return Array.from(merged.values());
  }

  private async getUserGoogleToken(userId: string): Promise<string> {
    // Fetch user's Google token from database
    // Implementation depends on your auth setup
    return ''; // Placeholder
  }

  private async getUserOutlookToken(userId: string): Promise<string> {
    // Fetch user's Outlook token from database
    // Implementation depends on your auth setup
    return ''; // Placeholder
  }
}
```

---

## 6. SEGURIDAD Y MEJORES PRÁCTICAS

### 6.1 Implementación de HTTPS en Todos los Servicios

```yaml
# docker-compose-security.yml
version: '3.9'

services:
  # Traefik con Let's Encrypt
  traefik:
    image: traefik:v3.0
    command:
      # API Configuration
      - "--api.dashboard=true"
      - "--api.debug=false"
      
      # Providers
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=microservices-net"
      - "--providers.file.directory=/etc/traefik/dynamic"
      
      # Entrypoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      
      # HTTP to HTTPS redirect
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.web.http.redirections.entrypoint.permanent=true"
      
      # Let's Encrypt
      - "--certificatesresolvers.letsencrypt.acme.email=${ACME_EMAIL}"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.caserver=https://acme-v02.api.letsencrypt.org/directory"
      
      # Security
      - "--pilot.dashboard=false"
      - "--global.sendAnonymousUsage=false"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik-certificates:/letsencrypt
      - ./traefik/dynamic:/etc/traefik/dynamic:ro
    environment:
      - TRAEFIK_LOG_LEVEL=INFO
    labels:
      # Dashboard
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.${DOMAIN}`)"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls=true"
      - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      
      # Basic auth for dashboard
      - "traefik.http.middlewares.auth.basicauth.users=${TRAEFIK_DASHBOARD_USERS}"
      
      # Security headers
      - "traefik.http.middlewares.security-headers.headers.frameDeny=true"
      - "traefik.http.middlewares.security-headers.headers.contentTypeNosniff=true"
      - "traefik.http.middlewares.security-headers.headers.browserXssFilter=true"
      - "traefik.http.middlewares.security-headers.headers.referrerPolicy=strict-origin-when-cross-origin"
      - "traefik.http.middlewares.security-headers.headers.forceSTSHeader=true"
      - "traefik.http.middlewares.security-headers.headers.stsSeconds=31536000"
      - "traefik.http.middlewares.security-headers.headers.stsIncludeSubdomains=true"
      - "traefik.http.middlewares.security-headers.headers.stsPreload=true"
      - "traefik.http.middlewares.security-headers.headers.contentSecurityPolicy=default-src 'self'"
    networks:
      - microservices-net
    restart: always
    
  # Service configurations with HTTPS
  frontend:
    build: ./frontend
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.frontend.rule=Host(`${DOMAIN}`) || Host(`www.${DOMAIN}`)"
      - "traefik.http.routers.frontend.entrypoints=websecure"
      - "traefik.http.routers.frontend.tls=true"
      - "traefik.http.routers.frontend.tls.certresolver=letsencrypt"
      - "traefik.http.routers.frontend.middlewares=security-headers,compress"
      - "traefik.http.services.frontend.loadbalancer.server.port=3000"
      
      # Compression
      - "traefik.http.middlewares.compress.compress=true"
    networks:
      - microservices-net
      
  api:
    build: ./api
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.${DOMAIN}`)"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls=true"
      - "traefik.http.routers.api.tls.certresolver=letsencrypt"
      - "traefik.http.routers.api.middlewares=security-headers,rate-limit,cors"
      - "traefik.http.services.api.loadbalancer.server.port=4000"
      
      # Rate limiting
      - "traefik.http.middlewares.rate-limit.ratelimit.average=100"
      - "traefik.http.middlewares.rate-limit.ratelimit.burst=200"
      
      # CORS
      - "traefik.http.middlewares.cors.headers.accesscontrolallowmethods=GET,OPTIONS,PUT,POST,DELETE,PATCH"
      - "traefik.http.middlewares.cors.headers.accesscontrolallowheaders=*"
      - "traefik.http.middlewares.cors.headers.accesscontrolalloworiginlist=https://${DOMAIN},https://www.${DOMAIN}"
      - "traefik.http.middlewares.cors.headers.accesscontrolmaxage=100"
      - "traefik.http.middlewares.cors.headers.addvaryheader=true"
    networks:
      - microservices-net

volumes:
  traefik-certificates:
    driver: local

networks:
  microservices-net:
    driver: bridge
```

```yaml
# traefik/dynamic/tls-config.yml
tls:
  options:
    default:
      minVersion: VersionTLS12
      cipherSuites:
        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
        - TLS_AES_128_GCM_SHA256
        - TLS_AES_256_GCM_SHA384
        - TLS_CHACHA20_POLY1305_SHA256
      curvePreferences:
        - X25519
        - CurveP521
        - CurveP384
      sniStrict: true
```

### 6.2 Secrets Management

```typescript
// lib/security/secrets-manager.ts
import { SecretsManager } from '@aws-sdk/client-secrets-manager';
import * as crypto from 'crypto';
import { Injectable } from '@nestjs/common';

@Injectable()
export class SecretsService {
  private secretsManager: SecretsManager;
  private cache: Map<string, { value: any; expiry: number }> = new Map();
  private encryptionKey: Buffer;

  constructor() {
    this.secretsManager = new SecretsManager({
      region: process.env.AWS_REGION || 'us-east-1'
    });
    
    // Derive encryption key from master key
    this.encryptionKey = crypto.scryptSync(
      process.env.MASTER_KEY || 'default-insecure-key',
      'salt',
      32
    );
  }

  // Get secret from AWS Secrets Manager
  async getSecret(secretName: string): Promise<any> {
    // Check cache first
    const cached = this.cache.get(secretName);
    if (cached && cached.expiry > Date.now()) {
      return cached.value;
    }

    try {
      const response = await this.secretsManager.getSecretValue({
        SecretId: secretName
      });

      let secretValue: any;
      if (response.SecretString) {
        secretValue = JSON.parse(response.SecretString);
      } else if (response.SecretBinary) {
        secretValue = Buffer.from(response.SecretBinary).toString('utf-8');
      }

      // Cache for 5 minutes
      this.cache.set(secretName, {
        value: secretValue,
        expiry: Date.now() + 5 * 60 * 1000
      });

      return secretValue;
    } catch (error) {
      console.error(`Error retrieving secret ${secretName}:`, error);
      throw error;
    }
  }

  // Rotate secrets
  async rotateSecret(secretName: string): Promise<void> {
    try {
      const response = await this.secretsManager.rotateSecret({
        SecretId: secretName,
        RotationRules: {
          AutomaticallyAfterDays: 30
        }
      });

      // Clear cache
      this.cache.delete(secretName);

      console.log(`Secret ${secretName} rotation initiated:`, response.VersionId);
    } catch (error) {
      console.error(`Error rotating secret ${secretName}:`, error);
      throw error;
    }
  }

  // Encrypt sensitive data
  encrypt(text: string): string {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(
      'aes-256-gcm',
      this.encryptionKey,
      iv
    );

    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');

    const authTag = cipher.getAuthTag();

    return iv.toString('hex') + ':' + authTag.toString('hex') + ':' + encrypted;
  }

  // Decrypt sensitive data
  decrypt(encryptedText: string): string {
    const parts = encryptedText.split(':');
    const iv = Buffer.from(parts[0], 'hex');
    const authTag = Buffer.from(parts[1], 'hex');
    const encrypted = parts[2];

    const decipher = crypto.createDecipheriv(
      'aes-256-gcm',
      this.encryptionKey,
      iv
    );
    decipher.setAuthTag(authTag);

    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');

    return decrypted;
  }

  // Generate secure tokens
  generateSecureToken(length: number = 32): string {
    return crypto.randomBytes(length).toString('hex');
  }

  // Hash passwords
  async hashPassword(password: string): Promise<string> {
    const salt = crypto.randomBytes(16).toString('hex');
    const hash = crypto.scryptSync(password, salt, 64).toString('hex');
    return `${salt}:${hash}`;
  }

  // Verify passwords
  async verifyPassword(password: string, hashedPassword: string): Promise<boolean> {
    const [salt, hash] = hashedPassword.split(':');
    const verifyHash = crypto.scryptSync(password, salt, 64).toString('hex');
    return hash === verifyHash;
  }
}

// Environment variables validation
export class EnvironmentConfig {
  private static requiredVars = [
    'DATABASE_URL',
    'REDIS_URL',
    'JWT_SECRET',
    'MASTER_KEY',
    'AWS_ACCESS_KEY_ID',
    'AWS_SECRET_ACCESS_KEY',
    'STRIPE_SECRET_KEY',
    'SMTP_PASSWORD'
  ];

  static validate() {
    const missing = this.requiredVars.filter(
      varName => !process.env[varName]
    );

    if (missing.length > 0) {
      throw new Error(
        `Missing required environment variables: ${missing.join(', ')}`
      );
    }

    // Validate formats
    this.validateDatabaseUrl();
    this.validateRedisUrl();
    this.validateJwtSecret();
  }

  private static validateDatabaseUrl() {
    const dbUrl = process.env.DATABASE_URL;
    if (!dbUrl?.startsWith('postgresql://')) {
      throw new Error('Invalid DATABASE_URL format');
    }
  }

  private static validateRedisUrl() {
    const redisUrl = process.env.REDIS_URL;
    if (!redisUrl?.startsWith('redis://')) {
      throw new Error('Invalid REDIS_URL format');
    }
  }

  private static validateJwtSecret() {
    const jwtSecret = process.env.JWT_SECRET;
    if (!jwtSecret || jwtSecret.length < 32) {
      throw new Error('JWT_SECRET must be at least 32 characters long');
    }
  }
}
```

```yaml
# kubernetes/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:
  database-url: "postgresql://user:pass@postgres:5432/db?sslmode=require"
  redis-password: "ultra-secure-redis-password"
  jwt-secret: "your-super-secure-jwt-secret-at-least-32-chars"
  master-key: "your-master-encryption-key"
  
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-external-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: app-secrets
    creationPolicy: Owner
  data:
  - secretKey: stripe-api-key
    remoteRef:
      key: production/stripe/api-key
  - secretKey: openai-api-key
    remoteRef:
      key: production/openai/api-key
  - secretKey: smtp-password
    remoteRef:
      key: production/smtp/password
```

### 6.3 Performance: Configuraciones Conservadoras

```typescript
// config/performance.config.ts
export const PERFORMANCE_CONFIG = {
  // Database connection pool
  database: {
    max: 20, // Start conservative
    min: 5,
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000,
    statementTimeout: 30000,
    query_timeout: 30000,
    
    // Connection retry
    retryAttempts: 3,
    retryDelay: 1000,
    
    // Performance monitoring
    log_min_duration_statement: 1000, // Log queries over 1s
    track_activity_query_size: 1024,
    
    // Memory settings
    shared_buffers: '256MB', // Start small
    effective_cache_size: '1GB',
    work_mem: '4MB',
    maintenance_work_mem: '64MB'
  },

  // Redis configuration
  redis: {
    maxRetriesPerRequest: 3,
    enableReadyCheck: true,
    maxLoadingRetryTime: 10000,
    
    // Connection pool
    connectionPool: {
      min: 2,
      max: 10
    },
    
    // Memory policy
    maxmemory: '512mb',
    maxmemoryPolicy: 'allkeys-lru',
    
    // Persistence
    save: '900 1 300 10 60 10000',
    appendonly: 'yes',
    appendfsync: 'everysec'
  },

  // LLM configuration
  llm: {
    // Request limits
    maxConcurrentRequests: 5,
    requestTimeout: 30000,
    maxRetries: 2,
    
    // Model specific
    llama: {
      num_threads: 4,
      num_gpu_layers: 0, // Start with CPU
      context_window: 2048,
      batch_size: 8,
      
      // Memory settings
      max_memory: '4GB',
      cache_size: '1GB'
    },
    
    embeddings: {
      batch_size: 32,
      max_concurrent: 3,
      cache_ttl: 86400
    }
  },

  // API rate limiting
  rateLimiting: {
    windowMs: 60 * 1000, // 1 minute
    max: 100, // Start with 100 requests per minute
    
    // Different limits per endpoint
    endpoints: {
      '/api/auth': { max: 20 },
      '/api/bookings': { max: 50 },
      '/api/search': { max: 100 },
      '/api/ai': { max: 10 }
    }
  },

  // Caching strategy
  caching: {
    // Default TTLs
    ttl: {
      static: 86400, // 1 day
      api: 300, // 5 minutes
      search: 600, // 10 minutes
      user: 1800 // 30 minutes
    },
    
    // Cache sizes
    maxSize: {
      memory: '100MB',
      redis: '1GB'
    }
  },

  // Worker processes
  workers: {
    // Start with minimal workers
    web: {
      instances: 2,
      maxMemory: '512MB',
      maxCpu: '0.5'
    },
    
    background: {
      instances: 1,
      maxMemory: '256MB',
      maxCpu: '0.25'
    }
  },

  // Monitoring thresholds
  monitoring: {
    alerts: {
      cpu: {
        warning: 70,
        critical: 90
      },
      memory: {
        warning: 80,
        critical: 95
      },
      responseTime: {
        warning: 1000, // 1s
        critical: 3000 // 3s
      },
      errorRate: {
        warning: 0.01, // 1%
        critical: 0.05 // 5%
      }
    }
  }
};

// Auto-scaling configuration
export const AUTOSCALING_CONFIG = {
  enabled: true,
  
  metrics: {
    cpu: {
      targetUtilization: 70,
      scaleUpThreshold: 80,
      scaleDownThreshold: 50
    },
    
    memory: {
      targetUtilization: 75,
      scaleUpThreshold: 85,
      scaleDownThreshold: 60
    },
    
    requestRate: {
      targetPerInstance: 1000,
      scaleUpThreshold: 1200,
      scaleDownThreshold: 500
    }
  },
  
  scaling: {
    minInstances: 1,
    maxInstances: 10,
    cooldownPeriod: 300, // 5 minutes
    
    // Scale gradually
    scaleUpStep: 1,
    scaleDownStep: 1,
    
    // Time windows
    scaleUpAfter: 60, // 1 minute of high load
    scaleDownAfter: 600 // 10 minutes of low load
  }
};
```

### 6.4 Monitoreo de Recursos

```typescript
// monitoring/resource-monitor.ts
import * as os from 'os';
import * as process from 'process';
import { Injectable } from '@nestjs/common';
import { Gauge, Counter, Histogram } from 'prom-client';

@Injectable()
export class ResourceMonitor {
  private cpuUsageGauge: Gauge;
  private memoryUsageGauge: Gauge;
  private diskUsageGauge: Gauge;
  private gcMetrics: {
    count: Counter;
    duration: Histogram;
  };

  constructor() {
    this.initializeMetrics();
    this.startMonitoring();
  }

  private initializeMetrics() {
    // CPU metrics
    this.cpuUsageGauge = new Gauge({
      name: 'process_cpu_usage_percent',
      help: 'CPU usage percentage',
      labelNames: ['type']
    });

    // Memory metrics
    this.memoryUsageGauge = new Gauge({
      name: 'process_memory_usage_bytes',
      help: 'Memory usage in bytes',
      labelNames: ['type']
    });

    // Disk metrics
    this.diskUsageGauge = new Gauge({
      name: 'disk_usage_bytes',
      help: 'Disk usage in bytes',
      labelNames: ['path', 'type']
    });

    // Garbage collection metrics
    this.gcMetrics = {
      count: new Counter({
        name: 'nodejs_gc_runs_total',
        help: 'Total number of garbage collection runs',
        labelNames: ['type']
      }),
      duration: new Histogram({
        name: 'nodejs_gc_duration_seconds',
        help: 'Garbage collection duration',
        labelNames: ['type'],
        buckets: [0.001, 0.01, 0.1, 1, 2, 5]
      })
    };
  }

  private startMonitoring() {
    // Update metrics every 10 seconds
    setInterval(() => {
      this.updateCPUMetrics();
      this.updateMemoryMetrics();
      this.updateDiskMetrics();
    }, 10000);

    // Monitor garbage collection
    this.monitorGarbageCollection();
  }

  private updateCPUMetrics() {
    const cpuUsage = process.cpuUsage();
    const totalCPU = cpuUsage.user + cpuUsage.system;
    const cpuPercent = (totalCPU / 1000000) / os.cpus().length;

    this.cpuUsageGauge.set({ type: 'user' }, cpuUsage.user / 1000000);
    this.cpuUsageGauge.set({ type: 'system' }, cpuUsage.system / 1000000);
    this.cpuUsageGauge.set({ type: 'total' }, cpuPercent);

    // System-wide CPU
    const cpus = os.cpus();
    let idle = 0;
    let total = 0;

    cpus.forEach(cpu => {
      for (const type in cpu.times) {
        total += cpu.times[type];
      }
      idle += cpu.times.idle;
    });

    const systemCpuUsage = 100 - ~~(100 * idle / total);
    this.cpuUsageGauge.set({ type: 'system_total' }, systemCpuUsage);
  }

  private updateMemoryMetrics() {
    const memUsage = process.memoryUsage();
    
    this.memoryUsageGauge.set({ type: 'rss' }, memUsage.rss);
    this.memoryUsageGauge.set({ type: 'heap_total' }, memUsage.heapTotal);
    this.memoryUsageGauge.set({ type: 'heap_used' }, memUsage.heapUsed);
    this.memoryUsageGauge.set({ type: 'external' }, memUsage.external);
    this.memoryUsageGauge.set({ type: 'array_buffers' }, memUsage.arrayBuffers);

    // System memory
    const totalMem = os.totalmem();
    const freeMem = os.freemem();
    const usedMem = totalMem - freeMem;

    this.memoryUsageGauge.set({ type: 'system_total' }, totalMem);
    this.memoryUsageGauge.set({ type: 'system_used' }, usedMem);
    this.memoryUsageGauge.set({ type: 'system_free' }, freeMem);
  }

  private updateDiskMetrics() {
    // This would need a platform-specific implementation
    // For Docker containers, you might want to check specific paths
    const paths = ['/app', '/data', '/tmp'];
    
    // Placeholder - actual implementation would use fs.statfs or similar
    paths.forEach(path => {
      this.diskUsageGauge.set({ path, type: 'used' }, 0);
      this.diskUsageGauge.set({ path, type: 'available' }, 0);
    });
  }

  private monitorGarbageCollection() {
    if (global.gc) {
      const originalGc = global.gc;
      global.gc = (...args: any[]) => {
        const start = process.hrtime.bigint();
        originalGc.apply(global, args);
        const end = process.hrtime.bigint();
        
        const duration = Number(end - start) / 1e9;
        this.gcMetrics.count.inc({ type: 'manual' });
        this.gcMetrics.duration.observe({ type: 'manual' }, duration);
      };
    }

    // Try to use perf_hooks for automatic GC monitoring
    try {
      const { PerformanceObserver } = require('perf_hooks');
      
      const obs = new PerformanceObserver((list: any) => {
        const entries = list.getEntries();
        entries.forEach((entry: any) => {
          if (entry.entryType === 'gc') {
            this.gcMetrics.count.inc({ type: entry.kind });
            this.gcMetrics.duration.observe(
              { type: entry.kind },
              entry.duration / 1000
            );
          }
        });
      });
      
      obs.observe({ entryTypes: ['gc'] });
    } catch (e) {
      console.log('GC monitoring not available');
    }
  }

  // Get current resource usage
  getResourceUsage() {
    const cpuUsage = process.cpuUsage();
    const memUsage = process.memoryUsage();
    
    return {
      cpu: {
        user: cpuUsage.user / 1000000,
        system: cpuUsage.system / 1000000,
        percent: ((cpuUsage.user + cpuUsage.system) / 1000000) / os.cpus().length
      },
      memory: {
        rss: this.formatBytes(memUsage.rss),
        heapTotal: this.formatBytes(memUsage.heapTotal),
        heapUsed: this.formatBytes(memUsage.heapUsed),
        external: this.formatBytes(memUsage.external),
        percent: (memUsage.rss / os.totalmem()) * 100
      },
      system: {
        loadAverage: os.loadavg(),
        uptime: os.uptime(),
        cpuCount: os.cpus().length,
        totalMemory: this.formatBytes(os.totalmem()),
        freeMemory: this.formatBytes(os.freemem())
      }
    };
  }

  private formatBytes(bytes: number): string {
    const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB'];
    if (bytes === 0) return '0 Bytes';
    const i = Math.floor(Math.log(bytes) / Math.log(1024));
    return Math.round(bytes / Math.pow(1024, i) * 100) / 100 + ' ' + sizes[i];
  }

  // Check if resources are within limits
  checkResourceLimits(): { healthy: boolean; warnings: string[] } {
    const warnings: string[] = [];
    const usage = this.getResourceUsage();
    const config = PERFORMANCE_CONFIG.monitoring.alerts;

    // Check CPU
    if (usage.cpu.percent > config.cpu.critical) {
      warnings.push(`CRITICAL: CPU usage at ${usage.cpu.percent.toFixed(2)}%`);
    } else if (usage.cpu.percent > config.cpu.warning) {
      warnings.push(`WARNING: CPU usage at ${usage.cpu.percent.toFixed(2)}%`);
    }

    // Check Memory
    if (usage.memory.percent > config.memory.critical) {
      warnings.push(`CRITICAL: Memory usage at ${usage.memory.percent.toFixed(2)}%`);
    } else if (usage.memory.percent > config.memory.warning) {
      