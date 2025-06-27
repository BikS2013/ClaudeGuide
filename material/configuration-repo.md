# Configuration Repository Integration

## Overview

This document serves as the source of truth for the design and implementation of using a GitHub repository as an asset provider for application configuration and text-based editable resources.

## Purpose

The configuration repository provides a centralized, version-controlled location for managing application configuration files and other text-based assets. This approach enables:

- Version control for all configuration changes
- Easy rollback capabilities
- Audit trail of configuration modifications
- Separation of configuration from application code
- Multi-environment configuration management

## Design

### Architecture

The system consists of:

1. **GitHub Configuration Repository**: A dedicated repository containing configuration files and assets
2. **PostgreSQL Database**: Caching layer and fallback storage for assets
3. **Asset Service**: A backend service that interfaces with both GitHub API and database
4. **API Endpoint**: REST endpoint for frontend/other services to request configuration assets

### Data Flow

1. **Asset Request**: Client requests an asset via API
2. **Database Check**: System first checks PostgreSQL cache
3. **GitHub Fetch**: If not in database or if GitHub is available, fetch from GitHub
4. **Cache Update**: Store/update asset in database with versioning
5. **Fallback**: If GitHub is unavailable, serve from database cache

### Asset Key Structure

Assets are identified by their full path within the repository:
- Key format: `path/to/file.extension`
- Example: `config/production/api-settings.json`
- Special paths:
  - `settings/` - Directory for application-specific settings
  - `config/` - Directory for environment-specific configurations
  - `templates/` - Directory for reusable templates

### Service Interface

```typescript
interface GitHubAssetService {
  getAsset(key: string, assetCategory?: string): Promise<string>;
  getAssetWithMetadata(key: string, assetCategory?: string): Promise<AssetResponse>;
  listAssets(path?: string): Promise<string[]>;
  clearCache(): void;
  clearCacheForKey(key: string): void;
  isServiceConfigured(): boolean;
}

interface AssetResponse {
  content: string;
  sha: string;
  size: number;
  encoding: string;
  lastModified: string;
}

interface AssetRecord {
  id: string;
  createdAt: Date;
  ownerCategory: string;
  assetCategory: string;
  ownerKey: string;
  assetKey: string;
  description: string;
  data: any;
  dataHash: string;
}
```

### Database Schema

The system uses two tables for asset management:

1. **asset**: Current version of each asset
2. **asset_log**: Historical versions for audit trail

Key fields:
- **owner_category**: Type of application (from ASSET_OWNER_CLASS env var)
- **owner_key**: Application identifier (from ASSET_OWNER_NAME env var)
- **asset_category**: Classification of the asset (optional parameter)
- **asset_key**: GitHub repository path
- **data**: JSONB storage of asset content
- **data_hash**: Hash of content for change detection

### Database Setup Requirements

Before configuring ASSET_DB, ensure:

1. **PostgreSQL Database**: Version 9.5+ (for JSONB support)
2. **Database Creation**: The database must exist before starting the application
3. **Table Creation**: Run the provided SQL scripts to create required tables
4. **User Permissions**: The database user must have:
   - SELECT, INSERT, UPDATE on `asset` table
   - INSERT on `asset_log` table
   - CREATE privilege if you want automatic table creation (not recommended for production)

#### SQL Scripts

