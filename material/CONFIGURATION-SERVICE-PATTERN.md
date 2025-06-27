# Configuration Service Pattern

This document describes the standard pattern for implementing configuration file dependencies that are retrieved through the configuration repository approach.

## Overview

This pattern provides a consistent way to implement configuration services that:
- Retrieve configuration files from a GitHub repository (configuration-repo approach)
- Maintain their own in-memory caching, independent from the generic asset cache
- Remain simple and lean, avoiding unnecessary interfaces or layers of complexity

## Core Implementation

### Base Configuration Service Template

```typescript
// config-service-template.ts
import { readFileSync, existsSync } from 'fs';
import { join } from 'path';
import { getGitHubAssetService } from './githubAssetService.js';

export class ConfigService<T> {
  private configs: Map<string, any> = new Map();
  private rawCache: string | null = null;
  private initialized: boolean = false;
  private initPromise: Promise<void> | null = null;
  private readonly assetKey: string;
  private readonly localPath: string;
  private readonly parser: (content: string) => T;

  constructor(
    assetKey: string,
    localFileName: string,
    parser: (content: string) => T
  ) {
    this.assetKey = assetKey;
    this.localPath = join(process.cwd(), localFileName);
    this.parser = parser;
  }

  private async loadConfiguration(): Promise<void> {
    let content: string | null = null;

    // Check in-memory cache first
    if (this.rawCache) {
      content = this.rawCache;
    } else {
      // Try GitHub asset service
      const assetService = getGitHubAssetService();
      if (assetService.isServiceConfigured()) {
        try {
          content = await assetService.getAsset(this.assetKey, 'config');
          this.rawCache = content; // Cache for future use
        } catch (error) {
          console.error(`Failed to load from GitHub: ${error}`);
          // Fall back to local file
        }
      }

      // Fall back to local file if needed
      if (!content && existsSync(this.localPath)) {
        content = readFileSync(this.localPath, 'utf8');
        this.rawCache = content;
      }
    }

    if (!content) {
      throw new Error(`Configuration not found: ${this.assetKey}`);
    }

    // Parse and store configuration
    const parsed = this.parser(content);
    this.processConfiguration(parsed);
  }

  protected processConfiguration(data: T): void {
    // Override in subclass to handle specific configuration structure
    throw new Error('processConfiguration must be implemented');
  }

  private async ensureInitialized(): Promise<void> {
    if (this.initialized) return;
    
    if (this.initPromise) {
      await this.initPromise;
      return;
    }

    this.initPromise = (async () => {
      await this.loadConfiguration();
      this.initialized = true;
    })();
    
    await this.initPromise;
  }

  async getConfig(key?: string): Promise<any> {
    await this.ensureInitialized();
    return key ? this.configs.get(key) : Array.from(this.configs.values());
  }

  async reload(): Promise<void> {
    this.configs.clear();
    this.rawCache = null;
    this.initialized = false;
    this.initPromise = null;
    await this.ensureInitialized();
  }
}

// Singleton factory helper
export function createConfigService<T>(
  assetKey: string,
  localFileName: string,
  parser: (content: string) => T,
  processor: (service: ConfigService<T>, data: T) => void
): () => ConfigService<T> {
  let instance: ConfigService<T> | null = null;

  return () => {
    if (!instance) {
      class SpecificConfigService extends ConfigService<T> {
        protected processConfiguration(data: T): void {
          processor(this, data);
        }
      }
      instance = new SpecificConfigService(assetKey, localFileName, parser);
    }
    return instance;
  };
}
```

## Implementation Examples

### Example 1: Feature Flags Service

```typescript
// feature-flags.service.ts
import yaml from 'js-yaml';
import { createConfigService } from './config-service-template.js';

interface FeatureFlags {
  flags: Array<{
    name: string;
    enabled: boolean;
    description?: string;
  }>;
}

export const getFeatureFlagsService = createConfigService<FeatureFlags>(
  process.env.FEATURE_FLAGS_ASSET_KEY || 'settings/feature-flags.yaml',
  'feature-flags.yaml',
  (content) => yaml.load(content) as FeatureFlags,
  (service, data) => {
    // Process the configuration into the service's map
    for (const flag of data.flags) {
      service['configs'].set(flag.name, flag);
    }
  }
);

// Usage in routes
router.get('/flags', async (req, res) => {
  const service = getFeatureFlagsService();
  const flags = await service.getConfig();
  res.json({ flags });
});
```

### Example 2: API Rate Limits Service

