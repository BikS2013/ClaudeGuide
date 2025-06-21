# Environment Configuration Pattern

## Overview

This document describes the design and implementation pattern for providing client applications with environment-specific configuration parameters retrieved from a centralized configuration repository. This pattern enables applications to seamlessly operate across different environments (development, staging, production) with appropriate settings for each environment without requiring code changes or redeployment.

## Purpose

The environment configuration pattern provides:

- **Environment Isolation**: Separate configurations for development, staging, and production environments
- **Dynamic Environment Switching**: Applications retrieve environment-specific settings at startup
- **Environment-Specific Features**: Enable/disable features based on the deployment environment
- **Centralized Management**: Manage all environment configurations from a single repository
- **Secure Defaults**: Environment-appropriate security settings and API endpoints
- **Deployment Flexibility**: Same codebase operates correctly in any environment

## Architecture

### System Components

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Client Apps    │────▶│  Configuration   │────▶│  GitHub Config  │
│  (React/Vue)    │     │  API Endpoint    │     │   Repository    │
│  [Environment]  │     │ /api/config/env  │     │  /env-configs   │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                               │                           │
                               ▼                           ▼
                        ┌──────────────────┐     ┌─────────────────┐
                        │  Memory Cache    │     │  Environment    │
                        │  (Per-Env)       │     │  Directories    │
                        └──────────────────┘     └─────────────────┘
```

### Data Flow

1. **Environment Detection**: Client determines its runtime environment (dev/staging/prod)
2. **Configuration Request**: Client requests environment-specific configuration
3. **Environment Resolution**: Server identifies requested environment from client context
4. **Cache Check**: Server checks environment-specific cache entry
5. **Repository Fetch**: Fetch from environment-specific path in GitHub repository
6. **Environment Merge**: Merge base configuration with environment overrides
7. **Response Delivery**: Return environment-tailored configuration
8. **Fallback Strategy**: Use environment-appropriate defaults if fetch fails

## Implementation Details

### API Endpoint Design

#### Endpoint Specification
```
GET /api/configuration/environment
```

- **Authentication**: None (unprotected endpoint)
- **Environment Detection**: Auto-detect from request origin or accept query parameter
- **Rate Limiting**: Implemented per environment to prevent abuse
- **Caching**: Environment-specific server-side cache with configurable TTL
- **Response Format**: JSON
- **Content-Type**: `application/json`
- **Query Parameters**: 
  - `env` (optional): Override environment detection (dev|staging|prod)
  - `version` (optional): Request specific configuration version

#### Response Structure
```json
{
  "environment": "production",
  "version": "1.0.0",
  "timestamp": "2025-01-21T10:30:00Z",
  "features": {
    "analytics": {
      "enabled": true,
      "providers": ["google", "segment"],
      "debugMode": false
    },
    "darkMode": {
      "enabled": true,
      "default": "auto"
    },
    "betaFeatures": {
      "enabled": false,
      "allowedUsers": []
    },
    "debugging": {
      "enabled": false,
      "logLevel": "error",
      "consoleOutput": false
    }
  },
  "ui": {
    "theme": {
      "primaryColor": "#007bff",
      "secondaryColor": "#6c757d",
      "fontFamily": "Inter, sans-serif"
    },
    "layout": {
      "sidebarCollapsed": false,
      "compactMode": false,
      "showDevTools": false
    }
  },
  "api": {
    "baseUrl": "https://api.production.example.com",
    "endpoints": {
      "analytics": "/v1/analytics",
      "feedback": "/v1/feedback",
      "auth": "/v1/auth"
    },
    "timeout": 30000,
    "retryAttempts": 3,
    "rateLimiting": {
      "enabled": true,
      "maxRequests": 1000,
      "windowMs": 60000
    }
  },
  "security": {
    "enableHttps": true,
    "csrfProtection": true,
    "allowedOrigins": ["https://app.example.com"],
    "cookieSettings": {
      "secure": true,
      "sameSite": "strict",
      "httpOnly": true
    }
  },
  "monitoring": {
    "errorReporting": {
      "enabled": true,
      "sampleRate": 1.0,
      "endpoint": "https://errors.example.com"
    },
    "performance": {
      "enabled": true,
      "sampleRate": 0.1
    }
  },
  "maintenance": {
    "enabled": false,
    "message": "",
    "allowedIPs": [],
    "scheduledDowntime": []
  }
}
```

### Server-Side Implementation

#### Configuration Service
```typescript
interface EnvironmentConfiguration {
  environment: 'development' | 'staging' | 'production';
  version: string;
  timestamp: string;
  features: FeatureConfiguration;
  ui: UIConfiguration;
  api: APIConfiguration;
  security: SecurityConfiguration;
  monitoring: MonitoringConfiguration;
  maintenance: MaintenanceConfiguration;
}

