# Google Photos Integration Specification

## Overview
Google Photos integration providing read-only access to photo albums and media library with scope override support.

## Integration Manifest Addition

This integration extends the GSuite manifest as an additional service:

```typescript
// Added to src/lib/integrations/services/gsuite/manifest.ts
const photosService: ServiceDefinition = {
  id: "photos",
  name: "Google Photos",
  description: "Access your photo library and albums",
  scopes: [
    "https://www.googleapis.com/auth/photoslibrary.readonly"
  ],
  adapter: "./adapters/photos.ts",
  configSchema: {
    type: "object",
    properties: {
      maxAlbums: {
        type: "number",
        default: 50,
        description: "Maximum albums to fetch"
      },
      maxPhotosPerAlbum: {
        type: "number", 
        default: 100,
        description: "Maximum photos per album to fetch"
      },
      includeSharedAlbums: {
        type: "boolean",
        default: true,
        description: "Include shared albums in results"
      }
    }
  }
};

// Add to services array in gsuiteManifest
services: [gmailService, driveService, calendarService, contactsService, photosService]
```

## Service Details

### Google Photos Service

**Service ID**: `photos`

**Description**: Read-only access to Google Photos library including albums and media items.

**Available Scopes**:
- `photoslibrary.readonly` - View and search photos and albums (no upload/edit)

**Note**: Google Photos API currently only offers readonly scope for most use cases. Write operations (upload, create albums) require additional permissions that are not publicly available for all apps.

**Config Schema**:
```json
{
  "type": "object",
  "properties": {
    "maxAlbums": {
      "type": "number",
      "default": 50,
      "description": "Maximum number of albums to fetch (API limit: 100 per request)"
    },
    "maxPhotosPerAlbum": {
      "type": "number",
      "default": 100,
      "description": "Maximum photos to fetch per album"
    },
    "includeSharedAlbums": {
      "type": "boolean",
      "default": true,
      "description": "Include shared albums in results"
    },
    "mediaTypes": {
      "type": "array",
      "items": { 
        "type": "string",
        "enum": ["IMAGE", "VIDEO", "UNKNOWN_MEDIA_TYPE"]
      },
      "default": ["IMAGE", "VIDEO"],
      "description": "Media types to include"
    },
    "thumbnailSize": {
      "type": "string",
      "default": "200x200",
      "description": "Thumbnail size for previews (e.g., '100x100', '400x400')"
    }
  }
}
```

**Adapter API**:
```typescript
interface PhotosAdapter extends ServiceAdapter {
  // Album operations
  listAlbums(options?: {
    pageSize?: number;
    includeShared?: boolean;
  }): Promise<PhotoAlbum[]>;
  
  getAlbum(albumId: string): Promise<PhotoAlbum>;
  
  getAlbumItems(albumId: string, options?: {
    pageSize?: number;
    mediaTypes?: string[];
  }): Promise<MediaItem[]>;
  
  // Library operations
  searchLibrary(options?: {
    pageSize?: number;
    mediaTypes?: string[];
    filters?: PhotoFilters;
  }): Promise<MediaItem[]>;
  
  getMediaItem(mediaItemId: string): Promise<MediaItem>;
  
  // Polling support (limited - Photos doesn't have strong polling)
  poll(): Promise<IntegrationEvent[]>; // Returns "new_photo" events if available
}

interface PhotoAlbum {
  id: string;
  title: string;
  mediaItemCount: number;
  coverImageUrl?: string;
  isShared: boolean;
  createdAt: number;
}

interface MediaItem {
  id: string;
  filename: string;
  mimeType: string;
  description?: string;
  imageUrl?: string; // Base URL for accessing the image
  videoUrl?: string; // For video items
  thumbnailUrl?: string;
  width?: number;
  height?: number;
  durationMs?: number; // For videos
  mediaMetadata: {
    creationTime: number;
    location?: {
      latitude: number;
      longitude: number;
    };
  };
  albumId?: string; // If part of an album
}

interface PhotoFilters {
  startDate?: number;
  endDate?: number;
  includeArchived?: boolean;
  mediaTypes?: string[];
}
```

