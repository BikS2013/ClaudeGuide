# NBG OAuth Backend Integration Guide

A comprehensive guide for implementing NBG OAuth authentication in Node.js/Express backends.

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Installation](#installation)
4. [Core Components](#core-components)
5. [Step-by-Step Implementation](#step-by-step-implementation)
6. [Configuration](#configuration)
7. [API Reference](#api-reference)
8. [Security Best Practices](#security-best-practices)
9. [Troubleshooting](#troubleshooting)
10. [Migration Guide](#migration-guide)

## Overview

NBG OAuth follows the OpenID Connect Authorization Code flow with specific requirements:
- Callback URLs must end with `/signin-nbg`
- Logout URLs must end with `/signout-callback-nbg`
- Supports reference token exchange (NBG-specific feature)
- Requires registration with NBG Identity Team (ids.tech@nbg.gr)

### Architecture Diagram

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Frontend  │────▶│   Backend   │────▶│ NBG Identity│
│  Application│◀────│   Express   │◀────│   Server    │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                    ┌──────▼──────┐
                    │   Database  │
                    │  (Optional) │
                    └─────────────┘
```

## Prerequisites

- Node.js 16+ with Express
- NBG Client ID and Secret (from ids.tech@nbg.gr)
- Registered callback URLs with NBG
- TypeScript (recommended)

## Installation

### 1. Install Required Packages

```bash
npm install jsonwebtoken jwks-rsa express cors dotenv
npm install --save-dev @types/jsonwebtoken @types/express
```

### 2. Directory Structure

```
backend/
├── src/
│   ├── middleware/
│   │   ├── nbg-auth.config.ts      # Configuration management
│   │   ├── token-validator.ts      # JWT validation
│   │   └── auth-helpers.ts         # OAuth utilities
│   ├── routes/
│   │   └── auth.routes.ts          # Auth endpoints
│   ├── types/
│   │   └── auth.types.ts           # TypeScript definitions
│   └── index.ts                    # Main application
├── .env
└── .env.example
```

## Core Components

### 1. Type Definitions (types/auth.types.ts)

```typescript
export interface NBGTokenPayload {
  sub: string;                    // Subject (user ID)
  name?: string;                  // Full name
  given_name?: string;            // First name
  family_name?: string;           // Last name
  email?: string;                 // Email address
  role?: string | string[];       // User roles
  iss: string;                    // Issuer
  aud: string | string[];         // Audience
  exp: number;                    // Expiration time
  iat: number;                    // Issued at
  nbf?: number;                   // Not before
  nonce?: string;                 // Nonce for replay protection
  sid?: string;                   // Session ID
  auth_time?: number;             // Authentication time
  idp?: string;                   // Identity provider
  amr?: string[];                 // Authentication methods
}

export interface AuthenticatedRequest extends Request {
  auth?: NBGTokenPayload;
  user?: NBGUserInfo;
  token?: string;
}
```

### 2. Configuration Module (middleware/nbg-auth.config.ts)

```typescript
import { NBGOAuthConfig } from '../types/auth.types.js';

export function getNBGOAuthConfig(): NBGOAuthConfig {
  const issuer = process.env.NBG_OAUTH_ISSUER;
  const clientId = process.env.NBG_CLIENT_ID;
  const clientSecret = process.env.NBG_CLIENT_SECRET;
  const baseUrl = process.env.BASE_URL || `http://localhost:${process.env.PORT || 3001}`;

  if (!issuer || !clientId || !clientSecret) {
    throw new Error('NBG OAuth configuration missing');
  }

  return {
    issuer,
    clientId,
    clientSecret,
    redirectUri: `${baseUrl}/signin-nbg`,
    postLogoutRedirectUri: `${baseUrl}/signout-callback-nbg`,
    scopes: (process.env.NBG_OAUTH_SCOPES || 'openid profile email').split(' '),
    jwksUri: `${issuer}/.well-known/jwks.json`,
    tokenEndpoint: `${issuer}/connect/token`,
    authorizationEndpoint: `${issuer}/connect/authorize`,
    userInfoEndpoint: `${issuer}/connect/userinfo`,
    introspectionEndpoint: `${issuer}/connect/introspect`,
    discoveryEndpoint: `${issuer}/.well-known/openid-configuration`
  };
}

export function isAuthEnabled(): boolean {
  return process.env.NBG_OAUTH_ENABLED !== 'false';
}
```

### 3. Token Validation Middleware (middleware/token-validator.ts)

```typescript
import jwt from 'jsonwebtoken';
import jwksClient from 'jwks-rsa';

// JWKS client for key retrieval
const getJwksClient = () => jwksClient({
  jwksUri: getNBGOAuthConfig().jwksUri!,
  cache: true,
  cacheMaxEntries: 5,
  cacheMaxAge: 600000, // 10 minutes
  rateLimit: true,
  jwksRequestsPerMinute: 10
});

// Main validation middleware
export function requireAuth(
  req: AuthenticatedRequest,
  res: Response,
  next: NextFunction
): void {
  const authHeader = req.headers.authorization;
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    res.status(401).json({ error: 'Authorization required' });
    return;
  }

  const token = authHeader.split(' ')[1];
  
  validateNBGToken(token)
    .then(payload => {
      req.auth = payload;
      req.token = token;
      next();
    })
    .catch(error => {
      res.status(401).json({ error: 'Invalid token' });
    });
}

// Role-based access control
export function requireRole(...roles: string[]) {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    if (!req.auth) {
      res.status(401).json({ error: 'Authentication required' });
      return;
    }

    const userRoles = Array.isArray(req.auth.role) 
      ? req.auth.role 
      : req.auth.role ? [req.auth.role] : [];

    if (!roles.some(role => userRoles.includes(role))) {
      res.status(403).json({ error: 'Insufficient permissions' });
      return;
    }

    next();
  };
}
```

### 4. Authentication Routes (routes/auth.routes.ts)

```typescript
import { Router } from 'express';
import { v4 as uuidv4 } from 'uuid';

const router = Router();

// Get authentication status
router.get('/status', optionalAuth, async (req, res) => {
  if (!req.auth) {
    return res.json({ authenticated: false });
  }
  
  const userInfo = await fetchUserInfo(req.token!);
  res.json({
    authenticated: true,
    user: userInfo,
    token: {
      expiresAt: new Date(req.auth.exp * 1000),
      issuer: req.auth.iss
    }
  });
});

// Initiate login
router.get('/login', (req, res) => {
  const state = uuidv4();
  const nonce = uuidv4();
  const authUrl = getAuthorizationUrl(state, nonce);
  
  res.json({ authUrl, state, nonce });
});

// Handle OAuth callback
router.get('/callback', async (req, res) => {
  const { code, state, error } = req.query;
  
  if (error) {
    return res.status(400).json({ error });
  }
  
  try {
    const tokens = await exchangeCodeForTokens(code as string);
    res.json({
      success: true,
      ...tokens,
      state
    });
  } catch (error) {
    res.status(500).json({ error: 'Authentication failed' });
  }
});

export default router;
```

## Step-by-Step Implementation

### Step 1: Environment Setup

Create `.env` file:
```env
# NBG OAuth Configuration
NBG_OAUTH_ENABLED=true
NBG_OAUTH_ISSUER=https://identity.nbg.gr
NBG_CLIENT_ID=your-client-id
NBG_CLIENT_SECRET=your-client-secret
NBG_OAUTH_SCOPES=openid profile email api1
BASE_URL=http://localhost:3001

# Optional Configuration
PUBLIC_ENDPOINTS=/health,/api/status
NBG_VALIDATE_ISSUER=true
NBG_VALIDATE_AUDIENCE=true
NBG_VALIDATE_EXPIRY=true
NBG_CLOCK_TOLERANCE=5
```

### Step 2: Update Express Application

```typescript
import express from 'express';
import authRouter from './routes/auth.routes.js';
import { globalAuthMiddleware } from './middleware/auth-helpers.js';
import { requireAuth } from './middleware/token-validator.js';

const app = express();

// Middleware
app.use(express.json());
app.use(cors({
  origin: process.env.CORS_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true,
  allowedHeaders: ['Content-Type', 'Authorization']
}));

// Auth routes (must be first)
app.use('/api/auth', authRouter);

// NBG callback routes
app.get('/signin-nbg', (req, res) => {
  const queryString = req.url.split('?')[1] || '';
  res.redirect(`/api/auth/callback${queryString ? '?' + queryString : ''}`);
});

app.get('/signout-callback-nbg', (req, res) => {
  const queryString = req.url.split('?')[1] || '';
  res.redirect(`/api/auth/logout-callback${queryString ? '?' + queryString : ''}`);
});

// Apply global auth check
app.use(globalAuthMiddleware);

// Protected routes
app.use('/api/users', requireAuth, userRouter);
app.use('/api/data', requireAuth, dataRouter);

// Public routes
app.get('/health', (req, res) => res.json({ status: 'ok' }));
```

### Step 3: Implement Token Exchange

```typescript
async function exchangeCodeForTokens(code: string): Promise<TokenResponse> {
  const config = getNBGOAuthConfig();
  
  const response = await fetch(config.tokenEndpoint, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
      'Accept': 'application/json'
    },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri: config.redirectUri,
      client_id: config.clientId,
      client_secret: config.clientSecret
    })
  });
  
  if (!response.ok) {
    throw new Error('Token exchange failed');
  }
  
  return response.json();
}
```

### Step 4: Add User Context to Routes

```typescript
// Example protected route with user context
app.get('/api/profile', requireAuth, async (req: AuthenticatedRequest, res) => {
  const userId = req.auth!.sub;
  const userEmail = req.auth!.email;
  
  // Fetch user-specific data
  const userData = await getUserData(userId);
  
  res.json({
    id: userId,
    email: userEmail,
    data: userData
  });
});

// Role-based endpoint
app.get('/api/admin', requireAuth, requireRole('admin'), (req, res) => {
  res.json({ message: 'Admin access granted' });
});
```

## Configuration

### Required Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `NBG_OAUTH_ISSUER` | NBG Identity Server URL | `https://identity.nbg.gr` |
| `NBG_CLIENT_ID` | Your application's client ID | `my-app-client` |
| `NBG_CLIENT_SECRET` | Your application's client secret | `secret-value` |
| `NBG_OAUTH_SCOPES` | Required OAuth scopes | `openid profile email` |
| `BASE_URL` | Your application's base URL | `https://myapp.com` |

### Optional Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `NBG_OAUTH_ENABLED` | Enable/disable auth | `true` |
| `PUBLIC_ENDPOINTS` | Comma-separated public paths | `/health` |
| `NBG_VALIDATE_ISSUER` | Validate token issuer | `true` |
| `NBG_VALIDATE_AUDIENCE` | Validate token audience | `true` |
| `NBG_CLOCK_TOLERANCE` | Clock skew tolerance (seconds) | `5` |

## API Reference

### Authentication Endpoints

#### GET /api/auth/status
Check authentication status and get user info.

**Headers:**
- `Authorization: Bearer <token>` (optional)

**Response:**
```json
{
  "authenticated": true,
  "user": {
    "id": "user123",
    "name": "John Doe",
    "email": "john@example.com",
    "roles": ["user", "admin"]
  }
}
```

#### GET /api/auth/login
Initiate OAuth login flow.

**Response:**
```json
{
  "authUrl": "https://identity.nbg.gr/connect/authorize?...",
  "state": "random-state-value",
  "nonce": "random-nonce-value"
}
```

#### GET /api/auth/callback
Handle OAuth callback (internal use).

**Query Parameters:**
- `code`: Authorization code from NBG
- `state`: State parameter for CSRF protection

#### POST /api/auth/refresh
Refresh access token.

**Body:**
```json
{
  "refresh_token": "your-refresh-token"
}
```

#### POST /api/auth/exchange-token
Exchange reference token for JWT (NBG-specific).

**Body:**
```json
{
  "token": "reference-token",
  "scopes": ["api1", "api2"]
}
```

### Middleware Functions

#### requireAuth
Requires valid authentication token.

```typescript
app.get('/protected', requireAuth, handler);
```

#### optionalAuth
Validates token if present but doesn't require it.

```typescript
app.get('/mixed', optionalAuth, handler);
```

#### requireRole(...roles)
Requires specific user roles.

```typescript
app.get('/admin', requireAuth, requireRole('admin'), handler);
```

## Security Best Practices

### 1. Token Storage
- Never log tokens in production
- Use secure session storage if needed
- Implement token rotation

### 2. HTTPS Requirements
```typescript
// Force HTTPS in production
if (process.env.NODE_ENV === 'production') {
  app.use((req, res, next) => {
    if (req.header('x-forwarded-proto') !== 'https') {
      res.redirect(`https://${req.header('host')}${req.url}`);
    } else {
      next();
    }
  });
}
```

### 3. Rate Limiting
```typescript
import rateLimit from 'express-rate-limit';

