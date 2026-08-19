# Integration Framework Specification

## Overview
A modular, extensible framework for integrating external services (GSuite, Telegram, etc.) into BrowserOS with secure authentication, per-service configuration, and scope override capabilities.

## UI Mockup

**Interactive drill-down mockup available**: [`mockup-drilldown.html`](./mockup-drilldown.html)

This mockup demonstrates the **master-detail navigation pattern**:
1. **Integration List** → Click integration to drill down
2. **Integration Detail** (GSuite overview) → Click service to configure  
3. **Service Configuration** (Gmail example) → Configure scopes, polling, webhooks

### Navigation Flow
```
Integrations List 
  ↓ (click GSuite)
GSuite Detail (shows all services)
  ↓ (click Gmail)
Gmail Configuration (scopes + polling + webhooks)
```

### Key Features Demonstrated
- **Breadcrumb navigation** with clickable back steps
- **Progressive disclosure**: Users see one level at a time
- **Scalable pattern**: Works for any integration/service combination
- **Scope configuration** per service with toggle switches
- **Polling configuration** with interval settings
- **Webhook configuration** with URL/secret management
- BOS design system colors and spacing (dark theme, opacity palette, violet accents)

### Design Notes for Implementation
- Use `rgba(255, 255, 255, X)` opacity scale for all colors (no named grays)
- Primary accent: violet (`#a78bfa` or `rgba(167, 139, 250, X)`)
- Warning/attention: amber (`#fbbf24`)
- Success: emerald (`#34d399`)
- Font size: `11px-13px` for dense UI
- Spacing: `gap-1` to `gap-4` (4px to 16px)
- Border radius: `6px` for cards, `8px` for panels
- All inputs/buttons use `rgba(255,255,255,0.1)` backgrounds with hover states
- **Navigation**: Use breadcrumb pattern for drill-down flow

See [BOS UI Style Guide](../../../docs/dev/guides/style-guide.md) for full conventions.

## Architecture

### Core Components

#### 1. SecretsStore
**Purpose**: Secure storage and retrieval of OAuth tokens and sensitive credentials.

**Design**:
- NOT using environment variables
- Encrypted storage at `data/integrations/secrets.json`
- AES-256-GCM encryption with key derived from:
  - Primary: User-provided master passphrase (stored in browser secure storage)
  - Fallback: Auto-generated key file at `data/.integrations-key` (chmod 600)
- API:
  ```typescript
  interface SecretsStore {
    set(integration: string, key: string, value: string): Promise<void>;
    get(integration: string, key: string): Promise<string | null>;
    delete(integration: string, key: string): Promise<void>;
    listKeys(integration: string): Promise<string[]>;
  }
  ```

#### 2. OAuth Manager
**Purpose**: Handle OAuth 2.0 loopback flow for all integrations.

**Flow**:
1. User clicks "Connect" in Settings → Integrations
2. System generates PKCE code challenge
3. Redirect to provider's authorization URL with:
   - `client_id`, `redirect_uri=http://localhost:3000/api/integrations/oauth/callback`
   - `scope`, `code_challenge`, `response_type=code`
4. Provider redirects to callback endpoint with authorization code
5. Exchange code for access token + refresh token
6. Store tokens in SecretsStore
7. Update integration state to "connected"

**API**:
```typescript
interface OAuthManager {
  startFlow(integration: string, scopes: string[]): Promise<string>; // returns auth URL
  handleCallback(code: string, verifier: string): Promise<OAuthTokens>;
  refreshToken(integration: string): Promise<OAuthTokens>;
  getValidToken(integration: string): Promise<OAuthTokens>; // auto-refresh if needed
}

interface OAuthTokens {
  access_token: string;
  refresh_token: string;
  expires_at: number; // timestamp
  granted_scopes: string[];
}
```

#### 3. Integration Registry
**Purpose**: Central registry of all available integrations.

**Structure**:
```typescript
interface IntegrationManifest {
  id: string; // e.g., "gsuite", "telegram"
  name: string;
  version: string;
  description: string;
  icon: string; // lucide icon name
  services: ServiceDefinition[];
  oauthConfig: OAuthConfig;
}

interface ServiceDefinition {
  id: string; // e.g., "gmail", "drive"
  name: string;
  description: string;
  scopes: string[]; // scopes required for this service
  adapter: string; // path to adapter implementation
  configSchema: JSONSchema; // per-service configuration options
}

interface OAuthConfig {
  authorizationUrl: string;
  tokenUrl: string;
  supportedScopes: string[]; // all scopes this integration can request
}
```