interface EnvironmentConfigurationService {
  getEnvironmentConfiguration(env?: string): Promise<EnvironmentConfiguration>;
  detectEnvironment(request: Request): string;
  clearCache(environment?: string): void;
  setEnvironmentDefaults(env: string, config: EnvironmentConfiguration): void;
}
```

#### Memory Cache Implementation
```typescript
class EnvironmentConfigurationCache {
  private cache: Map<string, CacheEntry> = new Map();
  private environmentTTL: Record<string, number> = {
    development: 60,     // 1 minute - frequent changes
    staging: 300,        // 5 minutes - moderate changes
    production: 900      // 15 minutes - stable
  };

  async get(environment: string): Promise<any | null> {
    const key = `config:${environment}`;
    const entry = this.cache.get(key);
    if (!entry) return null;
    
    if (Date.now() > entry.expiry) {
      this.cache.delete(key);
      return null;
    }
    
    return entry.value;
  }

  set(environment: string, value: any): void {
    const key = `config:${environment}`;
    const ttl = this.environmentTTL[environment] || 300;
    
    this.cache.set(key, {
      value,
      expiry: Date.now() + (ttl * 1000),
      environment
    });
  }

  clearEnvironment(environment: string): void {
    const key = `config:${environment}`;
    this.cache.delete(key);
  }
}
```

#### Environment-Specific Default Configurations
```typescript
const ENVIRONMENT_DEFAULTS: Record<string, EnvironmentConfiguration> = {
  development: {
    environment: 'development',
    version: "1.0.0",
    timestamp: new Date().toISOString(),
    features: {
      analytics: { enabled: false, providers: [], debugMode: true },
      darkMode: { enabled: true, default: "auto" },
      betaFeatures: { enabled: true, allowedUsers: [] },
      debugging: { enabled: true, logLevel: "debug", consoleOutput: true }
    },
    ui: {
      theme: {
        primaryColor: "#007bff",
        secondaryColor: "#6c757d",
        fontFamily: "system-ui, sans-serif"
      },
      layout: {
        sidebarCollapsed: false,
        compactMode: false,
        showDevTools: true
      }
    },
    api: {
      baseUrl: "http://localhost:3001",
      endpoints: {
        analytics: "/v1/analytics",
        feedback: "/v1/feedback",
        auth: "/v1/auth"
      },
      timeout: 60000,
      retryAttempts: 1,
      rateLimiting: { enabled: false }
    },
    security: {
      enableHttps: false,
      csrfProtection: false,
      allowedOrigins: ["http://localhost:3000", "http://localhost:5173"],
      cookieSettings: {
        secure: false,
        sameSite: "lax",
        httpOnly: true
      }
    },
    monitoring: {
      errorReporting: { enabled: true, sampleRate: 1.0, endpoint: "console" },
      performance: { enabled: false }
    },
    maintenance: { enabled: false, message: "", allowedIPs: [] }
  },
  
  staging: {
    environment: 'staging',
    version: "1.0.0",
    timestamp: new Date().toISOString(),
    features: {
      analytics: { enabled: true, providers: ["google"], debugMode: true },
      darkMode: { enabled: true, default: "auto" },
      betaFeatures: { enabled: true, allowedUsers: ["beta-testers"] },
      debugging: { enabled: true, logLevel: "warn", consoleOutput: false }
    },
    ui: {
      theme: {
        primaryColor: "#007bff",
        secondaryColor: "#6c757d",
        fontFamily: "Inter, sans-serif"
      },
      layout: {
        sidebarCollapsed: false,
        compactMode: false,
        showDevTools: false
      }
    },
    api: {
      baseUrl: "https://api.staging.example.com",
      endpoints: {
        analytics: "/v1/analytics",
        feedback: "/v1/feedback",
        auth: "/v1/auth"
      },
      timeout: 30000,
      retryAttempts: 2,
      rateLimiting: {
        enabled: true,
        maxRequests: 5000,
        windowMs: 60000
      }
    },
    security: {
      enableHttps: true,
      csrfProtection: true,
      allowedOrigins: ["https://staging.example.com"],
      cookieSettings: {
        secure: true,
        sameSite: "lax",
        httpOnly: true
      }
    },
    monitoring: {
      errorReporting: {
        enabled: true,
        sampleRate: 1.0,
        endpoint: "https://errors.staging.example.com"
      },
      performance: { enabled: true, sampleRate: 0.5 }
    },
    maintenance: { enabled: false, message: "", allowedIPs: [] }
  },
  
  production: {
    environment: 'production',
    version: "1.0.0",
    timestamp: new Date().toISOString(),
    features: {
      analytics: { enabled: true, providers: ["google", "segment"], debugMode: false },
      darkMode: { enabled: true, default: "auto" },
      betaFeatures: { enabled: false, allowedUsers: [] },
      debugging: { enabled: false, logLevel: "error", consoleOutput: false }
    },
    ui: {
      theme: {
        primaryColor: "#007bff",
        secondaryColor: "#6c757d",
        fontFamily: "Inter, sans-serif"
      },
      layout: {
        sidebarCollapsed: false,
        compactMode: false,
        showDevTools: false
      }
    },
    api: {
      baseUrl: "https://api.production.example.com",
      endpoints: {
        analytics: "/v1/analytics",
        feedback: "/v1/feedback",
        auth: "/v1/auth"
      },
      timeout: 30000,
      retryAttempts: 3,
      rateLimiting: {
        enabled: true,
        maxRequests: 1000,
        windowMs: 60000
      }
    },
    security: {
      enableHttps: true,
      csrfProtection: true,
      allowedOrigins: ["https://app.example.com"],
      cookieSettings: {
        secure: true,
        sameSite: "strict",
        httpOnly: true
      }
    },
    monitoring: {
      errorReporting: {
        enabled: true,
        sampleRate: 1.0,
        endpoint: "https://errors.example.com"
      },
      performance: { enabled: true, sampleRate: 0.1 }
    },
    maintenance: { enabled: false, message: "", allowedIPs: [] }
  }
};
```

### Client-Side Implementation

#### Environment Detection and Configuration Fetcher
```typescript
class EnvironmentConfigurationClient {
  private config: EnvironmentConfiguration | null = null;
  private fetchPromise: Promise<EnvironmentConfiguration> | null = null;
  private detectedEnvironment: string | null = null;

