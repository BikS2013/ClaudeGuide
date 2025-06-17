# Complete CORS Implementation Guide

## Overview

Cross-Origin Resource Sharing (CORS) is a security feature implemented by web browsers that blocks web pages from making requests to a different domain than the one serving the web page. This guide provides a comprehensive approach to implementing, testing, and debugging CORS in any backend API application.

## Understanding CORS

### What Triggers CORS?
CORS is triggered when a web application makes a request where any of these differ:
- **Protocol**: `http://` vs `https://`
- **Domain**: `example.com` vs `another.com`
- **Port**: `:3000` vs `:3001`
- **Subdomain**: `app.example.com` vs `api.example.com`

### Types of CORS Requests
1. **Simple Requests**: GET, POST (with certain content types), HEAD
2. **Preflight Requests**: PUT, DELETE, or requests with custom headers

## Implementation Strategies

### 1. Express.js / Node.js

#### Basic Implementation
```javascript
const cors = require('cors');
const express = require('express');
const app = express();

// Simple CORS - Allow all origins (NOT for production!)
app.use(cors());
```

#### Production-Ready Implementation
```javascript
const corsOptions = {
  origin: function (origin, callback) {
    const allowedOrigins = process.env.CORS_ORIGINS?.split(',') || ['http://localhost:3000'];
    
    // Allow requests with no origin (mobile apps, Postman, etc.)
    if (!origin) return callback(null, true);
    
    if (allowedOrigins.indexOf(origin) !== -1 || allowedOrigins.includes('*')) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Requested-With'],
  exposedHeaders: ['X-Total-Count', 'X-Page-Number'],
  maxAge: 86400 // 24 hours
};

app.use(cors(corsOptions));
```

#### Manual CORS Headers (without cors package)
```javascript
app.use((req, res, next) => {
  const allowedOrigins = process.env.CORS_ORIGINS?.split(',') || ['http://localhost:3000'];
  const origin = req.headers.origin;
  
  if (allowedOrigins.includes(origin) || allowedOrigins.includes('*')) {
    res.setHeader('Access-Control-Allow-Origin', origin || '*');
  }
  
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, PATCH, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  res.setHeader('Access-Control-Allow-Credentials', 'true');
  res.setHeader('Access-Control-Max-Age', '86400');
  
  if (req.method === 'OPTIONS') {
    res.sendStatus(204);
  } else {
    next();
  }
});
```

### 2. Python (Flask)

```python
from flask import Flask, jsonify
from flask_cors import CORS
import os

app = Flask(__name__)

# Configure CORS
cors_origins = os.environ.get('CORS_ORIGINS', 'http://localhost:3000').split(',')

CORS(app, 
     origins=cors_origins,
     supports_credentials=True,
     methods=['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
     allow_headers=['Content-Type', 'Authorization'],
     expose_headers=['X-Total-Count', 'X-Page-Number'],
     max_age=86400)

# Or use decorator for specific routes
from flask_cors import cross_origin

@app.route('/api/data')
@cross_origin(origins=['http://localhost:3000'], supports_credentials=True)
def get_data():
    return jsonify({'data': 'example'})
```

### 3. Python (FastAPI)

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import os

app = FastAPI()

origins = os.environ.get('CORS_ORIGINS', 'http://localhost:3000').split(',')

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
    allow_headers=["Content-Type", "Authorization"],
    expose_headers=["X-Total-Count", "X-Page-Number"],
    max_age=86400,
)
```

### 4. Java (Spring Boot)

```java
@Configuration
public class CorsConfiguration {
    
    @Value("${cors.origins}")
    private String corsOrigins;
    
    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                String[] origins = corsOrigins.split(",");
                
                registry.addMapping("/api/**")
                    .allowedOrigins(origins)
                    .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
                    .allowedHeaders("Content-Type", "Authorization")
                    .exposedHeaders("X-Total-Count", "X-Page-Number")
                    .allowCredentials(true)
                    .maxAge(86400);
            }
        };
    }
}
```

### 5. .NET Core

```csharp
// In Program.cs or Startup.cs
var corsOrigins = Configuration["CorsOrigins"].Split(',');