const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5 // limit each IP to 5 requests per windowMs
});

app.use('/api/auth/login', authLimiter);
```

### 4. CSRF Protection
- Always validate the `state` parameter
- Use cryptographically secure random values
- Store state server-side when possible

### 5. Token Validation
```typescript
// Validate all required claims
const validationOptions = {
  validateIssuer: true,
  validateAudience: true,
  validateExpiry: true,
  requiredClaims: ['sub', 'email']
};
```

## Troubleshooting

### Common Issues and Solutions

#### 1. "NBG OAuth configuration missing"
**Cause:** Missing environment variables
**Solution:** Ensure all required env vars are set:
```bash
NBG_OAUTH_ISSUER=https://identity.nbg.gr
NBG_CLIENT_ID=your-client-id
NBG_CLIENT_SECRET=your-client-secret
```

#### 2. "Invalid redirect_uri"
**Cause:** Callback URL not registered with NBG
**Solution:** 
- Ensure URL ends with `/signin-nbg`
- Register exact URL with NBG Identity Team
- Check for http vs https mismatch

#### 3. "Token validation failed"
**Cause:** Various token issues
**Debug steps:**
```typescript
// Add debug logging
const payload = jwt.decode(token);
console.log('Token claims:', payload);
console.log('Token expiry:', new Date(payload.exp * 1000));
console.log('Current time:', new Date());
```

#### 4. "JWKS endpoint unreachable"
**Cause:** Network or firewall issues
**Solution:**
- Check firewall rules
- Verify NBG Identity Server is accessible
- Check proxy settings if applicable

### Debug Mode

Enable detailed logging in development:
```typescript
if (process.env.NODE_ENV === 'development') {
  // Log all auth attempts
  app.use((req, res, next) => {
    if (req.headers.authorization) {
      console.log('Auth attempt:', {
        path: req.path,
        hasToken: !!req.headers.authorization
      });
    }
    next();
  });
}
```

## Migration Guide

### From Session-Based Auth

1. **Remove session middleware:**
```typescript
// Remove
app.use(session({ ... }));