## API Endpoints Used

### Google Photos Library API v1

**Base URL**: `https://photoslibrary.googleapis.com/v1`

**Key Endpoints**:
- `GET /albums` - List user's albums
- `GET /albums/{albumId}` - Get specific album details
- `GET /albums/{albumId}/mediaItems` - List items in an album
- `GET /mediaItems/{mediaItemId}` - Get media item metadata
- `POST /mediaItems:search` - Search library with filters

**Media Access**:
- Media items return a `baseUrl` that can be used to access the actual content
- URL suffixes control size: `=d` (download), `=w{width}-h{height}` (resize)
- Example: `https://lh3.googleusercontent.com/...=w400-h400-d`

## Scope Override Behavior

**Single Scope Model**:
- Only one scope available: `photoslibrary.readonly`
- If user disables this scope, the entire Photos service becomes unavailable
- Visual representation in Settings:
  - ✅ Granted + Enabled: Toggle ON, green indicator
  - ❌ Granted + Disabled: Toggle OFF, grayed out
  - 🔒 Not Granted: Lock icon, disabled toggle with tooltip

**Important Note**: 
Google Photos API does not currently offer granular scopes (e.g., "read albums only" vs "read photos only"). All read access requires the single `photoslibrary.readonly` scope.

## Integration with BOS Features

### Gallery/Photos App Integration (Future)
- Optional: Display Google Photos in a native BOS Gallery app
- Show albums as folders
- Display thumbnails using provided URLs
- Full-size view on demand

### File System Integration (Not Recommended)
- NOT recommended to mount Photos as file system
- Photos are not traditional files and don't have folder structure
- Better suited for dedicated gallery/viewer app

### Search Integration
- Photos metadata (dates, locations) could be indexed by BOS search
- Requires periodic sync of library metadata
- Privacy consideration: user must enable this feature

## Error Handling

### Common Errors
```typescript
const photosErrorMessages = {
  TOKEN_EXPIRED: "Your Google Photos connection has expired. Please reauthorize.",
  TOKEN_REVOKED: "Access to Google Photos has been revoked. Click 'Reauthorize' to reconnect.",
  QUOTA_EXCEEDED: "Google Photos API quota exceeded. Will retry in 15 minutes.",
  NOT_FOUND: "Album or photo not found. It may have been deleted or you don't have access.",
  UNSUPPORTED_MEDIA_TYPE: "This media type is not supported for preview.",
  RATE_LIMITED: "Too many requests. Please wait before trying again."
};
```

### Rate Limiting
- Google Photos API has strict rate limits (typically 1,000 requests per day)
- Implement request queuing and throttling
- Cache results to minimize API calls
- Show user-friendly message when quota is exhausted

## Polling & Webhook Considerations

**Limited Support**:
- Google Photos API does **not** provide webhook/push notification support
- No real-time notifications for new photos or album updates
- Best approach: Periodic sync of recent items only
- Recommended poll interval: 3600 seconds (1 hour) minimum

**Adapter Webhook Methods** (stub implementation):
```typescript
interface PhotosAdapter extends ServiceAdapter {
  // ... existing methods
  
  // Webhook support: Not available, but interface must be implemented
  supportsWebhooks(): boolean { return false; }
  setupWebhook(topicName: string, secret: string): Promise<never> { 
    throw new Error("Webhooks not supported by Google Photos API"); 
  }
  teardownWebhook(channelId: string): Promise<void> { /* no-op */ }
  processWebhook(payload: any): Promise<IntegrationEvent | null> { 
    return null; // Webhooks not supported
  }
}
```

**User Experience**:
- Webhook configuration option should be **hidden or disabled** for Photos service
- Clear message: "Google Photos does not support real-time notifications. Using periodic sync instead."