builder.Services.AddCors(options =>
{
    options.AddPolicy("ApiCorsPolicy", builder =>
    {
        builder.WithOrigins(corsOrigins)
               .AllowAnyMethod()
               .AllowAnyHeader()
               .AllowCredentials()
               .WithExposedHeaders("X-Total-Count", "X-Page-Number")
               .SetPreflightMaxAge(TimeSpan.FromSeconds(86400));
    });
});

// Apply the policy
app.UseCors("ApiCorsPolicy");
```

### 6. Ruby on Rails

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins ENV['CORS_ORIGINS']&.split(',') || 'http://localhost:3000'
    
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: true,
      expose: ['X-Total-Count', 'X-Page-Number'],
      max_age: 86400
  end
end
```

## Environment Configuration

### Development (.env.development)
```bash
# Allow common development origins
CORS_ORIGINS=http://localhost:3000,http://localhost:3001,http://127.0.0.1:3000

# Or allow all origins in development (careful!)
CORS_ORIGINS=*
```

### Production (.env.production)
```bash
# Only allow specific production origins
CORS_ORIGINS=https://app.example.com,https://www.example.com

# Never use * in production!
```

### Docker Configuration
```yaml
# docker-compose.yml
services:
  api:
    environment:
      - CORS_ORIGINS=http://frontend:80,https://app.example.com
  
  frontend:
    environment:
      - API_URL=http://api:3000
```

## Testing CORS

### 1. Browser DevTools Method
```javascript
// Open browser console on any website and run:
fetch('http://localhost:3001/api/health', {
  method: 'GET',
  credentials: 'include',
  headers: {
    'Content-Type': 'application/json'
  }
})
.then(res => res.json())
.then(data => console.log('Success:', data))
.catch(err => console.error('CORS Error:', err));
```

### 2. cURL Testing
```bash
# Test simple request
curl -H "Origin: http://localhost:3000" \
     -H "Access-Control-Request-Method: GET" \
     -H "Access-Control-Request-Headers: X-Requested-With" \
     -X OPTIONS \
     -v \
     http://localhost:3001/api/health

# Test preflight request
curl -H "Origin: http://localhost:3000" \
     -H "Access-Control-Request-Method: POST" \
     -H "Access-Control-Request-Headers: Content-Type, Authorization" \
     -X OPTIONS \
     -v \
     http://localhost:3001/api/users

# Test actual request with credentials
curl -H "Origin: http://localhost:3000" \
     -H "Content-Type: application/json" \
     -H "Cookie: session=abc123" \
     -X POST \
     -d '{"name":"test"}' \
     -v \
     http://localhost:3001/api/users
```

### 3. Automated Testing Script
```javascript
// cors-test.js
const testCors = async (apiUrl, origin) => {
  console.log(`Testing CORS for ${apiUrl} from origin ${origin}`);
  
  try {
    // Test preflight
    const preflightResponse = await fetch(apiUrl, {
      method: 'OPTIONS',
      headers: {
        'Origin': origin,
        'Access-Control-Request-Method': 'POST',
        'Access-Control-Request-Headers': 'Content-Type'
      }
    });
    
    console.log('Preflight Status:', preflightResponse.status);
    console.log('CORS Headers:');
    console.log('- Allow-Origin:', preflightResponse.headers.get('Access-Control-Allow-Origin'));
    console.log('- Allow-Methods:', preflightResponse.headers.get('Access-Control-Allow-Methods'));
    console.log('- Allow-Headers:', preflightResponse.headers.get('Access-Control-Allow-Headers'));
    console.log('- Allow-Credentials:', preflightResponse.headers.get('Access-Control-Allow-Credentials'));
    
    // Test actual request
    const actualResponse = await fetch(apiUrl, {
      method: 'GET',
      headers: {
        'Origin': origin,
        'Content-Type': 'application/json'
      },
      credentials: 'include'
    });
    
    console.log('\nActual Request Status:', actualResponse.status);
    
  } catch (error) {
    console.error('CORS Test Failed:', error.message);
  }
};

// Run tests
testCors('http://localhost:3001/api/health', 'http://localhost:3000');
```

