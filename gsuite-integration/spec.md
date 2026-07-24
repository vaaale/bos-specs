# GSuite Integration Specification

## Overview
GSuite (Google Workspace) integration providing access to Gmail, Drive, Calendar, and Contacts services with per-service configuration and scope override support.

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

## Integration Manifest

```typescript
// src/lib/integrations/services/gsuite/manifest.ts
export const gsuiteManifest: IntegrationManifest = {
  id: "gsuite",
  name: "GSuite",
  version: "1.0.0",
  description: "Google Workspace integration for email, files, calendar, and contacts",
  icon: "Google",
  services: [
    gmailService,
    driveService,
    calendarService,
    contactsService
  ],
  oauthConfig: {
    authorizationUrl: "https://accounts.google.com/o/oauth2/v2/auth",
    tokenUrl: "https://oauth2.googleapis.com/token",
    supportedScopes: [
      "https://www.googleapis.com/auth/gmail.readonly",
      "https://www.googleapis.com/auth/gmail.modify",
      "https://www.googleapis.com/auth/gmail.send",
      "https://www.googleapis.com/auth/drive.readonly",
      "https://www.googleapis.com/auth/drive.file",
      "https://www.googleapis.com/auth/calendar.readonly",
      "https://www.googleapis.com/auth/calendar.events",
      "https://www.googleapis.com/auth/contacts.readonly"
    ]
  }
};
```

## Services

### 1. Gmail Service

**Service ID**: `gmail`

**Description**: Email integration for reading, searching, and sending emails.

**Available Scopes**:
- `gmail.readonly` - Read email messages and labels
- `gmail.modify` - Read, modify, and manage email (includes readonly)
- `gmail.send` - Send emails

**Config Schema**:
```json
{
  "type": "object",
  "properties": {
    "pollInterval": {
      "type": "number",
      "default": 300,
      "description": "Polling interval in seconds (for checking new emails)"
    },
    "maxResults": {
      "type": "number",
      "default": 25,
      "description": "Maximum number of recent emails to fetch"
    },
    "notifyOnNewEmail": {
      "type": "boolean",
      "default": true,
      "description": "Show BOS notifications for new emails"
    },
    "labelFilter": {
      "type": "string",
      "default": "",
      "description": "Only monitor specific labels (e.g., 'INBOX', 'Important')"
    }
  }
}
```

**Adapter API**:
```typescript
interface GmailAdapter extends ServiceAdapter {
  // Email operations
  listEmails(options?: { 
    maxResults?: number; 
    label?: string; 
    query?: string;
  }): Promise<GmailMessage[]>;
  
  getEmail(messageId: string): Promise<GmailMessage>;
  
  sendEmail(recipient: string, subject: string, body: string): Promise<void>;
  
  markAsRead(messageIds: string[]): Promise<void>;
  
  // Polling support
  poll(): Promise<IntegrationEvent[]>; // Returns "new_email" events
  
  // Webhook support (Google Cloud Pub/Sub)
  supportsWebhooks(): boolean;
  setupWebhook(topicName: string, secret: string): Promise<string>; // Returns channel ID
  teardownWebhook(channelId: string): Promise<void>;
  processWebhook(payload: any): Promise<IntegrationEvent | null>; // Handle Pub/Sub push messages
}
```

**Webhook Implementation for Gmail**:
- Google Gmail uses **Google Cloud Pub/Sub** for push notifications
- User must create a Pub/Sub topic and grant permissions
- BOS registers a push subscription pointing to the webhook endpoint
- Google sends base64-encoded JSON payloads to the webhook URL
- Webhook Manager decodes payload, verifies signature (using topic verification), and processes event

**Webhook Configuration for Gmail**:
```json
{
  "type": "object",
  "properties": {
    "useWebhooks": {
      "type": "boolean",
      "default": false,
      "description": "Use push notifications instead of polling"
    },
    "pubsubTopic": {
      "type": "string",
      "description": "Google Cloud Pub/Sub topic name (e.g., 'projects/my-project/topics/gmail-webhooks')"
    },
    "webhookSecret": {
      "type": "string",
      "description": "Shared secret for HMAC verification (generated by BOS)"
    }
  }
}
```

**Webhook Flow for Gmail**:
1. User enables "Use webhooks" in Gmail settings
2. System prompts user to create a Pub/Sub topic (or use existing)
3. BOS registers push subscription with Google:
   - Push endpoint: `https://[bos-domain]/api/integrations/webhooks/gsuite/gmail`
   - Topic: User's Pub/Sub topic
4. Google sends messages when new emails arrive
5. Webhook Manager receives, decodes, and processes payload
6. Event dispatched to BOS event system