```typescript
// api-limits.service.ts
import { createConfigService } from './config-service-template.js';

interface ApiLimits {
  limits: Array<{
    endpoint: string;
    rateLimit: number;
    window: string;
  }>;
}

export const getApiLimitsService = createConfigService<ApiLimits>(
  process.env.API_LIMITS_ASSET_KEY || 'settings/api-limits.json',
  'api-limits.json',
  (content) => JSON.parse(content) as ApiLimits,
  (service, data) => {
    for (const limit of data.limits) {
      service['configs'].set(limit.endpoint, limit);
    }
  }
);
```

### Example 3: Custom Implementation Without Factory

```typescript
// custom-config.service.ts
import { ConfigService } from './config-service-template.js';
import xml2js from 'xml2js';

interface CustomConfig {
  settings: {
    items: Array<{ id: string; value: any }>;
  };
}

class CustomConfigService extends ConfigService<CustomConfig> {
  constructor() {
    super(
      process.env.CUSTOM_CONFIG_ASSET_KEY || 'settings/custom.xml',
      'custom.xml',
      async (content) => {
        const parser = new xml2js.Parser();
        return await parser.parseStringPromise(content);
      }
    );
  }

  protected processConfiguration(data: CustomConfig): void {
    for (const item of data.settings.items) {
      this.configs.set(item.id, item.value);
    }
  }

  // Add custom methods
  async getByPattern(pattern: string): Promise<any[]> {
    await this.ensureInitialized();
    const results = [];
    for (const [key, value] of this.configs) {
      if (key.includes(pattern)) {
        results.push(value);
      }
    }
    return results;
  }
}

// Singleton instance
let instance: CustomConfigService | null = null;
export const getCustomConfigService = (): CustomConfigService => {
  if (!instance) {
    instance = new CustomConfigService();
  }
  return instance;
};
```

## Environment Variables

Each configuration service should use its own environment variable for the asset key:

```bash
# Pattern: {SERVICE_NAME}_ASSET_KEY
LLM_CONFIG_ASSET_KEY=settings/llm-config.yaml
FEATURE_FLAGS_ASSET_KEY=settings/feature-flags.yaml
API_LIMITS_ASSET_KEY=settings/api-limits.json
CUSTOM_CONFIG_ASSET_KEY=settings/custom.xml
```

## Key Design Principles

1. **Independent Caching**: Each service maintains its own `rawCache` for the raw configuration content, completely independent of the generic asset cache.

2. **Lazy Initialization**: Configuration is loaded only when first accessed via `ensureInitialized()`.

3. **Singleton Pattern**: Each configuration service is a singleton to ensure consistency across the application.

4. **Parser Injection**: The parser (YAML, JSON, XML, etc.) is injected at construction time, making the pattern flexible for any file format.

5. **GitHub-First, Local Fallback**: Always attempts to load from GitHub asset service first, falls back to local file for development/testing.

6. **Simple and Lean**: No complex interfaces or unnecessary abstraction layers - just a base class with minimal methods.

## Benefits

- **Minimal Code**: The base template is approximately 100 lines of code
- **Type Safety**: Generic type parameter ensures type safety throughout
- **Independent Caching**: Each configuration maintains its own cache, unaffected by generic cache settings
- **Easy Testing**: Can use local files for testing without GitHub dependency
- **Consistent Pattern**: All configuration services follow the same pattern
- **No External Dependencies**: Only depends on the GitHub asset service
- **Hot Reload Support**: The `reload()` method allows configuration updates without restart

## Usage Guidelines

1. **Create a New Service**: Either use the `createConfigService` factory for simple cases or extend `ConfigService` for more complex scenarios.

2. **Define Types**: Always define TypeScript interfaces for your configuration structure.

3. **Process Configuration**: Implement the `processConfiguration` method to transform raw config data into the internal map structure.

4. **Access Pattern**: Always use the singleton getter function to access the service instance.

5. **Error Handling**: The service will throw errors if configuration cannot be loaded - handle these appropriately in your routes.

6. **Reload Capability**: Use the `reload()` method to refresh configuration without restarting the application.

## Migration from Direct File Access

To migrate existing configuration files to this pattern:

1. Create the type interface for your configuration
2. Choose appropriate parser (YAML, JSON, etc.)
3. Implement the service using the factory or by extending the base class
4. Replace direct file reads with service method calls
5. Add the asset key environment variable
6. Upload configuration file to the GitHub configuration repository

## Best Practices

1. **Logging**: Add appropriate logging for configuration loading and errors
2. **Validation**: Validate configuration structure after parsing
3. **Defaults**: Provide sensible defaults where appropriate
4. **Documentation**: Document the expected configuration structure
5. **Testing**: Create unit tests with mock configurations
6. **Monitoring**: Monitor configuration loading failures in production