```sql
-- Create asset table
CREATE TABLE public.asset (
    id             uuid                     DEFAULT gen_random_uuid() NOT NULL PRIMARY KEY,
    created_at     timestamp with time zone DEFAULT now(),
    owner_category text                     NOT NULL,
    asset_category text                     NOT NULL,
    owner_key      text                     NOT NULL,
    asset_key      text                     NOT NULL,
    description    text                     NOT NULL,
    data           jsonb                    DEFAULT '{}'::jsonb NOT NULL,
    data_hash      text                     NOT NULL
);

-- Create indexes for performance
CREATE INDEX asset_owner_idx ON public.asset (owner_key, asset_key);
CREATE UNIQUE INDEX asset_owner_2_idx ON public.asset (owner_key, asset_key);

-- Create asset_log table for version history
CREATE TABLE public.asset_log (
    id             uuid                     DEFAULT gen_random_uuid() NOT NULL PRIMARY KEY,
    asset_id       uuid                     NOT NULL,
    created_at     timestamp with time zone DEFAULT now(),
    owner_category text                     NOT NULL,
    asset_category text                     NOT NULL,
    owner_key      text                     NOT NULL,
    asset_key      text                     NOT NULL,
    description    text                     NOT NULL,
    data           jsonb                    DEFAULT '{}'::jsonb NOT NULL,
    data_hash      text                     NOT NULL
);

-- Create indexes for performance
CREATE INDEX asset_log_asset_idx ON public.asset_log (asset_id ASC, created_at DESC);
CREATE INDEX asset_log_owner_idx ON public.asset_log (owner_key ASC, asset_key ASC, created_at DESC);
```

## Implementation Details

### Environment Variables

```env
# GitHub Configuration Repository Settings
GITHUB_CONFIG_REPO=owner/repo-name  # Repository in format owner/repo
GITHUB_CONFIG_TOKEN=ghp_xxxx        # Personal access token or GitHub App token
GITHUB_CONFIG_BRANCH=main           # Branch to read from (default: main)

# Database Configuration (optional)
# PostgreSQL connection string for asset caching and fallback
# If not configured, the system operates without database caching (no warnings/errors)
ASSET_DB=postgresql://username:password@host:port/database_name

# Format details:
#   - username: PostgreSQL user with read/write access to the database
#   - password: User's password (URL-encoded if contains special characters)
#   - host: Database server hostname or IP (use 'localhost' for local)
#   - port: PostgreSQL port (default: 5432)
#   - database_name: Name of the database containing asset tables

# Example configurations:
#   Local: postgresql://myuser:mypass@localhost:5432/asset_cache
#   Docker: postgresql://postgres:secret@postgres:5432/assets
#   Remote: postgresql://dbuser:pass123@db.example.com:5432/production_assets

# Asset Owner Configuration (required only if ASSET_DB is configured)
# These identify your application in the asset database for multi-tenant scenarios
ASSET_OWNER_CLASS=application       # Category: 'application', 'service', 'api', etc.
ASSET_OWNER_NAME=owui-feedback     # Unique identifier for this instance

# Asset Memory Cache Configuration (optional)
# Controls in-memory caching behavior for asset retrieval
ASSET_MEMORY_CACHE_ENABLED=true     # Enable/disable in-memory caching (default: true)
ASSET_MEMORY_CACHE_TTL=300          # Cache time-to-live in seconds (default: 300 = 5 minutes)

# Cache behavior notes:
#   - When enabled: Assets are cached in memory for faster retrieval
#   - When disabled: Every request fetches from GitHub/database (useful for development)
#   - TTL: How long cached items remain valid before re-fetching
#   - Set ASSET_MEMORY_CACHE_ENABLED=false for immediate updates during development
#   - Memory cache is cleared on application restart
```

### Service Location

- Service implementation: `backend/src/services/githubAssetService.ts`
- API endpoint: `backend/src/routes/assets.ts`
- Swagger documentation: Available at `/api-docs` under the "Assets" tag

### API Endpoints

#### Get Asset
- **Endpoint**: `GET /api/assets/:key`
- **Description**: Retrieves the content of a specific asset
- **Parameters**: 
  - `key`: URL-encoded file path within the repository
- **Query Parameters**:
  - `category` (optional): Asset category for classification in database
- **Response**: Plain text content of the file with appropriate Content-Type header
- **Content-Type Detection**: Automatic based on file extension (json, yaml, xml, html, css, js, md)
- **Example**: `GET /api/assets/config%2Fapi-settings.json?category=configuration`
- **Status Codes**:
  - 200: Success (from GitHub or database cache)
  - 400: Asset key is missing
  - 401: Authentication failed
  - 404: Asset not found
  - 429: Rate limit exceeded
  - 500: Server error
  - 503: Service not configured

