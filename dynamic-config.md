# Dynamic Runtime Configuration for Docker Applications

## Overview

This document describes a generic approach for implementing dynamic runtime configuration in containerized applications. Instead of baking configuration values into the application at build time, this pattern allows environment variables to be injected when the Docker container starts, enabling the same image to be deployed across different environments without rebuilding.

## The Problem

Traditional approaches to configuration in containerized applications often involve:
- Building separate images for each environment
- Hardcoding environment variables at build time
- Managing multiple CI/CD pipelines for different deployments
- Rebuilding applications when only configuration changes

These approaches lead to:
- Increased build times and storage requirements
- Complex deployment pipelines
- Potential security risks from embedded credentials
- Difficulty in quickly updating configuration

## The Solution: Runtime Configuration Pattern

This pattern uses a web server (nginx, Apache, etc.) to dynamically serve configuration values as JSON at runtime, which the application fetches when it loads.

## Architecture Components

### 1. Client-Side Configuration Loader

Create a configuration loader in your application that:
- Fetches configuration from a known endpoint (e.g., `/config.json`)
- Caches the configuration to avoid repeated requests
- Provides a fallback mechanism for development/build-time values
- Handles errors gracefully

#### Example Implementation (JavaScript/TypeScript):

```typescript
// configLoader.ts
interface RuntimeConfig {
  apiUrl: string;
  analyticsKey?: string;
  featureFlags?: Record<string, boolean>;
}

let cachedConfig: RuntimeConfig | null = null;

export async function loadRuntimeConfig(): Promise<RuntimeConfig> {
  if (cachedConfig) {
    return cachedConfig;
  }

  try {
    const response = await fetch(`${window.location.origin}/config.json`);
    if (response.ok) {
      const config = await response.json();
      cachedConfig = config;
      return config;
    }
  } catch (error) {
    console.error('Failed to load runtime config:', error);
  }

  // Fallback to build-time values
  cachedConfig = {
    apiUrl: process.env.REACT_APP_API_URL || 'http://localhost:3000',
    analyticsKey: process.env.REACT_APP_ANALYTICS_KEY,
    featureFlags: {}
  };
  
  return cachedConfig;
}
```

### 2. Web Server Configuration Template

Create a template for your web server that will be processed at container startup:

#### Nginx Example (`nginx.conf.template`):

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    
    # Serve runtime configuration
    location /config.json {
        default_type application/json;
        add_header Cache-Control "no-cache, no-store, must-revalidate";
        return 200 '{
            "apiUrl": "${API_URL}",
            "analyticsKey": "${ANALYTICS_KEY}",
            "featureFlags": {
                "newFeature": ${FEATURE_NEW_ENABLED},
                "betaMode": ${FEATURE_BETA_MODE}
            }
        }';
    }
    
    # Serve application files
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

#### Apache Example (`.htaccess.template` or `httpd.conf.template`):

```apache
<Location /config.json>
    Header set Content-Type "application/json"
    Header set Cache-Control "no-cache, no-store, must-revalidate"
    
    # Using mod_rewrite to return JSON
    RewriteEngine On
    RewriteRule ^.*$ - [R=200,L]
    
    # Set the response body
    ErrorDocument 200 '{
        "apiUrl": "${API_URL}",
        "analyticsKey": "${ANALYTICS_KEY}"
    }'
</Location>
```

### 3. Dockerfile Implementation

Create a Dockerfile that supports runtime configuration:

```dockerfile
# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Runtime stage
FROM nginx:alpine

# Install envsubst (included in gettext)
RUN apk add --no-cache gettext

# Copy built application
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy nginx template
COPY nginx.conf.template /etc/nginx/templates/default.conf.template

# Set default values for environment variables
ENV API_URL=http://localhost:3000
ENV ANALYTICS_KEY=""
ENV FEATURE_NEW_ENABLED=false
ENV FEATURE_BETA_MODE=false

# Process template and start nginx
CMD sh -c "envsubst '\$API_URL \$ANALYTICS_KEY \$FEATURE_NEW_ENABLED \$FEATURE_BETA_MODE' < /etc/nginx/templates/default.conf.template > /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'"
```

## How It Works

### 1. Build Time
- Application is built once with default/development configuration
- No environment-specific values are hardcoded
- The build artifact is environment-agnostic

### 2. Container Startup
1. Environment variables are passed to the container via `-e` flags or orchestration tools
2. `envsubst` replaces placeholders in the web server template with actual values
3. The processed configuration is written to the web server's config directory
4. Web server starts and serves both the application and configuration endpoint

### 3. Application Runtime
1. When the application loads, it fetches `/config.json`
2. The web server returns dynamically generated JSON with current environment values
3. Application uses this configuration for all runtime decisions
4. If fetch fails, application falls back to build-time defaults

## Implementation Examples

### React Application

```javascript
// App.jsx
import { useEffect, useState } from 'react';
import { loadRuntimeConfig } from './configLoader';

function App() {
  const [config, setConfig] = useState(null);

  useEffect(() => {
    loadRuntimeConfig().then(setConfig);
  }, []);

  if (!config) return <div>Loading configuration...</div>;

  return <YourApp config={config} />;
}
```

### Vue.js Application

```javascript
// main.js
import { createApp } from 'vue';
import App from './App.vue';
import { loadRuntimeConfig } from './configLoader';

loadRuntimeConfig().then(config => {
  const app = createApp(App);
  app.config.globalProperties.$config = config;
  app.mount('#app');
});
```