  async initialize(): Promise<EnvironmentConfiguration> {
    // Prevent multiple simultaneous fetches
    if (this.fetchPromise) {
      return this.fetchPromise;
    }

    this.fetchPromise = this.fetchConfiguration();
    
    try {
      this.config = await this.fetchPromise;
      return this.config;
    } finally {
      this.fetchPromise = null;
    }
  }

  private detectEnvironment(): string {
    if (this.detectedEnvironment) {
      return this.detectedEnvironment;
    }

    const hostname = window.location.hostname;
    const origin = window.location.origin;

    // Environment detection logic
    if (hostname === 'localhost' || hostname === '127.0.0.1' || hostname.includes('.local')) {
      this.detectedEnvironment = 'development';
    } else if (hostname.includes('staging') || hostname.includes('stage') || origin.includes('staging')) {
      this.detectedEnvironment = 'staging';
    } else if (hostname.includes('production') || hostname.includes('prod') || !hostname.includes('.')) {
      this.detectedEnvironment = 'production';
    } else {
      // Default to production for safety
      this.detectedEnvironment = 'production';
    }

    console.info(`Detected environment: ${this.detectedEnvironment}`);
    return this.detectedEnvironment;
  }

  private async fetchConfiguration(): Promise<EnvironmentConfiguration> {
    try {
      const environment = this.detectEnvironment();
      const response = await fetch(`/api/configuration/environment?env=${environment}`, {
        method: 'GET',
        headers: {
          'Accept': 'application/json',
          'X-Environment': environment
        }
      });

      if (!response.ok) {
        throw new Error(`Configuration fetch failed: ${response.status}`);
      }

      const config = await response.json();
      
      // Validate environment matches
      if (config.environment !== environment) {
        console.warn(`Environment mismatch: expected ${environment}, got ${config.environment}`);
      }

      return config;
    } catch (error) {
      console.error('Failed to fetch configuration:', error);
      return this.getDefaultConfiguration();
    }
  }

  private getDefaultConfiguration(): EnvironmentConfiguration {
    const environment = this.detectEnvironment();
    return ENVIRONMENT_DEFAULTS[environment] || ENVIRONMENT_DEFAULTS.production;
  }

  getEnvironment(): string {
    return this.detectedEnvironment || this.detectEnvironment();
  }
}
```

#### React Integration Example
```tsx
// Environment Configuration Context
interface EnvironmentContextValue {
  config: EnvironmentConfiguration;
  environment: string;
  isProduction: boolean;
  isStaging: boolean;
  isDevelopment: boolean;
}