interface GmailMessage {
  id: string;
  threadId: string;
  from: string;
  to: string[];
  subject: string;
  snippet: string;
  body: string;
  receivedAt: number;
  labels: string[];
  isRead: boolean;
}
```

**Scope Override Behavior**:
- If user has `gmail.modify` but disables it, fallback to `gmail.readonly` if granted
- Cannot enable `gmail.send` unless explicitly granted by OAuth
- Disabling all Gmail scopes effectively disables the service

### 2. Drive Service

**Service ID**: `drive`

**Description**: Google Drive file storage integration.

**Available Scopes**:
- `drive.readonly` - View and search files (cannot modify)
- `drive.file` - Access files created by this app only
- `drive.install` - Extended Drive access (rarely needed)

**Config Schema**:
```json
{
  "type": "object",
  "properties": {
    "rootFolderId": {
      "type": "string",
      "default": "",
      "description": "Drive folder ID to use as root (empty = Drive root)"
    },
    "fileTypes": {
      "type": "array",
      "items": { "type": "string" },
      "default": [],
      "description": "Filter by MIME types (empty = all types)"
    },
    "showInFileApp": {
      "type": "boolean",
      "default": false,
      "description": "Mount Drive as accessible folder in Files app"
    }
  }
}
```

**Adapter API**:
```typescript
interface DriveAdapter extends ServiceAdapter {
  // File operations
  listFiles(options?: { 
    folderId?: string; 
    mimeType?: string;
    pageSize?: number;
  }): Promise<DriveFile[]>;
  
  getFileContent(fileId: string): Promise<Buffer>;
  
  searchFiles(query: string): Promise<DriveFile[]>;
  
  // Note: Write operations require higher scopes and are not included in basic spec
}

interface DriveFile {
  id: string;
  name: string;
  mimeType: string;
  size?: number;
  modifiedAt: number;
  parents: string[];
  webViewLink?: string;
}
```

**Scope Override Behavior**:
- `drive.file` scope is more restrictive than `drive.readonly`
- If user has `drive.readonly` but disables it, service becomes read-only for app-created files only (if `drive.file` granted)

### 3. Calendar Service

**Service ID**: `calendar`

**Description**: Google Calendar event management and reminders.

**Available Scopes**:
- `calendar.readonly` - View calendar events
- `calendar.events` - Create, modify, and delete events

**Config Schema**:
```json
{
  "type": "object",
  "properties": {
    "calendarId": {
      "type": "string",
      "default": "primary",
      "description": "Calendar ID to use ('primary' or specific calendar)"
    },
    "pollInterval": {
      "type": "number",
      "default": 300,
      "description": "Polling interval in seconds"
    },
    "reminderMinutesBefore": {
      "type": "number",
      "default": 15,
      "description": "Notify X minutes before event"
    },
    "maxEvents": {
      "type": "number",
      "default": 20,
      "description": "Maximum upcoming events to track"
    }
  }
}
```

**Adapter API**:
```typescript
interface CalendarAdapter extends ServiceAdapter {
  // Event operations
  listEvents(options?: { 
    calendarId?: string;
    timeMin?: number;
    timeMax?: number;
    maxResults?: number;
  }): Promise<CalendarEvent[]>;
  
  createEvent(event: CalendarEventCreate): Promise<void>;
  
  updateEvent(eventId: string, updates: Partial<CalendarEvent>): Promise<void>;
  
  deleteEvent(eventId: string): Promise<void>;
  
  // Polling support
  poll(): Promise<IntegrationEvent[]>; // Returns "calendar_reminder" events
}

interface CalendarEvent {
  id: string;
  summary: string;
  description?: string;
  location?: string;
  startAt: number;
  endAt: number;
  attendees?: string[];
  status: 'confirmed' | 'tentative' | 'cancelled';
}

interface CalendarEventCreate {
  summary: string;
  description?: string;
  startAt: number;
  endAt: number;
  attendees?: string[];
}
```

**Scope Override Behavior**:
- If `calendar.events` is disabled but `calendar.readonly` is granted, service becomes read-only
- Cannot create/update events without `calendar.events` scope enabled

### 4. Contacts Service

**Service ID**: `contacts`

**Description**: Google Contacts address book integration.

**Available Scopes**:
- `contacts.readonly` - View and search contacts

**Config Schema**:
```json
{
  "type": "object",
  "properties": {
    "maxContacts": {
      "type": "number",
      "default": 500,
      "description": "Maximum contacts to fetch"
    },
    "syncInterval": {
      "type": "number",
      "default": 3600,
      "description": "Sync interval in seconds (1 hour default)"
    }
  }
}
```

**Adapter API**:
```typescript
interface ContactsAdapter extends ServiceAdapter {
  // Contact operations
  listContacts(options?: { 
    maxResults?: number;
    query?: string;
  }): Promise<Contact[]>;
  