**Registry API**:
```typescript
interface IntegrationRegistry {
  register(manifest: IntegrationManifest): void;
  getIntegration(id: string): IntegrationManifest | null;
  listIntegrations(): IntegrationManifest[];
  getIntegrationState(integrationId: string): IntegrationState;
}
```

#### 4. Integration State Store
**Purpose**: Persistent storage of integration configuration and status.

**Location**: `data/integrations/<integration-id>/state.json`

**Structure**:
```typescript
interface IntegrationState {
  connected: boolean;
  lastConnected: number;
  services: {
    [serviceId: string]: {
      enabled: boolean; // user-enabled (may be false even if OAuth granted)
      config: Record<string, any>; // service-specific settings
      lastSync?: number;
      error?: string;
    };
  };
  scopeOverrides: {
    [scope: string]: boolean; // true=enabled, false=disabled by user
    // Note: only scopes in granted_scopes can be set to false
    // Scopes not granted cannot be set to true
  };
  oauthTokens?: OAuthTokens; // encrypted via SecretsStore, this is just metadata
}
```

#### 5. Service Adapters
**Purpose**: Abstraction layer for each service within an integration.

**Base Interface**:
```typescript
interface ServiceAdapter {
  getId(): string;
  getName(): string;
  
  // Lifecycle
  connect(tokens: OAuthTokens): Promise<void>;
  disconnect(): Promise<void>;
  isEnabled(): boolean;
  setEnabled(enabled: boolean): void;
  
  // Scope management
  getRequiredScopes(): string[];
  getEffectiveScopes(grantedScopes: string[], userOverrides: Record<string, boolean>): string[];
  
  // Service-specific API (implemented by each adapter)
  getConfigSchema(): JSONSchema;
  validateConfig(config: Record<string, any>): boolean;
  
  // Event support
  supportsPolling(): boolean;
  getRecommendedPollInterval(): number; // in seconds
  poll(): Promise<IntegrationEvent[]>;
}

interface IntegrationEvent {
  type: string; // e.g., "new_email", "calendar_reminder"
  service: string;
  timestamp: number;
  data: Record<string, any>;
}
```

#### 6. Event System
**Purpose**: Dispatch events from polling/listening integrations to the rest of BOS.

**Components**:
- Event dispatcher that broadcasts to subscribed listeners
- Integration with BOS notification system
- Scheduler integration for polling jobs
- **Webhook Manager** for receiving push notifications from external services

**API**:
```typescript
interface EventSystem {
  subscribe(eventType: string, handler: (event: IntegrationEvent) => void): () => void;
  dispatch(event: IntegrationEvent): void;
  
  // Polling management
  registerPollingJob(integrationId: string, serviceId: string, intervalSec: number): void;
  unregisterPollingJob(integrationId: string, serviceId: string): void;
  
  // Webhook management
  registerWebhook(integrationId: string, serviceId: string, webhookUrl: string, secret?: string): Promise<void>;
  unregisterWebhook(integrationId: string, serviceId: string): Promise<void>;
  verifyWebhookSignature(payload: any, signature: string, secret: string): boolean;
}
```

#### 7. Webhook Manager (New Component)
**Purpose**: Handle incoming webhooks from external services securely and reliably.

**Architecture**:
- **Public Endpoint**: `POST /api/integrations/webhooks/[integration-id]/[service-id]`
- **Security**: HMAC signature verification using shared secrets
- **Retry Logic**: Automatic retry with exponential backoff for failed processing
- **Idempotency**: Deduplication of duplicate webhook events