const EnvironmentContext = React.createContext<EnvironmentContextValue | null>(null);

// Environment Configuration Provider
export const EnvironmentProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [config, setConfig] = useState<EnvironmentConfiguration | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const configClient = new EnvironmentConfigurationClient();
    
    configClient.initialize()
      .then(configuration => {
        setConfig(configuration);
        
        // Apply environment-specific settings
        applyEnvironmentSettings(configuration);
      })
      .catch(err => {
        console.error('Configuration initialization failed:', err);
        setError(err);
        
        // Use fallback configuration
        const fallbackConfig = configClient.getDefaultConfiguration();
        setConfig(fallbackConfig);
      })
      .finally(() => setLoading(false));
  }, []);

  if (loading) {
    return <LoadingSpinner message="Loading environment configuration..." />;
  }

  if (!config) {
    return <ErrorBoundary error={error} />;
  }

  const contextValue: EnvironmentContextValue = {
    config,
    environment: config.environment,
    isProduction: config.environment === 'production',
    isStaging: config.environment === 'staging',
    isDevelopment: config.environment === 'development'
  };

  return (
    <EnvironmentContext.Provider value={contextValue}>
      {children}
    </EnvironmentContext.Provider>
  );
};

// Environment Configuration Hook
export const useEnvironment = () => {
  const context = useContext(EnvironmentContext);
  if (!context) {
    throw new Error('useEnvironment must be used within EnvironmentProvider');
  }
  return context;
};

// Helper hook for feature flags
export const useFeature = (featureName: string) => {
  const { config } = useEnvironment();
  return config.features[featureName]?.enabled || false;
};

// Apply environment-specific settings
function applyEnvironmentSettings(config: EnvironmentConfiguration) {
  // Set log level
  if (config.features.debugging?.enabled) {
    console.log(`Setting log level to: ${config.features.debugging.logLevel}`);
    // Configure your logging library
  }

  // Configure error reporting
  if (config.monitoring?.errorReporting?.enabled) {
    // Initialize error reporting service
    console.log(`Initializing error reporting for ${config.environment}`);
  }

  // Apply security headers
  if (config.security?.csrfProtection) {
    // Configure CSRF tokens
  }
}
```

### Configuration Repository Structure

```
environments/
├── base/
│   ├── defaults.json          # Shared default values
│   ├── features.json          # Base feature definitions
│   └── security.json          # Base security settings
├── development/
│   ├── config.json            # Main development config
│   ├── api-endpoints.json     # Dev API endpoints
│   └── feature-overrides.json # Dev-specific features
├── staging/
│   ├── config.json            # Main staging config
│   ├── api-endpoints.json     # Staging API endpoints
│   ├── feature-overrides.json # Staging features
│   └── beta-users.json        # Beta test users
└── production/
    ├── config.json            # Main production config
    ├── api-endpoints.json     # Production API endpoints
    ├── security.json          # Production security
    └── monitoring.json        # Production monitoring
```

### Environment Variables

```env
# GitHub Configuration Repository
GITHUB_CONFIG_REPO=owner/environment-configs
GITHUB_CONFIG_TOKEN=ghp_xxxxxxxxxxxx
GITHUB_CONFIG_BRANCH=main

# Environment Detection
APP_ENVIRONMENT=production     # Override auto-detection if needed
ALLOWED_ENVIRONMENTS=development,staging,production

# Environment-Specific Cache Settings
DEV_CONFIG_CACHE_TTL=60        # 1 minute for development
STAGING_CONFIG_CACHE_TTL=300   # 5 minutes for staging
PROD_CONFIG_CACHE_TTL=900      # 15 minutes for production
CONFIG_CACHE_ENABLED=true

# Rate Limiting per Environment
DEV_RATE_LIMIT=1000           # Higher for development
STAGING_RATE_LIMIT=500        # Moderate for staging
PROD_RATE_LIMIT=100           # Conservative for production