```typescript
// Minimal polling implementation
async poll(): Promise<IntegrationEvent[]> {
  const events: IntegrationEvent[] = [];
  
  // Only check for very recent photos (last hour)
  const oneHourAgo = Date.now() - (60 * 60 * 1000);
  
  try {
    const recentPhotos = await this.searchLibrary({
      startDate: oneHourAgo,
      pageSize: 10
    });
    
    recentPhotos.forEach(photo => {
      events.push({
        type: "new_photo",
        service: "photos",
        timestamp: Date.now(),
        data: {
          mediaItemId: photo.id,
          filename: photo.filename,
          thumbnailUrl: photo.thumbnailUrl
        }
      });
    });
  } catch (error) {
    // Log error but don't fail the entire poll
    console.error("Photos polling failed:", error);
  }
  
  return events;
}
```

## UI Mockup for Settings

### Photos Service Configuration Panel
```
┌─────────────────────────────────────────┐
│  Google Photos                          │
├─────────────────────────────────────────┤
│  ☑ Enabled                              │
│                                         │
│  Configuration                          │
│  ┌───────────────────────────────────┐  │
│  │ Max albums to fetch: [50      ]   │  │
│  │ Max photos per album: [100    ]   │  │
│  │ ✓ Include shared albums           │  │
│  │ Media types: [Images, Videos ▼]   │  │
│  │ Thumbnail size: [400x400      ]   │  │
│  └───────────────────────────────────┘  │
│                                         │
│  Scope Status                           │
│  ┌───────────────────────────────────┐  │
│  │ photoslibrary.readonly    [✓]     │  │
│  │   • View and search photos         │  │
│  │   • Access albums                  │  │
│  └───────────────────────────────────┘  │
│                                         │
│  [Test Connection] [Sync Now]           │
└─────────────────────────────────────────┘
```

## Security & Privacy

### Data Handling
1. **Read-Only Access**: Cannot modify or delete user's photos
2. **No Persistent Storage**: Photo metadata cached in memory only
3. **User Control**: Can disable service at any time via Settings
4. **Minimal Data**: Only fetch what's configured (album/photo limits)

### Token Security
- OAuth token stored encrypted via SecretsStore
- Refresh token handled automatically
- User can revoke access anytime from Google Account settings

## Testing Requirements

### Unit Tests
- Scope override logic for single-scope model
- Album and media item parsing
- URL generation for thumbnails/media access
- Filter application

### Integration Tests
- Full OAuth flow with Photos scope
- Album listing and pagination
- Media item retrieval
- Search with various filters
- Thumbnail URL validity

### UI Tests
- Service enable/disable behavior
- Configuration persistence
- Scope status display (granted vs not granted)
- Error state handling

## Limitations & Considerations

### API Limitations
1. **Read-Only**: Cannot upload photos or create albums via public API
2. **Rate Limits**: Strict daily quotas (1,000 requests/day typical)
3. **No Webhooks**: Limited real-time notification support
4. **Media Access**: URLs expire after ~1 hour, must refresh

### User Experience Considerations
1. **Large Libraries**: Users with 10,000+ photos need pagination UI
2. **Network Usage**: High-resolution photos consume bandwidth
3. **Privacy**: Some users may not want cloud photos accessible locally
4. **Performance**: Loading many thumbnails can be slow

## Future Enhancements

- [ ] Support for Google Photos shared albums and collaboration
- [ ] Advanced search with AI tags (people, places, things)
- [ ] Offline caching of frequently accessed albums
- [ ] Integration with BOS image viewer/editor
- [ ] Photo timeline view in native BOS app
- [ ] Album creation/upload (if API access expanded)

## References

- Google Photos Library API Documentation: https://developers.google.com/photos/library
- OAuth Scopes Reference: https://developers.google.com/identity/protocols/oauth2/scopes#google-photos
- API Quotas and Limits: https://developers.google.com/photos/library/guides/quota-costs-and-limits