## Client-Side Debugging

### 1. Check Network Tab
```javascript
// What to look for in DevTools Network tab:
// 1. OPTIONS request (preflight) - should return 204 or 200
// 2. Actual request - check response headers
// 3. Look for Access-Control-* headers in response
```

### 2. Common Client Errors and Solutions

#### Error: "CORS policy: No 'Access-Control-Allow-Origin' header"
```javascript
// ❌ Wrong
fetch('http://api.example.com/data')

// ✅ Correct - Include credentials if needed
fetch('http://api.example.com/data', {
  credentials: 'include'  // or 'same-origin' or 'omit'
})
```

#### Error: "CORS policy: Cannot use wildcard in 'Access-Control-Allow-Origin' when credentials flag is true"
```javascript
// Backend fix: Use specific origin instead of *
// Frontend fix: Remove credentials if not needed
fetch('http://api.example.com/data', {
  credentials: 'omit'  // Don't send cookies
})
```

#### Error: "CORS policy: Request header field content-type is not allowed"
```javascript
// Ensure backend allows Content-Type header
// Or use simple content type:
fetch('http://api.example.com/data', {
  method: 'POST',
  headers: {
    'Content-Type': 'text/plain'  // Simple CORS
  },
  body: JSON.stringify(data)
})
```

### 3. Debug Helper Function
```javascript
// cors-debugger.js - Include in your frontend for debugging
window.corsDebugger = {
  test: async (url, options = {}) => {
    console.group(`🔍 CORS Debug: ${url}`);
    
    try {
      const response = await fetch(url, {
        ...options,
        mode: 'cors'
      });
      
      console.log('✅ Request succeeded');
      console.log('Status:', response.status);
      console.log('Headers:', Object.fromEntries(response.headers.entries()));
      
      // Check for CORS headers
      const corsHeaders = [
        'access-control-allow-origin',
        'access-control-allow-credentials',
        'access-control-expose-headers'
      ];
      
      corsHeaders.forEach(header => {
        const value = response.headers.get(header);
        if (value) {
          console.log(`✅ ${header}: ${value}`);
        } else {
          console.warn(`⚠️ Missing ${header}`);
        }
      });
      
    } catch (error) {
      console.error('❌ Request failed:', error.message);
      console.log('Possible issues:');
      console.log('- Server doesn\'t allow origin:', window.location.origin);
      console.log('- Preflight request failed (check OPTIONS request)');
      console.log('- Network error or server is down');
    }
    
    console.groupEnd();
  },
  
  testPreflight: async (url, method = 'POST', headers = {}) => {
    console.group(`🔍 Preflight Debug: ${method} ${url}`);
    
    try {
      const response = await fetch(url, {
        method: 'OPTIONS',
        headers: {
          'Origin': window.location.origin,
          'Access-Control-Request-Method': method,
          'Access-Control-Request-Headers': Object.keys(headers).join(', ')
        }
      });
      
      console.log('Preflight Status:', response.status);
      console.log('Allowed Methods:', response.headers.get('access-control-allow-methods'));
      console.log('Allowed Headers:', response.headers.get('access-control-allow-headers'));
      console.log('Max Age:', response.headers.get('access-control-max-age'));
      
    } catch (error) {
      console.error('❌ Preflight failed:', error);
    }
    
    console.groupEnd();
  }
};

// Usage:
// corsDebugger.test('http://localhost:3001/api/health');
// corsDebugger.testPreflight('http://localhost:3001/api/users', 'POST', {'Content-Type': 'application/json'});
```

## Production Best Practices

### 1. Security Configuration
```javascript
// ✅ Production-ready CORS configuration
const corsOptions = {
  origin: function (origin, callback) {
    // Whitelist of allowed origins
    const whitelist = [
      'https://app.example.com',
      'https://www.example.com',
      'https://mobile.example.com'
    ];
    
    // Allow server-to-server requests (no origin)
    if (!origin && process.env.ALLOW_NO_ORIGIN === 'true') {
      return callback(null, true);
    }
    
    if (whitelist.indexOf(origin) !== -1) {
      callback(null, true);
    } else {
      callback(new Error(`Origin ${origin} not allowed by CORS`));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  exposedHeaders: ['X-Total-Count'],
  maxAge: 3600, // 1 hour cache for preflight
  optionsSuccessStatus: 204
};
```

