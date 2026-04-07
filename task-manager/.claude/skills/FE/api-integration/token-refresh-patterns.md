# Token Refresh Patterns

Strategies for handling authentication token expiration and refresh in frontend API clients.

## Core Problem

JWT access tokens expire (typically 15min-1hr). When a request fails with 401, the client must:

1. Detect token expiration vs invalid credentials
2. Refresh the token without race conditions
3. Retry the original request(s) with new token
4. Handle refresh failure gracefully

## Pattern 1: Interceptor-Based Refresh

### Implementation

```typescript
interface TokenPair {
  accessToken: string;
  refreshToken: string;
  expiresAt: number; // Unix timestamp
}

class TokenManager {
  private tokens: TokenPair | null = null;
  private refreshPromise: Promise<TokenPair> | null = null;

  getAccessToken(): string | null {
    return this.tokens?.accessToken ?? null;
  }

  setTokens(tokens: TokenPair): void {
    this.tokens = tokens;
  }

  clearTokens(): void {
    this.tokens = null;
    this.refreshPromise = null;
  }

  async refreshIfNeeded(): Promise<string> {
    // If refresh already in progress, wait for it
    if (this.refreshPromise) {
      const tokens = await this.refreshPromise;
      return tokens.accessToken;
    }

    // Check if token is expired or expiring soon (5min buffer)
    const now = Date.now();
    if (this.tokens && this.tokens.expiresAt > now + 5 * 60 * 1000) {
      return this.tokens.accessToken;
    }

    // Start refresh
    this.refreshPromise = this.performRefresh();

    try {
      const tokens = await this.refreshPromise;
      this.setTokens(tokens);
      return tokens.accessToken;
    } finally {
      this.refreshPromise = null;
    }
  }

  private async performRefresh(): Promise<TokenPair> {
    if (!this.tokens?.refreshToken) {
      throw new Error('No refresh token available');
    }

    const response = await fetch('/auth/refresh', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken: this.tokens.refreshToken }),
    });

    if (!response.ok) {
      this.clearTokens();
      throw new Error('Token refresh failed');
    }

    return response.json();
  }
}

class ApiClient {
  constructor(private tokenManager: TokenManager) {}

  async request<T>(config: RequestConfig): Promise<T> {
    // Try request with current token
    let response = await this.executeRequest(config);

    // If 401, refresh and retry once
    if (response.status === 401) {
      try {
        const newToken = await this.tokenManager.refreshIfNeeded();
        config.headers = {
          ...config.headers,
          Authorization: `Bearer ${newToken}`,
        };
        response = await this.executeRequest(config);
      } catch (error) {
        // Refresh failed, redirect to login
        this.tokenManager.clearTokens();
        window.location.href = '/login';
        throw error;
      }
    }

    if (!response.ok) {
      throw new ApiError(response.status, await response.text());
    }

    return response.json();
  }

  private async executeRequest(config: RequestConfig): Promise<Response> {
    const token = this.tokenManager.getAccessToken();
    return fetch(config.url, {
      ...config,
      headers: {
        ...config.headers,
        ...(token && { Authorization: `Bearer ${token}` }),
      },
    });
  }
}
```

### Key Points

- **Single refresh promise** prevents multiple simultaneous refreshes
- **5-minute buffer** refreshes token before it expires
- **Retry once** after refresh—if still 401, credentials are invalid
- **Clear tokens on failure** and redirect to login

## Pattern 2: Proactive Refresh

Refresh token before it expires rather than waiting for 401.

```typescript
class ProactiveTokenManager extends TokenManager {
  private refreshTimer: NodeJS.Timeout | null = null;

  setTokens(tokens: TokenPair): void {
    super.setTokens(tokens);
    this.scheduleRefresh(tokens.expiresAt);
  }

  clearTokens(): void {
    super.clearTokens();
    if (this.refreshTimer) {
      clearTimeout(this.refreshTimer);
      this.refreshTimer = null;
    }
  }

  private scheduleRefresh(expiresAt: number): void {
    if (this.refreshTimer) {
      clearTimeout(this.refreshTimer);
    }

    // Refresh 5 minutes before expiry
    const refreshAt = expiresAt - 5 * 60 * 1000;
    const delay = Math.max(0, refreshAt - Date.now());

    this.refreshTimer = setTimeout(async () => {
      try {
        await this.refreshIfNeeded();
      } catch (error) {
        console.error('Background token refresh failed:', error);
        // Don't clear tokens—let next request trigger refresh
      }
    }, delay);
  }
}
```