**Webhook Handler Interface**:
```typescript
interface WebhookHandler {
  getId(): string; // e.g., "gsuite_gmail", "telegram_messages"
  
  // Validation
  validateSignature(payload: any, signature: string, secret: string): boolean;
  
  // Processing
  processWebhook(payload: any): Promise<IntegrationEvent | null>; // Returns event or null if invalid
  
  // Configuration
  getRequiredSecret(): boolean; // Does this webhook require a shared secret?
  generateSecret(): string; // Generate random secret for user to configure
}

interface WebhookConfig {
  enabled: boolean;
  url: string; // BOS public URL (user-configured)
  secret: string; // HMAC secret (stored in SecretsStore)
  events: string[]; // Specific event types to subscribe to
  lastVerified?: number;
  verificationStatus?: 'pending' | 'verified' | 'failed';
}
```

**Webhook Flow**:
1. User enables webhook support in Integration Settings
2. System generates a unique secret (or user provides one)
3. User configures their external service with:
   - Webhook URL: `https://[bos-domain]/api/integrations/webhooks/[integration-id]/[service-id]`
   - Secret: The generated/shared secret
   - Event types to subscribe to
4. BOS registers webhook endpoint (internal routing)
5. External service sends POST request with signed payload
6. Webhook Manager:
   - Verifies HMAC signature
   - Validates payload structure
   - Calls service adapter's `processWebhook()` method
   - Dispatches resulting event via EventSystem
7. If processing fails, retry with exponential backoff (max 3 attempts)

**Security Requirements**:
- All webhook payloads must be signed (HMAC-SHA256)
- Secrets stored encrypted in SecretsStore
- Rate limiting per webhook endpoint (prevent abuse)
- IP whitelisting optional (for services that publish sender IPs)
- Automatic rotation of secrets on user request

**Configuration UI for Webhooks**:
```
┌─────────────────────────────────────────┐
│  Webhook Configuration                  │
├─────────────────────────────────────────┤
│  ☑ Enable webhooks                      │
│                                         │
│  Webhook URL                            │
│  ┌───────────────────────────────────┐  │
│  │ https://mydomain.com/api/...      │  │ [Copy] │
│  └───────────────────────────────────┘  │
│                                         │
│  Shared Secret                          │
│  ┌───────────────────────────────────┐  │
│  │ •••••••••••••••••••••••••••••     │  │ [Regenerate] [Show] │
│  └───────────────────────────────────┘  │
│                                         │
│  Events to Subscribe To                 │
│  ☑ new_email                            │
│  ☑ email_sent                           │
│  ☐ label_updated                        │
│                                         │
│  Status                                 │
│  ● Verified (last ping: 2 min ago)     │
│  [Test Webhook]                         │
└─────────────────────────────────────────┘
```

**Testing Webhooks**:
- "Test Webhook" button sends a sample payload to verify configuration
- Logs show recent webhook deliveries with status
- User can view raw payloads for debugging

## Settings UI Requirements

### Integrations Tab Structure

#### Main View (List of Integrations)
```
┌─────────────────────────────────────────┐
│  Integrations                           │
├─────────────────────────────────────────┤
│  ┌───────────────────────────────────┐  │
│  │ GSuite         ● Connected        │  │
│  │   Gmail, Drive, Calendar...       │  │
│  │   Last sync: 2 min ago            │  │
│  └───────────────────────────────────┘  │
│  ┌───────────────────────────────────┐  │
│  │ Telegram       ○ Not connected    │  │
│  │   Messages, Channels              │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

Status indicators:
- ● Connected (green)
- ⚠️ Partial (yellow - some services disabled/errors)
- ○ Not connected (gray)

#### Detail View (Per Integration)
```
┌─────────────────────────────────────────┐
│  ← GSuite                               │
├─────────────────────────────────────────┤
│  Authentication                         │
│  ┌───────────────────────────────────┐  │
│  │ Connected as user@example.com     │  │
│  │ [Reauthorize] [Disconnect]        │  │
│  └───────────────────────────────────┘  │
│                                         │
│  Services                               │
│  ┌───────────────────────────────────┐  │
│  │ ☑ Gmail              ● Active     │  │
│  │   • Read emails                    │  │
│  │   • Send emails                    │  │
│  │                                    │  │
│  │ ☑ Drive              ● Active     │  │
│  │   • View files                     │  │
│  │                                    │  │
│  │ ☐ Calendar           ⚠ Disabled   │  │
│  │   • View events (granted but disabled) │
│  │                                    │  │
│  │ 🔒 Photos            🔒 Not granted│  │
│  │   • View photos (scope not granted)│  │
│  └───────────────────────────────────┘  │
│                                         │
│  Scope Configuration                    │
│  ┌───────────────────────────────────┐  │
│  │ Scope                    Enabled  │  │
│  │ ├─ gmail.readonly        [✓]      │  │
│  │ ├─ gmail.send            [✓]      │  │
│  │ ├─ calendar.readonly     [ ]      │  │ ← User disabled
│  │ └─ photoslibrary.readonly [✗]    │  │ ← Lock icon, tooltip: "Not granted by OAuth"
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### Scope Override UI Logic

