# NBG OAuth Client Integration Guide

This guide explains how to implement the client-side flow for NBG OAuth authentication.

## OAuth Flow Overview

```
1. User clicks "Login" → 2. Get auth URL from backend → 3. Redirect to NBG
    ↓
4. User logs in at NBG → 5. NBG redirects back → 6. Backend exchanges code for token
    ↓
7. Client receives token → 8. Client stores token → 9. Client includes token in API calls
```

## Frontend Implementation

### 1. Login Button/Component

```javascript
// React example
const LoginButton = () => {
  const handleLogin = async () => {
    try {
      // Step 1: Get auth URL from backend
      const response = await fetch('/api/auth/login');
      const { authUrl, state, nonce } = await response.json();
      
      // Step 2: Store state in sessionStorage for CSRF protection
      sessionStorage.setItem('oauth_state', state);
      sessionStorage.setItem('oauth_nonce', nonce);
      
      // Step 3: Redirect to NBG Identity Server
      window.location.href = authUrl;
    } catch (error) {
      console.error('Login failed:', error);
    }
  };

  return <button onClick={handleLogin}>Login with NBG</button>;
};
```

### 2. OAuth Callback Handler

Create a component/page that handles the OAuth callback:

```javascript
// React Router example: /callback route
const OAuthCallback = () => {
  const [status, setStatus] = useState('Processing...');
  const navigate = useNavigate();

  useEffect(() => {
    handleOAuthCallback();
  }, []);

  const handleOAuthCallback = async () => {
    // Get query parameters
    const params = new URLSearchParams(window.location.search);
    const code = params.get('code');
    const state = params.get('state');
    const error = params.get('error');

    // Check for errors
    if (error) {
      setStatus(`Login failed: ${params.get('error_description') || error}`);
      return;
    }

    // Verify state for CSRF protection
    const savedState = sessionStorage.getItem('oauth_state');
    if (state !== savedState) {
      setStatus('Invalid state - possible CSRF attack');
      return;
    }

    try {
      // Exchange code for tokens
      const response = await fetch(`/api/auth/callback?${params.toString()}`);
      const data = await response.json();

      if (data.success) {
        // Store tokens
        localStorage.setItem('access_token', data.access_token);
        localStorage.setItem('refresh_token', data.refresh_token);
        localStorage.setItem('id_token', data.id_token);
        
        // Clean up
        sessionStorage.removeItem('oauth_state');
        sessionStorage.removeItem('oauth_nonce');
        
        // Redirect to main app
        navigate('/dashboard');
      } else {
        setStatus('Authentication failed');
      }
    } catch (error) {
      setStatus('Error processing authentication');
    }
  };

  return <div>{status}</div>;
};
```

### 3. Configure Routes

Make sure your frontend router handles the callback URL:

```javascript
// React Router example
<Routes>
  <Route path="/" element={<HomePage />} />
  <Route path="/signin-nbg" element={<OAuthCallback />} />
  <Route path="/dashboard" element={<ProtectedDashboard />} />
</Routes>
```

### 4. API Request Interceptor

Add the token to all API requests:

```javascript
// Axios example
import axios from 'axios';

// Create axios instance
const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL || 'http://localhost:3001'
});

// Request interceptor
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('access_token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor for token refresh
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        const refreshToken = localStorage.getItem('refresh_token');
        const response = await fetch('/api/auth/refresh', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ refresh_token: refreshToken })
        });

        if (response.ok) {
          const data = await response.json();
          localStorage.setItem('access_token', data.access_token);
          originalRequest.headers.Authorization = `Bearer ${data.access_token}`;
          return api(originalRequest);
        }
      } catch (refreshError) {
        // Refresh failed, redirect to login
        window.location.href = '/login';
      }
    }

    return Promise.reject(error);
  }
);
```

### 5. Auth Context/Hook (React)