# Asset Owner Configuration (for database caching)
ASSET_OWNER_CLASS=application
ASSET_OWNER_NAME=owui-feedback-${APP_ENVIRONMENT}
```

## Security Considerations

### 1. Environment-Specific Security
- **Production Hardening**: Strictest security settings for production
- **Development Flexibility**: Relaxed settings for local development
- **Staging Balance**: Production-like security with debugging capabilities
- **Environment Isolation**: Prevent cross-environment configuration access

### 2. Unprotected Endpoint Security
- **Environment-Based Rate Limiting**: Different limits per environment
- **CORS Configuration**: Environment-specific allowed origins
- **Request Validation**: Validate environment parameter against whitelist
- **Response Filtering**: Remove sensitive data based on environment

### 3. Configuration Content Security
- **Environment Secrets**: Store environment-specific secrets separately
- **Public vs Private**: Separate public config from environment secrets
- **Access Control**: Environment-based repository access controls
- **Audit Trail**: Track who modifies each environment's configuration

### 4. Client-Side Security
- **Environment Validation**: Ensure client runs in expected environment
- **Feature Authorization**: Environment-aware feature access
- **Debug Protection**: Disable debugging features in production
- **Secure Defaults**: Production-safe defaults for all environments

## Performance Optimization

### 1. Environment-Aware Server-Side Caching
```typescript
// Environment-specific caching strategy
class EnvironmentConfigurationCache {
  private memoryCache: Map<string, CacheEntry> = new Map();
  private etagCache: Map<string, string> = new Map();

  async getWithEtag(environment: string): Promise<{ value: any, etag: string } | null> {
    const key = `env:${environment}`;
    const cached = this.memoryCache.get(key);
    
    if (cached && Date.now() < cached.expiry) {
      return {
        value: cached.value,
        etag: this.etagCache.get(key) || '',
        environment: cached.environment,
        cachedAt: cached.cachedAt
      };
    }
    return null;
  }

  setWithEtag(environment: string, value: any, etag: string): void {
    const key = `env:${environment}`;
    const ttl = this.getEnvironmentTTL(environment);
    
    this.memoryCache.set(key, {
      value,
      expiry: Date.now() + (ttl * 1000),
      environment,
      cachedAt: new Date().toISOString()
    });
    this.etagCache.set(key, etag);
  }

  private getEnvironmentTTL(environment: string): number {
    const ttlConfig = {
      development: parseInt(process.env.DEV_CONFIG_CACHE_TTL || '60'),
      staging: parseInt(process.env.STAGING_CONFIG_CACHE_TTL || '300'),
      production: parseInt(process.env.PROD_CONFIG_CACHE_TTL || '900')
    };
    
    return ttlConfig[environment] || 300;
  }
}
```

### 2. Client-Side Caching
```typescript
// Browser storage with expiration
class ClientConfigCache {
  private readonly STORAGE_KEY = 'client-config';
  private readonly CACHE_DURATION = 60 * 60 * 1000; // 1 hour

  save(config: ClientConfiguration): void {
    const cacheData = {
      config,
      expiry: Date.now() + this.CACHE_DURATION
    };
    localStorage.setItem(this.STORAGE_KEY, JSON.stringify(cacheData));
  }

  load(): ClientConfiguration | null {
    try {
      const stored = localStorage.getItem(this.STORAGE_KEY);
      if (!stored) return null;

      const cacheData = JSON.parse(stored);
      if (Date.now() > cacheData.expiry) {
        localStorage.removeItem(this.STORAGE_KEY);
        return null;
      }

      return cacheData.config;
    } catch {
      return null;
    }
  }
}
```

### 3. Compression
- Enable gzip/brotli compression for API responses
- Minimize configuration JSON structure
- Use short property names for frequently accessed values

## Error Handling

### 1. Network Failures
```typescript
async function fetchWithRetry(url: string, attempts = 3): Promise<Response> {
  for (let i = 0; i < attempts; i++) {
    try {
      const response = await fetch(url);
      if (response.ok) return response;
      
      if (response.status >= 500 && i < attempts - 1) {
        await delay(Math.pow(2, i) * 1000); // Exponential backoff
        continue;
      }
      
      throw new Error(`HTTP ${response.status}`);
    } catch (error) {
      if (i === attempts - 1) throw error;
      await delay(Math.pow(2, i) * 1000);
    }
  }
  throw new Error('Max retries exceeded');
}
```

### 2. Fallback Strategies
```typescript
class ConfigurationFallback {
  private readonly fallbackChain = [
    () => this.fetchFromAPI(),
    () => this.loadFromLocalStorage(),
    () => this.loadFromBrowserCache(),
    () => this.useHardcodedDefaults()
  ];