  getContact(contactId: string): Promise<Contact>;
  
  searchContacts(query: string): Promise<Contact[]>;
}

interface Contact {
  id: string;
  displayName: string;
  names?: Name[];
  emails?: Email[];
  phones?: Phone[];
  organization?: Organization[];
}

interface Name {
  givenName?: string;
  familyName?: string;
  displayName?: string;
}

interface Email {
  value: string;
  type?: 'home' | 'work' | 'other';
}

interface Phone {
  value: string;
  type?: 'mobile' | 'work' | 'home';
}

interface Organization {
  name?: string;
  title?: string;
}
```

**Scope Override Behavior**:
- Only one scope available (`contacts.readonly`)
- Disabling this scope completely disables the Contacts service

## OAuth Configuration

### Initial Setup Flow

1. User navigates to Settings → Integrations → GSuite → Connect
2. System prompts for Google Cloud OAuth credentials (if not already configured):
   - Client ID
   - Client Secret
3. User selects which services to enable (determines scopes requested)
4. Redirect to Google OAuth consent screen
5. User grants permissions
6. Callback receives authorization code
7. Exchange for tokens, store in SecretsStore
8. Integration state updated to "connected"

### Scope Request Logic

```typescript
function buildScopeList(enabledServices: string[], userOverrides: Record<string, boolean>): string[] {
  const allScopes: string[] = [];
  
  enabledServices.forEach(serviceId => {
    const service = getServiceDefinition(serviceId);
    service.scopes.forEach(scope => {
      // Only include scope if:
      // 1. It's in the manifest's supported scopes
      // 2. User hasn't explicitly disabled it
      if (!userOverrides[scope] === false) {
        allScopes.push(scope);
      }
    });
  });
  
  // Remove duplicates
  return [...new Set(allScopes)];
}
```

## Integration with BOS Features

### Notification System
- New emails → BOS notification with sender, subject preview
- Calendar reminders → BOS notification X minutes before event
- Configurable per-service in Settings

### Scheduler Integration
- Gmail polling: Check for new emails every N seconds (configurable)
- Calendar polling: Check for upcoming events/reminders
- Contacts sync: Periodic sync to keep local cache updated

### Files App Integration (Future)
- Optional: Mount Drive folders as accessible locations in Files app
- Requires `drive.readonly` or higher scope
- Implemented as virtual file system mount point

## Error Handling

### Common Errors
1. **Token Expired**: Auto-refresh using refresh token
2. **Token Revoked**: Show "Reauthorize" button in Settings
3. **API Quota Exceeded**: Implement exponential backoff, show warning
4. **Network Error**: Retry with backoff, mark service as temporarily unavailable

### User-Facing Messages
```typescript
const errorMessages = {
  TOKEN_EXPIRED: "Your GSuite connection has expired. Please reauthorize.",
  TOKEN_REVOKED: "Access to GSuite has been revoked. Click 'Reauthorize' to reconnect.",
  QUOTA_EXCEEDED: "Google API quota exceeded. Will retry in 15 minutes.",
  NETWORK_ERROR: "Unable to connect to Google services. Check your internet connection.",
  SCOPE_DENIED: "This feature requires additional permissions. Please enable the required scopes in Settings."
};
```

## Testing Requirements

### Unit Tests
- OAuth flow simulation
- Scope override logic
- Service adapter interfaces
- Token refresh mechanism

### Integration Tests
- Full OAuth flow with real Google credentials
- Email listing, sending (if scope granted)
- Calendar event operations
- File listing from Drive
- Contact retrieval

### UI Tests
- Scope toggle behavior (can't enable ungranted scopes)
- Service enable/disable persistence
- Error state display
- Connection status indicators

## Security Considerations

1. **Minimal Scopes**: Request only scopes needed for enabled services
2. **User Control**: Users can disable any granted scope at any time
3. **Token Encryption**: All tokens stored encrypted via SecretsStore
4. **No Data Persistence**: Email/contacts data cached only in memory, not persisted
5. **Audit Trail**: Log all authentication and authorization changes

## Future Enhancements

- [ ] Google Docs/Sheets/Slides integration
- [ ] Two-way sync for contacts (requires write scope)
- [ ] Drive file upload/download with larger file support
- [ ] Gmail label management
- [ ] Calendar event creation/editing UI in BOS
- [ ] Offline caching and sync