#### List Assets
- **Endpoint**: `GET /api/assets`
- **Description**: Lists available assets in a directory
- **Query Parameters**:
  - `path`: Directory path to list (optional, defaults to root)
- **Response**: JSON array of asset keys
- **Example Response**:
  ```json
  [
    "config/production/api-settings.json",
    "config/staging/api-settings.json",
    "templates/email-template.html"
  ]
  ```
- **Status Codes**:
  - 200: Success
  - 500: Server error
  - 503: Service not configured

#### Get Asset with Metadata
- **Endpoint**: `GET /api/assets/:key/metadata`
- **Description**: Retrieves an asset with its metadata
- **Parameters**:
  - `key`: URL-encoded file path within the repository
- **Query Parameters**:
  - `category` (optional): Asset category for classification in database
- **Response**: JSON object with content and metadata
- **Example Response**:
  ```json
  {
    "content": "file content here...",
    "sha": "abc123...",
    "size": 1234,
    "encoding": "base64",
    "lastModified": "2025-01-09T10:30:00Z",
    "source": "github",  // or "database" if served from cache
    "databaseVersion": "2025-01-09T10:25:00Z"  // if exists in database
  }
  ```
- **Status Codes**:
  - 200: Success
  - 400: Asset key is missing
  - 404: Asset not found
  - 500: Server error
  - 503: Service not configured

#### Clear Cache
- **Endpoint**: `POST /api/assets/cache/clear`
- **Description**: Clears the asset cache
- **Request Body** (optional):
  ```json
  {
    "key": "config/production/api-settings.json"
  }
  ```
- **Response**: JSON message confirming cache clear
- **Status Codes**:
  - 200: Success
  - 500: Server error
  - 503: Service not configured

### Route Order (Critical)

The routes must be defined in the following order to ensure proper matching:
1. `GET /api/assets/` - List assets (most specific)
2. `POST /api/assets/cache/clear` - Clear cache
3. `GET /api/assets/:key/metadata` - Get with metadata
4. `GET /api/assets/:key` - Get asset content (wildcard - least specific)

This order prevents the wildcard route from catching requests meant for other endpoints.

### Security Considerations

1. **Token Management**: 
   - GitHub access token must have read-only access to the configuration repository
   - Token is stored as environment variable and never exposed in responses
   - Debug logs show only last 4 characters of token

2. **Access Control**: 
   - API endpoints should implement appropriate authentication/authorization
   - Service availability check via `isServiceConfigured()` method

3. **Rate Limiting**: 
   - GitHub API has rate limits (60/hour unauthenticated, 5000/hour authenticated)
   - Service returns 429 status code when rate limit is exceeded

4. **Caching**: 
   - In-memory cache with 5-minute TTL to reduce API calls
   - Cache can be cleared on-demand via API endpoint

### Error Handling

The service handles the following error scenarios:
- **Configuration Missing**: Returns 503 with clear instructions to set environment variables
- **Invalid GitHub Token**: Returns 401 with authentication error
- **Repository Not Found**: Included in 404 responses
- **Asset Not Found**: Returns 404 with specific error message
- **Network Failures**: Returns 500 with generic error
- **GitHub API Rate Limit**: Returns 429 with rate limit message

### Caching Strategy

Multi-layer caching implementation:

1. **In-memory cache**: 
   - JavaScript Map with timestamp tracking
   - Configurable via environment variables:
     - `ASSET_MEMORY_CACHE_ENABLED`: Enable/disable caching (default: true)
     - `ASSET_MEMORY_CACHE_TTL`: TTL in seconds (default: 300 = 5 minutes)
   - Format: `{repo}:{branch}:{asset-key}`
   - Can be disabled for development to see immediate updates