### Benefits

- Users never experience 401 errors
- No request delays for token refresh
- Better UX for long-lived sessions

### Tradeoffs

- More complex—timer management, background refresh
- Still need 401 handling as fallback
- Refreshes even if user is idle

## Pattern 3: Request Queue

When multiple requests fail with 401 simultaneously, queue them and retry all after refresh.

```typescript
type QueuedRequest = {
  config: RequestConfig;
  resolve: (value: any) => void;
  reject: (error: any) => void;
};

class QueuedTokenManager extends TokenManager {
  private requestQueue: QueuedRequest[] = [];
  private isRefreshing = false;

  async handleUnauthorized<T>(
    config: RequestConfig,
    executeRequest: (config: RequestConfig) => Promise<T>
  ): Promise<T> {
    // If refresh in progress, queue this request
    if (this.isRefreshing) {
      return new Promise<T>((resolve, reject) => {
        this.requestQueue.push({ config, resolve, reject });
      });
    }

    // Start refresh
    this.isRefreshing = true;

    try {
      const newToken = await this.refreshIfNeeded();

      // Update config with new token
      config.headers = {
        ...config.headers,
        Authorization: `Bearer ${newToken}`,
      };

      // Retry original request
      const result = await executeRequest(config);

      // Process queued requests
      this.processQueue(newToken, executeRequest);

      return result;
    } catch (error) {
      // Refresh failed, reject all queued requests
      this.rejectQueue(error);
      throw error;
    } finally {
      this.isRefreshing = false;
    }
  }

  private processQueue(
    newToken: string,
    executeRequest: (config: RequestConfig) => Promise<any>
  ): void {
    this.requestQueue.forEach(({ config, resolve, reject }) => {
      config.headers = {
        ...config.headers,
        Authorization: `Bearer ${newToken}`,
      };
      executeRequest(config).then(resolve).catch(reject);
    });
    this.requestQueue = [];
  }

  private rejectQueue(error: any): void {
    this.requestQueue.forEach(({ reject }) => reject(error));
    this.requestQueue = [];
  }
}
```

### Use Case

When user navigates to a page that fires 10 API requests simultaneously, and token expires. Without queueing:

1. All 10 requests fail with 401
2. All 10 trigger refresh (race condition)
3. Multiple refresh requests sent
4. Potential token invalidation

With queueing:

1. First 401 triggers refresh
2. Other 9 requests queued
3. After refresh, all 10 retried with new token

## Pattern 4: Refresh Token Rotation

Some backends issue a new refresh token with each refresh (rotation). Handle this:

```typescript
interface RotatingTokenPair {
  accessToken: string;
  refreshToken: string;
  expiresAt: number;
}

class RotatingTokenManager {
  private tokens: RotatingTokenPair | null = null;

  async performRefresh(): Promise<RotatingTokenPair> {
    if (!this.tokens?.refreshToken) {
      throw new Error('No refresh token available');
    }

    const response = await fetch('/auth/refresh', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken: this.tokens.refreshToken }),
    });

    if (!response.ok) {
      this.clearTokens();
      throw new Error('Token refresh failed');
    }

    const newTokens: RotatingTokenPair = await response.json();

    // CRITICAL: Update tokens immediately to prevent using old refresh token
    this.setTokens(newTokens);

    // Persist to localStorage/sessionStorage
    this.persistTokens(newTokens);

    return newTokens;
  }

  private persistTokens(tokens: RotatingTokenPair): void {
    localStorage.setItem('auth_tokens', JSON.stringify(tokens));
  }

  loadTokens(): void {
    const stored = localStorage.getItem('auth_tokens');
    if (stored) {
      this.tokens = JSON.parse(stored);
    }
  }
}
```

### Critical Points

- **Update tokens immediately** after refresh—old refresh token is now invalid
- **Persist to storage** so tokens survive page refresh
- **Handle race conditions** carefully—if two tabs refresh simultaneously, one will fail

## Handling Multiple Tabs

When user has multiple tabs open, token refresh in one tab should update others.