  async getConfiguration(): Promise<ClientConfiguration> {
    for (const fallback of this.fallbackChain) {
      try {
        const config = await fallback();
        if (config) return config;
      } catch (error) {
        console.warn('Fallback failed:', error);
      }
    }
    
    throw new Error('All configuration sources failed');
  }
}
```

### 3. Validation
```typescript
function validateConfiguration(config: any): config is ClientConfiguration {
  // Use a schema validation library like Joi or Yup
  const schema = {
    version: 'string',
    timestamp: 'string',
    features: 'object',
    ui: 'object',
    api: 'object',
    maintenance: 'object'
  };

  // Validate structure and types
  return validateSchema(config, schema);
}
```

## Monitoring and Observability

### 1. Environment-Specific Metrics
- Configuration fetch rates per environment
- Environment detection accuracy
- Cache performance by environment
- Environment-specific error rates
- Cross-environment configuration drift

### 2. Environment-Aware Logging
```typescript
interface EnvironmentConfigurationEvent {
  type: 'fetch' | 'cache_hit' | 'cache_miss' | 'fallback' | 'error' | 'env_mismatch';
  environment: string;
  detectedEnvironment: string;
  source: 'github' | 'cache' | 'default';
  version?: string;
  duration?: number;
  error?: string;
}

class EnvironmentConfigurationLogger {
  private environment: string;

  constructor(environment: string) {
    this.environment = environment;
  }

  log(event: EnvironmentConfigurationEvent): void {
    const logData = {
      ...event,
      timestamp: new Date().toISOString(),
      serverEnvironment: process.env.APP_ENVIRONMENT || 'unknown'
    };

    // Environment-specific logging
    if (this.environment === 'development') {
      console.log('[EnvConfig]', logData);
    } else if (this.environment === 'staging') {
      console.info('[EnvConfig]', { ...logData, details: event.error });
    } else {
      // Production: minimal logging
      if (event.type === 'error') {
        console.error('[EnvConfig]', event.error);
      }
    }
    
    // Send metrics based on environment
    this.sendMetrics(logData);
  }

  private sendMetrics(data: any): void {
    const { monitoring } = data.config || {};
    
    if (monitoring?.errorReporting?.enabled && window.errorReporter) {
      window.errorReporter.track('env_config_event', data);
    }
  }
}
```

### 3. Health Checks
```typescript
// Environment-specific health check endpoint
app.get('/api/configuration/health/:environment?', async (req, res) => {
  const environment = req.params.environment || detectEnvironment(req);
  
  const health = {
    status: 'healthy',
    environment,
    checks: {
      github: await checkGitHubConnectivity(),
      cache: checkEnvironmentCacheStatus(environment),
      rateLimit: checkEnvironmentRateLimit(environment),
      configValid: await validateEnvironmentConfig(environment)
    },
    timestamp: new Date().toISOString()
  };
  
  const overallStatus = Object.values(health.checks).every(c => c.status === 'ok');
  res.status(overallStatus ? 200 : 503).json(health);
});
```

## Environment Detection Strategies

### 1. Hostname-Based Detection
```typescript
function detectEnvironmentByHostname(hostname: string): string {
  const rules = [
    { pattern: /^localhost|127\.0\.0\.1|.*\.local$/, environment: 'development' },
    { pattern: /^staging\.|.*-staging\.|.*\.staging\./, environment: 'staging' },
    { pattern: /^www\.|.*\.com$|.*\.org$|.*\.net$/, environment: 'production' }
  ];

  for (const rule of rules) {
    if (rule.pattern.test(hostname)) {
      return rule.environment;
    }
  }

  return 'production'; // Default to production for safety
}
```

### 2. URL Path-Based Detection
```typescript
function detectEnvironmentByPath(pathname: string): string | null {
  // Check for environment indicators in URL path
  if (pathname.includes('/dev/') || pathname.includes('/development/')) {
    return 'development';
  }
  if (pathname.includes('/staging/') || pathname.includes('/stage/')) {
    return 'staging';
  }
  if (pathname.includes('/prod/') || pathname.includes('/production/')) {
    return 'production';
  }
  
  return null;
}
```

### 3. Header-Based Detection
```typescript
function detectEnvironmentByHeaders(headers: Headers): string | null {
  // Check custom headers
  const envHeader = headers.get('X-Environment');
  if (envHeader && ['development', 'staging', 'production'].includes(envHeader)) {
    return envHeader;
  }

  // Check origin header patterns
  const origin = headers.get('Origin');
  if (origin) {
    return detectEnvironmentByHostname(new URL(origin).hostname);
  }

  return null;
}
```

### 4. Combined Detection Strategy
```typescript
class EnvironmentDetector {
  private static readonly strategies = [
    this.checkOverrideParameter,
    this.checkEnvironmentVariable,
    this.checkHeaders,
    this.checkHostname,
    this.checkPath,
    this.checkPort
  ];

  static detect(request: Request): string {
    for (const strategy of this.strategies) {
      const result = strategy(request);
      if (result) {
        console.info(`Environment detected as '${result}' using ${strategy.name}`);
        return result;
      }
    }

    // Default to production for safety
    console.warn('Could not detect environment, defaulting to production');
    return 'production';
  }