// Add
app.use('/api/auth', authRouter);
```

2. **Update protected routes:**
```typescript
// Before
app.get('/api/data', ensureAuthenticated, handler);

// After
app.get('/api/data', requireAuth, handler);
```

3. **Update user context:**
```typescript
// Before
const userId = req.session.userId;

// After
const userId = req.auth.sub;
```

### From Another OAuth Provider

1. **Update configuration:**
   - Change issuer URL
   - Update callback paths to NBG format
   - Adjust token validation

2. **Handle NBG-specific features:**
   - Reference token exchange
   - Specific claim mappings
   - Required callback URL format

## Production Checklist

- [ ] All environment variables configured
- [ ] HTTPS enabled and enforced
- [ ] Rate limiting implemented
- [ ] Error logging configured
- [ ] Token refresh implemented
- [ ] CORS properly configured
- [ ] Security headers added
- [ ] Monitoring/alerting setup
- [ ] Backup authentication method (if needed)
- [ ] Documentation updated

## Support and Resources

- **NBG Identity Team:** ids.tech@nbg.gr
- **OpenID Connect Spec:** https://openid.net/connect/
- **JWT Debugger:** https://jwt.io/
- **JWKS Validator:** https://www.jsonwebtoken.io/

## Version History

- **1.0.0** - Initial NBG OAuth implementation
- **1.1.0** - Added reference token exchange
- **1.2.0** - Added role-based access control
- **1.3.0** - Added automatic token refresh