**Visual States**:
1. **Granted + Enabled** (default): Toggle ON, green indicator
2. **Granted + Disabled**: Toggle OFF, grayed out, tooltip "Disabled by you"
3. **Not Granted**: Toggle disabled (locked), lock icon, tooltip "Not granted by OAuth authorization"

**Business Logic**:
```typescript
function getEffectiveScopes(grantedScopes: string[], userOverrides: Record<string, boolean>): string[] {
  return grantedScopes.filter(scope => {
    // If scope not in granted list, it's not available
    if (!grantedScopes.includes(scope)) return false;
    
    // If user explicitly disabled it (false), exclude it
    if (userOverrides[scope] === false) return false;
    
    // Otherwise, include it (default true)
    return true;
  });
}

function canToggleScope(scope: string, grantedScopes: string[], currentOverride: boolean | undefined): boolean {
  // Can only toggle if scope is in granted list
  return grantedScopes.includes(scope);
}
```

## Extension Points

### Adding a New Integration

1. Create manifest at `src/lib/integrations/services/<integration-id>/manifest.ts`
2. Implement service adapters in `src/lib/integrations/services/<integration-id>/adapters/`
3. Register with IntegrationRegistry on BOS startup
4. Add OAuth configuration (authorization/token URLs)
5. Define service-specific config schemas

### Example: Adding Telegram Integration

```typescript
// src/lib/integrations/services/telegram/manifest.ts
export const telegramManifest: IntegrationManifest = {
  id: "telegram",
  name: "Telegram",
  version: "1.0.0",
  description: "Integrate with Telegram for messaging and notifications",
  icon: "MessageSquare",
  services: [
    {
      id: "messages",
      name: "Messages",
      description: "Send and receive messages",
      scopes: ["telegram.read", "telegram.write"],
      adapter: "./adapters/messages.ts",
      configSchema: {
        type: "object",
        properties: {
          pollInterval: { type: "number", default: 30 },
          notifyOnNewMessage: { type: "boolean", default: true }
        }
      }
    }
  ],
  oauthConfig: {
    authorizationUrl: "https://telegram.org/oauth/authorize",
    tokenUrl: "https://telegram.org/oauth/token",
    supportedScopes: ["telegram.read", "telegram.write"]
  }
};
```

## Security Considerations

1. **Token Storage**: All OAuth tokens encrypted via SecretsStore
2. **Scope Minimization**: Request only required scopes per service
3. **User Control**: Users can disable any granted scope
4. **No Persistent Access**: Support token revocation and reauthorization
5. **Audit Logging**: Log all authentication events and scope changes

## Implementation Phases

### Phase 1: Core Framework + Gmail
- [ ] SecretsStore implementation
- [ ] OAuth Manager with loopback flow
- [ ] Integration Registry and State Store
- [ ] Base ServiceAdapter interface
- [ ] Settings UI (Integrations tab)
- [ ] GSuite manifest
- [ ] Gmail adapter (FULLY functional with scope overrides)

### Phase 2: Polling Infrastructure
- [ ] Event System implementation
- [ ] Scheduler integration
- [ ] Polling job management
- [ ] Test with Gmail polling

### Phase 3: Additional Services
- [ ] Drive adapter
- [ ] Calendar adapter
- [ ] Contacts adapter
- [ ] Google Photos adapter

### Phase 4: Documentation
- [ ] Developer guide for creating integrations
- [ ] User guide for configuration
- [ ] API reference for service adapters

## References
- OAuth 2.0 RFC 6749
- PKCE RFC 7636
- Existing BOS patterns: Settings tabs, Config system, Scheduler app