  private static checkOverrideParameter(request: Request): string | null {
    const url = new URL(request.url);
    const override = url.searchParams.get('env');
    
    if (override && this.isValidEnvironment(override)) {
      return override;
    }
    
    return null;
  }

  private static checkEnvironmentVariable(request: Request): string | null {
    // Server-side only
    if (typeof process !== 'undefined' && process.env.APP_ENVIRONMENT) {
      return process.env.APP_ENVIRONMENT;
    }
    return null;
  }

  private static checkPort(request: Request): string | null {
    const url = new URL(request.url);
    const port = url.port || (url.protocol === 'https:' ? '443' : '80');
    
    // Common development ports
    if (['3000', '3001', '5173', '8080', '8000'].includes(port)) {
      return 'development';
    }
    
    return null;
  }

  private static isValidEnvironment(env: string): boolean {
    const allowed = process.env.ALLOWED_ENVIRONMENTS?.split(',') || 
                   ['development', 'staging', 'production'];
    return allowed.includes(env);
  }
}
```

## Testing Strategy

### 1. Unit Tests
```typescript
describe('EnvironmentConfigurationService', () => {
  it('should return environment-specific configuration', async () => {
    const service = new EnvironmentConfigurationService();
    const config = await service.getEnvironmentConfiguration('staging');
    
    expect(config.environment).toBe('staging');
    expect(config.api.baseUrl).toContain('staging');
  });

  it('should cache configurations per environment', async () => {
    const service = new EnvironmentConfigurationService();
    
    const devConfig = await service.getEnvironmentConfiguration('development');
    const prodConfig = await service.getEnvironmentConfiguration('production');
    
    expect(devConfig.environment).toBe('development');
    expect(prodConfig.environment).toBe('production');
    expect(devConfig.api.baseUrl).not.toBe(prodConfig.api.baseUrl);
  });

  it('should use environment-specific TTL', async () => {
    const cache = new EnvironmentConfigurationCache();
    const devConfig = { environment: 'development' };
    const prodConfig = { environment: 'production' };
    
    cache.set('development', devConfig);
    cache.set('production', prodConfig);
    
    // Development cache expires faster
    jest.advanceTimersByTime(65000); // 65 seconds
    
    expect(await cache.get('development')).toBeNull();
    expect(await cache.get('production')).toBeTruthy();
  });
});
```

### 2. Integration Tests
```typescript
describe('Environment Configuration API', () => {
  it('should auto-detect environment from request', async () => {
    const response = await request(app)
      .get('/api/configuration/environment')
      .set('Host', 'staging.example.com')
      .expect(200);
    
    expect(response.body.environment).toBe('staging');
  });

  it('should respect environment override parameter', async () => {
    const response = await request(app)
      .get('/api/configuration/environment?env=development')
      .set('Host', 'production.example.com')
      .expect(200);
    
    expect(response.body.environment).toBe('development');
  });

  it('should apply environment-specific rate limits', async () => {
    // Production has stricter limits
    process.env.APP_ENVIRONMENT = 'production';
    
    for (let i = 0; i < 101; i++) {
      await request(app).get('/api/configuration/environment');
    }
    
    await request(app)
      .get('/api/configuration/environment')
      .expect(429);

    // Development has higher limits
    process.env.APP_ENVIRONMENT = 'development';
    
    for (let i = 0; i < 500; i++) {
      await request(app).get('/api/configuration/environment');
    }
    
    await request(app)
      .get('/api/configuration/environment')
      .expect(200);
  });
});
```

### 3. E2E Tests
```typescript
describe('Environment Configuration Flow', () => {
  it('should load development configuration on localhost', () => {
    cy.visit('http://localhost:3000');
    
    cy.window().then(win => {
      expect(win.appConfig.environment).to.equal('development');
      expect(win.appConfig.features.debugging.enabled).to.be.true;
      expect(win.appConfig.ui.layout.showDevTools).to.be.true;
    });
  });

  it('should load production configuration on production domain', () => {
    cy.visit('https://app.example.com');
    
    cy.window().then(win => {
      expect(win.appConfig.environment).to.equal('production');
      expect(win.appConfig.features.debugging.enabled).to.be.false;
      expect(win.appConfig.security.enableHttps).to.be.true;
    });
  });

  it('should apply environment-specific features', () => {
    // Development environment
    cy.visit('http://localhost:3000');
    cy.get('[data-testid="dev-tools"]').should('be.visible');
    cy.get('[data-testid="beta-features"]').should('be.visible');
    
    // Production environment
    cy.visit('https://app.example.com');
    cy.get('[data-testid="dev-tools"]').should('not.exist');
    cy.get('[data-testid="beta-features"]').should('not.exist');
  });
});
```

## Migration Guide

### 1. Preparing Existing Configuration
```typescript
// Extract current hardcoded configuration
const extractConfiguration = () => {
  const config = {
    features: {
      // Collect all feature flags
    },
    ui: {
      // Extract UI settings
    },
    api: {
      // Gather API endpoints
    }
  };
  
  return config;
};
```

### 2. Environment-Specific Migration
```typescript
// Gradual migration per environment
class EnvironmentMigrationService {
  async getConfiguration(environment: string): Promise<EnvironmentConfiguration> {
    const migrationFlags = {
      development: true,   // Migrate dev first
      staging: process.env.MIGRATE_STAGING === 'true',
      production: process.env.MIGRATE_PRODUCTION === 'true'
    };

    if (migrationFlags[environment]) {
      return this.fetchRemoteConfiguration(environment);
    }
    
    return this.getLocalConfiguration(environment);
  }
}
```

### 3. Rollback Plan
```typescript
// Quick rollback mechanism
const configSource = {
  remote: async () => fetchFromGitHub(),
  local: async () => importLocalConfig(),
  emergency: async () => EMERGENCY_CONFIG
};