2. **Database cache**:
   - PostgreSQL persistent storage
   - Versioning with asset_log table
   - Hash-based change detection
   - Serves as fallback when GitHub is unavailable

3. **Cache Flow**:
   - Check in-memory cache first (fastest)
   - Query database if not in memory
   - Fetch from GitHub if available
   - Update database with new/changed content
   - Serve from database if GitHub fails

4. **Invalidation**:
   - In-memory: After TTL or manual clear
   - Database: Never invalidated, only updated
   - Manual: Via `/api/assets/cache/clear` endpoint

### Database Integration Workflow

1. **Asset Retrieval Process**:
   ```
   Request Asset → Check Memory Cache → Check Database → Fetch from GitHub
                                      ↓                 ↓
                                   Found            Not Found/Changed
                                      ↓                 ↓
                                   Return         Store/Update in DB
                                                        ↓
                                                     Return
   ```

2. **Change Detection**:
   - Calculate SHA-256 hash of content
   - Compare with stored hash
   - If different, archive old version to asset_log
   - Update asset table with new content

3. **Fallback Mechanism**:
   - If GitHub unavailable, serve from database
   - Log warning about fallback mode
   - Continue normal operations

4. **Data Storage**:
   - Content stored as JSONB for flexibility
   - Supports any text-based format
   - Maintains original content structure

### Service Initialization

The service uses lazy initialization to avoid startup failures:
- Service instance created only when first accessed
- Environment variables checked at runtime, not module load time
- Graceful degradation with 503 status when GitHub not configured
- Database connection pooling for performance (when configured)
- **Silent operation**: If ASSET_DB is not configured, the system operates without database caching with no console warnings or errors

## Usage Examples

### Retrieving Configuration File

```javascript
// Frontend usage
const response = await fetch('/api/assets/config%2Fproduction%2Fapi-settings.json');
const config = await response.json();

// Backend usage
const assetService = new GitHubAssetService();
const content = await assetService.getAsset('config/production/api-settings.json');
```

### Listing Available Configurations

```javascript
const response = await fetch('/api/assets?path=config');
const assets = await response.json();
// Returns: ['config/production/api-settings.json', 'config/staging/api-settings.json', ...]
```

## Troubleshooting

### Common Issues

1. **503 Service Unavailable**
   - Cause: Environment variables not set
   - Solution: Set `GITHUB_CONFIG_REPO` and `GITHUB_CONFIG_TOKEN` in `.env` file

2. **404 Asset Not Found**
   - Cause: Wrong file path or file doesn't exist in repository
   - Solution: Verify the file exists at the specified path in the GitHub repository

3. **401 Authentication Failed**
   - Cause: Invalid or expired GitHub token
   - Solution: Generate a new personal access token with repo read permissions

4. **Route Not Found**
   - Cause: Incorrect route order in implementation
   - Solution: Ensure routes are defined in the correct order (most specific first)

5. **Database Connection Issues** (when ASSET_DB is configured)
   - **"Asset Database: ❌ Not configured"** despite having ASSET_DB set:
     - Check connection string format is correct
     - Verify all required components are present (username, password, host, port, database)
   - **Silent failures during operation**:
     - Database is unreachable but system continues with GitHub-only mode
     - Check PostgreSQL server is running and accessible
     - Verify firewall/network settings allow connection
   - **Missing tables error** (visible in PostgreSQL logs):
     - Run the SQL scripts to create `asset` and `asset_log` tables
     - Ensure user has proper permissions

6. **Cache-Related Issues**
   - **Changes not reflected immediately**:
     - In-memory cache is enabled by default with 5-minute TTL
     - Set `ASSET_MEMORY_CACHE_ENABLED=false` in `.env` for development
     - Or use the cache clear endpoint: `POST /api/assets/cache/clear`
   - **Performance concerns**:
     - Increase `ASSET_MEMORY_CACHE_TTL` for less frequent GitHub API calls
     - Monitor GitHub API rate limits if cache is disabled
   - **Memory usage**:
     - Large assets cached in memory may increase application memory footprint
     - Consider reducing TTL or disabling cache for very large assets

