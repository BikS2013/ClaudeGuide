# Configuration Manager Usage Guide

This guide provides comprehensive instructions for using the `@biks2013` configuration management modules in your applications.

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Module Overview](#module-overview)
- [Common Use Cases](#common-use-cases)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## Installation

Install the required packages based on your needs:

```bash
# Full stack (config service with both sources)
npm install @biks2013/config-service @biks2013/github-asset-client @biks2013/asset-database

# GitHub only
npm install @biks2013/github-asset-client

# Database only
npm install @biks2013/asset-database
```

## Quick Start

### 1. Basic Configuration Service

```typescript
import { createConfigService } from '@biks2013/config-service';
import { GitHubAssetClient } from '@biks2013/github-asset-client';
import { AssetDatabaseService } from '@biks2013/asset-database';

// Setup clients
const githubClient = new GitHubAssetClient({
  repo: 'myorg/config-repo',
  token: process.env.GITHUB_TOKEN!,
  branch: 'main'
});

const database = new AssetDatabaseService({
  connectionString: process.env.DATABASE_URL!,
  ownerCategory: 'my-service',
  ownerKey: 'production'
});

// Create service
const configServiceFactory = createConfigService({
  sources: [
    {
      type: 'github',
      priority: 1,
      options: {
        client: githubClient,
        assetKey: 'config/app.json'
      }
    },
    {
      type: 'database',
      priority: 2,
      options: {
        service: database,
        assetKey: 'app-config',
        category: 'configuration'
      }
    }
  ],
  parser: async (content) => JSON.parse(content),
  verbose: true
});

// Use it
const configService = configServiceFactory();
const config = await configService.getConfig();
```

### 2. Direct GitHub Usage

```typescript
import { GitHubAssetClient } from '@biks2013/github-asset-client';

const client = new GitHubAssetClient({
  repo: 'myorg/config-repo',
  token: process.env.GITHUB_TOKEN!,
  branch: 'main',
  cacheEnabled: true,
  cacheTTL: 300000 // 5 minutes
});

// Read a file
const file = await client.getAsset('config/production.yaml');
console.log(file.content);

// List directory
const files = await client.listAssets('config');

// Search files
const results = await client.searchAssets('database settings');
```

### 3. Direct Database Usage

```typescript
import { AssetDatabaseService } from '@biks2013/asset-database';

const database = new AssetDatabaseService({
  connectionString: 'postgresql://user:pass@localhost:5432/mydb',
  ownerCategory: 'my-app',
  ownerKey: 'production',
  verbose: true
});

// Ensure schema exists
await database.ensureSchema();

// Store configuration
await database.storeAsset(
  'app-config',
  JSON.stringify({ version: '1.0.0', features: { api: true } }),
  'configuration',
  'Main application configuration'
);

// Retrieve configuration
const asset = await database.getAsset('app-config', 'configuration');
if (asset) {
  const config = JSON.parse(asset.data.content);
  console.log(config);
}

// Get history
const history = await database.getAssetHistory('app-config');
```

## Module Overview

### @biks2013/config-service

The main orchestrator that provides:
- **Priority-based loading**: Try sources in order
- **Automatic fallback**: Use database when GitHub fails
- **Transparent caching**: Cache from GitHub to database
- **Type safety**: Full TypeScript support

### @biks2013/github-asset-client

Direct GitHub integration with:
- **File retrieval**: Get any file from your repo
- **Directory listing**: Browse repository structure
- **Search capability**: Find files by content
- **Built-in caching**: Reduce API calls
- **Retry logic**: Handle transient failures

### @biks2013/asset-database

PostgreSQL storage with:
- **Version history**: Track all changes
- **Audit logging**: Know who changed what
- **Hash tracking**: Detect content changes
- **Categories**: Organize your assets
- **Connection pooling**: Production-ready

## Common Use Cases

### 1. Multi-Environment Configuration

```typescript
// Load environment-specific config
const env = process.env.NODE_ENV || 'development';

const configService = createConfigService({
  sources: [
    {
      type: 'github',
      priority: 1,
      options: {
        client: githubClient,
        assetKey: `config/${env}/app.json`
      }
    },
    {
      type: 'database',
      priority: 2,
      options: {
        service: database,
        assetKey: `app-config-${env}`,
        category: env
      }
    }
  ],
  parser: async (content) => JSON.parse(content)
})();
```

### 2. Feature Flags Service

```typescript
interface FeatureFlags {
  features: {
    [key: string]: {
      enabled: boolean;
      rolloutPercentage?: number;
      allowedUsers?: string[];
    }
  }
}

const featureFlagService = createConfigService<FeatureFlags>({
  sources: [
    {
      type: 'github',
      priority: 1,
      options: {
        client: githubClient,
        assetKey: 'config/feature-flags.json'
      }
    }
  ],
  parser: async (content) => JSON.parse(content)
})();

// Check if feature is enabled
async function isFeatureEnabled(feature: string, userId?: string): Promise<boolean> {
  const flags = await featureFlagService.getConfig();
  const flag = flags?.features[feature];
  
  if (!flag) return false;
  if (!flag.enabled) return false;
  
  if (flag.allowedUsers && userId) {
    return flag.allowedUsers.includes(userId);
  }
  
  return true;
}
```

### 3. Dynamic API Configuration

```typescript
interface ApiConfig {
  endpoints: {
    [service: string]: {
      url: string;
      timeout: number;
      retries: number;
    }
  };
  rateLimit: {
    requests: number;
    window: number;
  };
}

const apiConfigService = createConfigService<ApiConfig>({
  sources: [
    {
      type: 'github',
      priority: 1,
      options: {
        client: githubClient,
        assetKey: 'config/api-endpoints.yaml'
      }
    },
    {
      type: 'database',
      priority: 2,
      options: {
        service: database,
        assetKey: 'api-config',
        category: 'api'
      }
    }
  ],
  parser: async (content) => parseYaml(content),
  verbose: true
})();

// Use in API client
class ApiClient {
  async callService(service: string, path: string) {
    const config = await apiConfigService.getConfig();
    const endpoint = config?.endpoints[service];
    
    if (!endpoint) {
      throw new Error(`Unknown service: ${service}`);
    }
    
    return axios({
      url: `${endpoint.url}${path}`,
      timeout: endpoint.timeout,
      retry: endpoint.retries
    });
  }
}
```

### 4. Database Connection Management

```typescript
interface DbConfig {
  databases: {
    [name: string]: {
      host: string;
      port: number;
      database: string;
      poolSize: number;
      ssl: boolean;
    }
  }
}

const dbConfigService = createConfigService<DbConfig>({
  sources: [
    {
      type: 'github',
      priority: 1,
      options: {
        client: githubClient,
        assetKey: 'config/databases.json'
      }
    }
  ],
  parser: async (content) => JSON.parse(content)
})();

// Database connection factory
async function getDbConnection(name: string) {
  const config = await dbConfigService.getConfig();
  const dbConfig = config?.databases[name];
  
  if (!dbConfig) {
    throw new Error(`Database config not found: ${name}`);
  }
  
  return new Pool({
    host: dbConfig.host,
    port: dbConfig.port,
    database: dbConfig.database,
    max: dbConfig.poolSize,
    ssl: dbConfig.ssl
  });
}
```

## Best Practices

### 1. Error Handling

```typescript
try {
  const config = await configService.getConfig();
  if (!config) {
    // Handle missing configuration
    console.error('Configuration not found in any source');
    // Use defaults or throw error based on your needs
  }
} catch (error) {
  // Handle loading errors
  console.error('Failed to load configuration:', error);
  // Implement fallback strategy
}
```

### 2. Configuration Validation

```typescript
const configServiceFactory = createConfigService({
  sources: [...],
  parser: async (content) => {
    const config = JSON.parse(content);
    
    // Validate schema
    const { error } = validateSchema(config);
    if (error) {
      throw new Error(`Invalid configuration: ${error.message}`);
    }
    
    // Validate required fields
    const required = ['apiKey', 'database', 'features'];
    for (const field of required) {
      if (!config[field]) {
        throw new Error(`Missing required field: ${field}`);
      }
    }
    
    return config;
  }
});
```

### 3. Environment Variables

```typescript
// .env file
GITHUB_TOKEN=ghp_xxxxxxxxxxxxx
GITHUB_REPO=myorg/config-repo
GITHUB_BRANCH=main
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
CONFIG_VERBOSE=true

// Usage
const githubClient = new GitHubAssetClient({
  repo: process.env.GITHUB_REPO!,
  token: process.env.GITHUB_TOKEN!,
  branch: process.env.GITHUB_BRANCH || 'main'
});

const database = new AssetDatabaseService({
  connectionString: process.env.DATABASE_URL!,
  ownerCategory: 'production',
  ownerKey: process.env.SERVICE_NAME || 'my-service',
  verbose: process.env.CONFIG_VERBOSE === 'true'
});
```

### 4. Refresh Strategy

```typescript
// Periodic refresh
const REFRESH_INTERVAL = 5 * 60 * 1000; // 5 minutes

setInterval(async () => {
  try {
    await configService.reload();
    console.log('Configuration refreshed');
  } catch (error) {
    console.error('Configuration refresh failed:', error);
    // Continue using cached configuration
  }
}, REFRESH_INTERVAL);

// Event-based refresh
eventEmitter.on('config-update', async () => {
  await configService.reload();
});
```

### 5. Testing

```typescript
// Mock configuration for tests
import { createConfigService } from '@biks2013/config-service';

const mockConfig = {
  database: { host: 'localhost', port: 5432 },
  features: { cache: true }
};

const testConfigService = createConfigService({
  sources: [
    {
      type: 'custom',
      priority: 1,
      options: {
        load: async () => JSON.stringify(mockConfig)
      }
    }
  ],
  parser: async (content) => JSON.parse(content)
})();
```

## Troubleshooting

### GitHub Token Issues

```typescript
// Verify token permissions
const client = new GitHubAssetClient({
  repo: 'myorg/repo',
  token: process.env.GITHUB_TOKEN!,
  timeout: 5000 // Fail fast for testing
});

try {
  await client.getAsset('README.md');
  console.log('Token is valid');
} catch (error) {
  if (error.message.includes('401')) {
    console.error('Invalid token');
  } else if (error.message.includes('403')) {
    console.error('Token lacks required permissions');
  }
}
```

### Database Connection

```typescript
// Test database connection
const database = new AssetDatabaseService({
  connectionString: process.env.DATABASE_URL!,
  ownerCategory: 'test',
  ownerKey: 'test',
  verbose: true
});

try {
  await database.ensureSchema();
  console.log('Database connected successfully');
} catch (error) {
  console.error('Database connection failed:', error);
  // Check connection string format
  // Verify PostgreSQL is running
  // Check network connectivity
}
```

### Debugging Configuration Loading

Enable verbose mode to see detailed logs:

```typescript
const configService = createConfigService({
  sources: [...],
  parser: async (content) => JSON.parse(content),
  verbose: true // Enable detailed logging
})();

// You'll see:
// - Which source is being tried
// - Success/failure of each attempt
// - Content differences when caching
// - Detailed error messages
```

## Advanced Patterns

### Custom Configuration Source

```typescript
// Implement a custom source
class RedisConfigSource {
  constructor(private redis: RedisClient) {}
  
  async load(key: string): Promise<string | null> {
    return await this.redis.get(key);
  }
  
  async store(key: string, content: string): Promise<void> {
    await this.redis.set(key, content);
  }
}

// Use in config service
const configService = createConfigService({
  sources: [
    {
      type: 'custom',
      priority: 1,
      options: {
        load: async () => {
          const content = await redisSource.load('app-config');
          if (!content) throw new Error('Not found');
          return content;
        },
        store: async (content) => {
          await redisSource.store('app-config', content);
        }
      }
    }
  ],
  parser: async (content) => JSON.parse(content)
})();
```

### Configuration Inheritance

```typescript
// Base configuration
const baseConfig = await githubClient.getAsset('config/base.json');

// Environment-specific override
const envConfig = await githubClient.getAsset(`config/${env}.json`);

// Merge configurations
const finalConfig = {
  ...JSON.parse(baseConfig.content),
  ...JSON.parse(envConfig.content)
};
```

## Summary

The `@biks2013` configuration management modules provide a robust, production-ready solution for managing application configuration with:

- **High availability** through GitHub + Database fallback
- **Version control** integration via GitHub
- **Audit trails** through database history
- **Type safety** with TypeScript
- **Flexible architecture** for various use cases

For more examples, check the test server implementation in `/test_scripts/config-tester/`.