```javascript
// authContext.js
import React, { createContext, useContext, useState, useEffect } from 'react';

const AuthContext = createContext();

export const useAuth = () => useContext(AuthContext);

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    checkAuthStatus();
  }, []);

  const checkAuthStatus = async () => {
    const token = localStorage.getItem('access_token');
    if (!token) {
      setLoading(false);
      return;
    }

    try {
      const response = await fetch('/api/auth/status', {
        headers: { Authorization: `Bearer ${token}` }
      });

      if (response.ok) {
        const data = await response.json();
        if (data.authenticated) {
          setUser(data.user);
        }
      }
    } catch (error) {
      console.error('Auth check failed:', error);
    } finally {
      setLoading(false);
    }
  };

  const login = () => {
    window.location.href = '/api/auth/login';
  };

  const logout = async () => {
    const idToken = localStorage.getItem('id_token');
    
    // Get logout URL
    const response = await fetch(`/api/auth/logout?id_token=${idToken}`);
    const { logoutUrl } = await response.json();
    
    // Clear local storage
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    localStorage.removeItem('id_token');
    
    // Redirect to NBG logout
    window.location.href = logoutUrl;
  };

  return (
    <AuthContext.Provider value={{ user, loading, login, logout, checkAuthStatus }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### 6. Protected Route Component

```javascript
const ProtectedRoute = ({ children }) => {
  const { user, loading } = useAuth();
  
  if (loading) return <div>Loading...</div>;
  
  if (!user) {
    return <Navigate to="/login" />;
  }
  
  return children;
};

// Usage
<ProtectedRoute>
  <Dashboard />
</ProtectedRoute>
```

## Complete Example: Login Flow

### Step 1: User Clicks Login

```javascript
// LoginPage.jsx
const LoginPage = () => {
  const [loading, setLoading] = useState(false);

  const handleLogin = async () => {
    setLoading(true);
    try {
      // Get auth URL from your backend
      const response = await fetch('/api/auth/login');
      const { authUrl } = await response.json();
      
      // Redirect to NBG
      window.location.href = authUrl;
    } catch (error) {
      console.error('Login initiation failed:', error);
      setLoading(false);
    }
  };

  return (
    <div className="login-container">
      <h1>Welcome</h1>
      <button onClick={handleLogin} disabled={loading}>
        {loading ? 'Redirecting...' : 'Login with NBG'}
      </button>
    </div>
  );
};
```

### Step 2: NBG Redirects Back

After successful authentication, NBG redirects to: `https://yourapp.com/signin-nbg?code=xxx&state=yyy`

### Step 3: Your App Handles Callback

The backend route `/signin-nbg` redirects to `/api/auth/callback` which exchanges the code for tokens.

### Step 4: Frontend Receives Tokens

Your callback component receives and stores the tokens.

## Environment Configuration

```javascript
// .env (React)
REACT_APP_API_URL=http://localhost:3001
REACT_APP_AUTH_CALLBACK_URL=/signin-nbg
```

## Security Considerations

1. **Always use HTTPS** in production
2. **Store tokens securely**:
   - Consider using HttpOnly cookies instead of localStorage
   - If using localStorage, ensure XSS protection
3. **Implement CSRF protection** using the state parameter
4. **Handle token expiration** gracefully
5. **Clear tokens on logout**

## Troubleshooting

### Common Issues

1. **CORS errors**: Ensure your backend allows your frontend origin
2. **Redirect URI mismatch**: The callback URL must exactly match what's registered with NBG
3. **State mismatch**: Ensure state is properly stored and retrieved
4. **Token expiration**: Implement proper token refresh logic

### Debug Checklist

- [ ] Is the frontend origin in CORS_ORIGINS?
- [ ] Is the redirect URI registered with NBG?
- [ ] Are you storing/retrieving state correctly?
- [ ] Is the Authorization header format correct?
- [ ] Are tokens being refreshed before expiration?

## Full Integration Example

```javascript
// App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { AuthProvider } from './contexts/AuthContext';

function App() {
  return (
    <BrowserRouter>
      <AuthProvider>
        <Routes>
          <Route path="/login" element={<LoginPage />} />
          <Route path="/signin-nbg" element={<OAuthCallback />} />
          <Route path="/" element={
            <ProtectedRoute>
              <Dashboard />
            </ProtectedRoute>
          } />
        </Routes>
      </AuthProvider>
    </BrowserRouter>
  );
}
```

This creates a complete OAuth flow where:
1. Unauthenticated users are redirected to login
2. Login redirects to NBG
3. NBG redirects back to your app
4. Tokens are stored and used for API calls
5. Token refresh happens automatically
6. Logout clears tokens and ends NBG session