### Angular Application

```typescript
// app.config.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class ConfigService {
  private config: any;

  constructor(private http: HttpClient) {}

  async loadConfig(): Promise<any> {
    try {
      this.config = await this.http.get('/config.json').toPromise();
    } catch {
      this.config = { apiUrl: 'http://localhost:3000' }; // fallback
    }
    return this.config;
  }

  get(key: string): any {
    return this.config?.[key];
  }
}

// app.module.ts - Initialize before app starts
export function initializeApp(configService: ConfigService) {
  return (): Promise<any> => configService.loadConfig();
}
```

## Framework-Specific Considerations

### Single-Page Applications (SPA)
- Fetch configuration once during app initialization
- Cache configuration in memory or state management
- Consider loading screens while configuration loads

### Server-Side Rendering (SSR)
- Load configuration on the server during request handling
- Pass configuration to client via window object or hydration
- Ensure configuration endpoint is accessible from both server and client

### Static Site Generators
- Generate a configuration loading script during build
- Inject configuration at runtime via script tag
- Consider using service workers for offline support

## Deployment Examples

### Docker Compose

```yaml
version: '3.8'
services:
  frontend:
    image: myapp:latest
    ports:
      - "80:80"
    environment:
      - API_URL=http://backend:3000
      - ANALYTICS_KEY=UA-123456
      - FEATURE_NEW_ENABLED=true
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: API_URL
          value: "https://api.production.com"
        - name: ANALYTICS_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: analytics-key
```

### Docker Swarm

```bash
docker service create \
  --name frontend \
  --publish 80:80 \
  --env API_URL=https://api.production.com \
  --env FEATURE_BETA_MODE=true \
  myapp:latest
```

## Best Practices

### 1. Security
- Never expose sensitive credentials in `/config.json`
- Use separate mechanisms for secrets (environment variables on backend)
- Implement proper CORS policies
- Consider encrypting sensitive configuration values

### 2. Performance
- Cache configuration after first load
- Set appropriate cache headers (no-cache for config endpoint)
- Consider using CDN for static assets but exclude `/config.json`
- Implement retry logic for configuration loading

### 3. Reliability
- Always provide sensible defaults
- Implement graceful degradation
- Log configuration loading failures
- Consider health check endpoints that verify configuration

### 4. Development Experience
- Use same configuration mechanism in development
- Provide clear error messages for missing configuration
- Document all configuration options
- Create configuration templates for different environments

## Testing Strategy

### Unit Tests
```javascript
// configLoader.test.js
describe('configLoader', () => {
  it('should load configuration from endpoint', async () => {
    global.fetch = jest.fn(() =>
      Promise.resolve({
        ok: true,
        json: () => Promise.resolve({ apiUrl: 'http://test.com' })
      })
    );
    
    const config = await loadRuntimeConfig();
    expect(config.apiUrl).toBe('http://test.com');
  });

  it('should fallback when endpoint fails', async () => {
    global.fetch = jest.fn(() => Promise.reject('Network error'));
    
    const config = await loadRuntimeConfig();
    expect(config.apiUrl).toBe('http://localhost:3000');
  });
});
```

### Integration Tests
```bash
#!/bin/bash
# test-config.sh
docker run -d -e API_URL=http://test-api.com -p 8080:80 myapp:latest
sleep 5
response=$(curl -s http://localhost:8080/config.json)
if [[ $response == *"http://test-api.com"* ]]; then
  echo "Configuration test passed"
else
  echo "Configuration test failed"
  exit 1
fi
```

## Troubleshooting Guide

### Common Issues and Solutions

1. **Configuration not updating**
   - Clear browser cache and try hard refresh
   - Verify environment variables are set correctly in container
   - Check web server logs for template processing errors

2. **CORS errors when fetching config**
   - Ensure config endpoint is served from same origin
   - Add appropriate CORS headers if needed
   - Check browser console for specific error messages

3. **Template substitution not working**
   - Verify `envsubst` or equivalent is installed
   - Check syntax of variable placeholders
   - Ensure variables are exported in shell environment

4. **Application starts before config loads**
   - Implement proper loading states
   - Use promises or async/await properly
   - Consider synchronous loading for critical config

### Debug Commands

```bash
# Verify environment variables in container
docker exec <container> env | grep -E "API_URL|ANALYTICS"

# Check processed web server config
docker exec <container> cat /etc/nginx/conf.d/default.conf

# Test configuration endpoint
curl -H "Cache-Control: no-cache" http://localhost:8080/config.json

# View web server error logs
docker logs <container> 2>&1 | grep -i error
```

## Migration Guide

### From Build-Time to Runtime Configuration

1. **Identify all environment-specific values**
   - API endpoints
   - Feature flags
   - Third-party service keys
   - Environment names

2. **Update build process**
   - Remove environment-specific build steps
   - Create single build artifact
   - Add configuration loader to application

3. **Update deployment**
   - Add environment variables to deployment scripts
   - Update CI/CD pipelines
   - Test in staging environment

4. **Gradual rollout**
   - Start with non-critical configuration
   - Monitor for issues
   - Gradually move all configuration to runtime

## Conclusion

This runtime configuration pattern provides a flexible, secure, and maintainable approach to managing application configuration in containerized environments. By separating build-time and runtime concerns, teams can deploy the same artifact across all environments while maintaining environment-specific behavior through configuration.