```typescript
class MultiTabTokenManager extends TokenManager {
  constructor() {
    super();
    // Listen for storage changes from other tabs
    window.addEventListener('storage', this.handleStorageChange.bind(this));
  }

  private handleStorageChange(event: StorageEvent): void {
    if (event.key === 'auth_tokens' && event.newValue) {
      const tokens: TokenPair = JSON.parse(event.newValue);
      this.setTokens(tokens);
    }

    if (event.key === 'auth_tokens' && !event.newValue) {
      // Tokens cleared in another tab (logout)
      this.clearTokens();
      window.location.href = '/login';
    }
  }

  setTokens(tokens: TokenPair): void {
    super.setTokens(tokens);
    localStorage.setItem('auth_tokens', JSON.stringify(tokens));
  }

  clearTokens(): void {
    super.clearTokens();
    localStorage.removeItem('auth_tokens');
  }
}
```

## Security Considerations

### Storing Tokens

**localStorage:**

- Persists across sessions
- Vulnerable to XSS
- Accessible from any script

**sessionStorage:**

- Cleared on tab close
- Still vulnerable to XSS
- Better for sensitive apps

**Memory only:**

- Most secure
- Lost on page refresh
- Requires re-login frequently

**httpOnly cookies:**

- Not accessible to JavaScript
- Immune to XSS
- Requires backend support
- Best security, but limits client control

### Recommendations

- Use **httpOnly cookies** for refresh tokens if possible
- Store access tokens in **memory** (class property)
- If using localStorage, implement **CSP headers** to mitigate XSS
- Never log tokens to console or error tracking

## Testing Token Refresh

```typescript
describe('Token Refresh', () => {
  let tokenManager: TokenManager;
  let apiClient: ApiClient;

  beforeEach(() => {
    tokenManager = new TokenManager();
    apiClient = new ApiClient(tokenManager);
  });

  it('refreshes token on 401 and retries request', async () => {
    tokenManager.setTokens({
      accessToken: 'expired-token',
      refreshToken: 'valid-refresh',
      expiresAt: Date.now() - 1000, // Expired
    });

    // Mock fetch to return 401 first, then succeed
    global.fetch = jest
      .fn()
      .mockResolvedValueOnce({
        ok: false,
        status: 401,
      })
      .mockResolvedValueOnce({
        ok: true,
        json: async () => ({ data: 'success' }),
      });

    const result = await apiClient.request({ url: '/api/data' });
    expect(result).toEqual({ data: 'success' });
    expect(fetch).toHaveBeenCalledTimes(3); // Original + refresh + retry
  });

  it('prevents multiple simultaneous refreshes', async () => {
    tokenManager.setTokens({
      accessToken: 'expired-token',
      refreshToken: 'valid-refresh',
      expiresAt: Date.now() - 1000,
    });

    const refreshSpy = jest.spyOn(tokenManager as any, 'performRefresh');

    // Fire 5 requests simultaneously
    await Promise.all([
      apiClient.request({ url: '/api/1' }),
      apiClient.request({ url: '/api/2' }),
      apiClient.request({ url: '/api/3' }),
      apiClient.request({ url: '/api/4' }),
      apiClient.request({ url: '/api/5' }),
    ]);

    // Refresh should only be called once
    expect(refreshSpy).toHaveBeenCalledTimes(1);
  });

  it('clears tokens and redirects on refresh failure', async () => {
    tokenManager.setTokens({
      accessToken: 'expired-token',
      refreshToken: 'invalid-refresh',
      expiresAt: Date.now() - 1000,
    });

    global.fetch = jest.fn().mockResolvedValue({
      ok: false,
      status: 401,
    });

    const clearSpy = jest.spyOn(tokenManager, 'clearTokens');

    await expect(apiClient.request({ url: '/api/data' })).rejects.toThrow();
    expect(clearSpy).toHaveBeenCalled();
  });
});
```

## Decision Matrix

| Pattern | Use When | Complexity | UX Impact |
|---------|----------|------------|----------|
| Interceptor-Based | Simple apps, short sessions | Low | Occasional 401 delays |
| Proactive Refresh | Long sessions, real-time apps | Medium | Seamless |
| Request Queue | High request volume | Medium | Seamless |
| Rotating Tokens | High security requirements | High | Seamless |
| Multi-Tab Sync | Users use multiple tabs | High | Seamless |

## Key Takeaways

- **Prevent race conditions** with a single refresh promise
- **Retry only once** after refresh—subsequent 401 means invalid credentials
- **Queue concurrent requests** during refresh
- **Proactive refresh** (before expiry) provides best UX
- **Persist tokens** to survive page refresh
- **Sync across tabs** via localStorage events
- **Use httpOnly cookies** for refresh tokens when possible
- **Clear tokens and redirect** when refresh fails