### Debug Mode

Enable debug logging by checking console output which shows:
- Repository configuration status
- Token presence (last 4 characters only)
- Branch being used
- Service configuration state
- Database configuration status (on startup)
- Memory cache configuration (enabled/disabled and TTL)

### Database Connection String Examples

Valid PostgreSQL connection string formats:

```bash
# Standard format
postgresql://username:password@hostname:port/database

# Local development
postgresql://devuser:devpass@localhost:5432/asset_cache

# Docker container
postgresql://postgres:mysecretpassword@postgres_container:5432/assets

# Remote server with SSL
postgresql://produser:prodpass@db.example.com:5432/production_assets?sslmode=require

# Special characters in password (URL-encoded)
postgresql://user:p%40ssw0rd%21@localhost:5432/mydb  # Password: p@ssw0rd!

# IPv6 address
postgresql://user:pass@[2001:db8::1234]:5432/database

# Unix socket (local only)
postgresql://user:pass@/database?host=/var/run/postgresql
```

Common encoding for special characters:
- `@` → `%40`
- `!` → `%21`
- `#` → `%23`
- `$` → `%24`
- `&` → `%26`
- `'` → `%27`
- `(` → `%28`
- `)` → `%29`

## Best Practices

1. **Repository Structure**:
   ```
   config/
   ├── production/
   │   ├── api-settings.json
   │   └── database.yaml
   ├── staging/
   │   ├── api-settings.json
   │   └── database.yaml
   └── development/
       ├── api-settings.json
       └── database.yaml
   settings/
   ├── feature-flags.json
   └── ui-config.json
   templates/
   ├── email-welcome.html
   └── report-template.md
   ```

2. **Naming Conventions**:
   - Use lowercase with hyphens for file names
   - Group related configurations in directories
   - Use clear, descriptive names

3. **Version Control**:
   - Tag releases for configuration snapshots
   - Use meaningful commit messages
   - Document configuration changes in commit descriptions

4. **Security**:
   - Never commit sensitive data (use environment variables)
   - Rotate access tokens regularly
   - Use branch protection for production configurations

## Future Enhancements

1. **Webhook Integration**: Automatic cache invalidation on repository changes
2. **Multi-Branch Support**: Ability to read from different branches for different environments
3. **Asset Validation**: Schema validation for JSON/YAML configuration files
4. **Encryption Support**: For sensitive configuration values
5. **Bulk Operations**: Retrieve multiple assets in a single request
6. **Change Notifications**: WebSocket support for real-time configuration updates
7. **Redis Cache Integration**: For distributed caching in multi-instance deployments
8. **Configuration Hot Reload**: Automatic application configuration updates without restart

## Migration Guide

When migrating existing configurations to the GitHub repository:

1. **Preparation**:
   - Audit existing configuration files
   - Identify environment-specific settings
   - Plan directory structure

2. **Repository Setup**:
   - Create new GitHub repository
   - Set repository to private (if needed)
   - Create initial directory structure
   - Add README documenting the structure

3. **Security Configuration**:
   - Generate personal access token with minimal permissions (repo:read)
   - Set up branch protection rules for main/production branches
   - Configure CODEOWNERS file for approval workflows

4. **Application Updates**:
   - Add environment variables to `.env`:
     ```
     GITHUB_CONFIG_REPO=owner/config-repo
     GITHUB_CONFIG_TOKEN=ghp_xxxxxxxxxxxx
     GITHUB_CONFIG_BRANCH=main
     ```
   - Update application code to use asset service
   - Implement fallback mechanism during transition

5. **Testing**:
   - Test each configuration file retrieval
   - Verify content-type headers are correct
   - Test cache clearing functionality
   - Validate error handling for missing files

6. **Deployment**:
   - Deploy with dual configuration support (local + GitHub)
   - Monitor for any issues
   - Gradually migrate all configurations
   - Remove local configuration files once stable