const activeSource = process.env.CONFIG_SOURCE || 'remote';
const config = await configSource[activeSource]();
```

## Best Practices

### 1. Environment Configuration Structure
- **Base + Override Pattern**: Share common settings, override per environment
- **Environment Naming**: Use consistent names (development, staging, production)
- **Environment Indicators**: Clear visual/functional differences between environments
- **Secret Management**: Never store secrets in environment configs

### 2. Environment Promotion Strategy
- **Dev → Staging → Production**: Test configurations progressively
- **Environment Parity**: Keep environments as similar as possible
- **Configuration Drift Detection**: Monitor differences between environments
- **Automated Validation**: Validate configs before promoting

### 3. Environment-Specific Performance
- **Development**: Minimal caching for rapid iteration
- **Staging**: Production-like caching with debug capabilities
- **Production**: Aggressive caching for optimal performance
- **Cache Invalidation**: Environment-specific cache clear mechanisms

### 4. Environment Safety
- **Production Guards**: Prevent accidental production changes
- **Environment Indicators**: Clear UI indicators of current environment
- **Rollback Strategy**: Quick rollback per environment
- **Audit Logging**: Track all environment configuration changes

## Common Pitfalls

### 1. Environment Security Issues
- ❌ Same secrets across all environments
- ❌ Production config accessible in development
- ❌ No environment validation
- ✅ Environment-specific secrets management
- ✅ Strict environment access controls
- ✅ Environment whitelist validation

### 2. Environment Detection Problems
- ❌ Hardcoded environment values
- ❌ Client-side only detection
- ❌ No fallback detection strategy
- ✅ Multiple detection methods
- ✅ Server-side environment validation
- ✅ Clear detection precedence rules

### 3. Environment Configuration Drift
- ❌ Manual environment config updates
- ❌ No validation between environments
- ❌ Inconsistent feature flags
- ✅ Automated config deployment
- ✅ Environment parity checks
- ✅ Centralized feature flag management

## Future Enhancements

### 1. WebSocket Updates
```typescript
// Real-time configuration updates
class ConfigurationWebSocket {
  private ws: WebSocket;
  
  connect() {
    this.ws = new WebSocket('wss://api.example.com/config/stream');
    
    this.ws.on('message', (data) => {
      const update = JSON.parse(data);
      this.applyConfigurationUpdate(update);
    });
  }
}
```

### 2. A/B Testing Integration
```typescript
interface ABTestConfiguration {
  experiments: {
    [key: string]: {
      enabled: boolean;
      variants: string[];
      traffic: number[];
    };
  };
}
```

### 3. Machine Learning Optimization
- Predictive caching based on usage patterns
- Automatic configuration optimization
- Anomaly detection for configuration changes

### 4. Multi-Region Support
- Geographic configuration distribution
- Region-specific configurations
- Automatic failover between regions

## Conclusion

The environment configuration pattern provides a robust, scalable solution for managing environment-specific configurations across different deployment stages. By implementing automatic environment detection, environment-specific caching strategies, and proper fallback mechanisms, applications can seamlessly operate in any environment without code changes or manual configuration. This pattern ensures that development remains flexible, staging mirrors production closely, and production maintains the highest security and performance standards. Following the principles outlined in this document enables teams to maintain clear environment separation while reducing configuration drift and deployment risks.