### 2. Monitoring and Logging
```javascript
// Log CORS errors for monitoring
app.use((err, req, res, next) => {
  if (err && err.message && err.message.includes('CORS')) {
    console.error('CORS Error:', {
      origin: req.headers.origin,
      method: req.method,
      path: req.path,
      error: err.message,
      timestamp: new Date().toISOString()
    });
    
    res.status(403).json({
      error: 'CORS policy violation',
      message: 'Origin not allowed'
    });
  } else {
    next(err);
  }
});
```

### 3. Dynamic Origin Validation
```javascript
// For multi-tenant applications
const corsOptions = {
  origin: async function (origin, callback) {
    try {
      // Check origin against database
      const isAllowed = await checkOriginInDatabase(origin);
      callback(null, isAllowed);
    } catch (error) {
      callback(error);
    }
  }
};
```

### 4. Environment-Specific Configuration
```javascript
const getCorsConfig = () => {
  const env = process.env.NODE_ENV || 'development';
  
  switch (env) {
    case 'production':
      return {
        origin: process.env.CORS_ORIGINS.split(','),
        credentials: true,
        maxAge: 86400 // 24 hours
      };
    
    case 'staging':
      return {
        origin: [
          /\.staging\.example\.com$/,  // Regex for subdomains
          'https://staging.example.com'
        ],
        credentials: true,
        maxAge: 3600 // 1 hour
      };
    
    case 'development':
      return {
        origin: true, // Allow all origins
        credentials: true,
        maxAge: 600 // 10 minutes
      };
    
    default:
      throw new Error(`Unknown environment: ${env}`);
  }
};

app.use(cors(getCorsConfig()));
```

## Troubleshooting Checklist

### Backend Checklist
- [ ] CORS middleware is applied before route handlers
- [ ] Origins include protocol (http:// or https://)
- [ ] Credentials flag matches frontend expectations
- [ ] OPTIONS requests are handled (preflight)
- [ ] Required headers are in allowedHeaders
- [ ] Response headers are in exposedHeaders
- [ ] Error handling doesn't break CORS headers

### Frontend Checklist
- [ ] Using correct URL with protocol
- [ ] Credentials mode matches backend configuration
- [ ] Not sending unnecessary custom headers
- [ ] Content-Type is appropriate for request
- [ ] Not mixing HTTP and HTTPS in production

### Infrastructure Checklist
- [ ] Load balancer preserves Origin header
- [ ] Reverse proxy forwards CORS headers
- [ ] CDN doesn't cache CORS headers incorrectly
- [ ] SSL certificates are valid (for HTTPS)
- [ ] Firewall allows OPTIONS requests

## Quick Reference

### CORS Headers Explained
```
Access-Control-Allow-Origin: Specifies allowed origins
Access-Control-Allow-Methods: HTTP methods allowed
Access-Control-Allow-Headers: Request headers allowed
Access-Control-Expose-Headers: Response headers exposed to frontend
Access-Control-Allow-Credentials: Whether to include cookies/auth
Access-Control-Max-Age: How long to cache preflight response
```

### Common Patterns
```javascript
// 1. Public API (no auth)
cors({ origin: '*' })

// 2. Private API (with auth)
cors({ origin: 'https://app.example.com', credentials: true })

// 3. Multiple clients
cors({ origin: ['https://app.example.com', 'https://mobile.example.com'] })

// 4. Development + Production
cors({ origin: process.env.NODE_ENV === 'production' ? 'https://app.example.com' : true })

// 5. Subdomain wildcard
cors({ origin: /^https:\/\/.*\.example\.com$/ })
```

## Conclusion

Proper CORS configuration is crucial for API security and functionality. Use this guide to implement CORS correctly from the start, saving hours of debugging time. Remember: be permissive in development, but restrictive in production.