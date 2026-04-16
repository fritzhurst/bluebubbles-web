# BlueBubbles Web App Analysis and Setup Guide

**Date:** March 17, 2026  
**Prepared for:** Fritz Hurst  
**Analysis based on:** BlueBubbles-app source code review and built web app inspection

## Executive Summary

This document provides a comprehensive analysis of the BlueBubbles web app synchronization issues and step-by-step guides for setting up both testing and development environments. The core issue is that the web app lacks local data persistence, causing data loss on browser refresh.

## Repository Information

- **Your BlueBubbles Server Fork:** https://github.com/fritzhurst/bluebubbles-server
- **Your BlueBubbles App Fork:** https://github.com/fritzhurst/bluebubbles-app
- **Original BlueBubbles App:** https://github.com/BlueBubblesApp/bluebubbles-app
- **Original BlueBubbles Server:** https://github.com/BlueBubblesApp/BlueBubbles-Server

## Root Cause Analysis

### Why Stale Data Appears on Refresh

1. **No Local Persistence Layer**
   - Desktop/mobile apps use ObjectBox database for persistent storage
   - Web app stores all data in memory only (GetX RxLists)
   - Data is completely lost on page refresh

2. **Service Worker Limitations**
   - Caches static assets (JS, CSS, images) effectively
   - Does NOT cache dynamic data (messages, chats, contacts)
   - Index.html uses "online-first" strategy but data layer has no persistence

3. **Incomplete State Recovery**
   - On refresh: Empty services → asynchronous refetch from server
   - Race condition between UI loading and data fetching
   - WebSocket reconnection timing issues

4. **Contact Information Loss**
   - Contacts cached in `webCachedHandles` (RAM only)
   - Cleared on refresh until server responds
   - No fallback to cached contact names

### Technical Architecture Comparison

| Component | Desktop/Mobile | Web Implementation | Issue |
|-----------|----------------|-------------------|-------|
| Database | ObjectBox (persistent) | None (in-memory) | Data loss on refresh |
| Sync | Full + Incremental | WebSocket + REST | Dependent on server state |
| Contacts | Persistent storage | Server fetch only | Lost on refresh |
| Messages | Cached locally | Server fetch only | Lost on refresh |

## Hosting Recommendations for ProxMox Server

Since BlueBubbles-web is a static Flutter web app, you can host it on any web server. For your ProxMox setup:

### Recommended: Nginx
- Lightweight and efficient for static files
- Easy SSL termination
- Good caching capabilities

### Alternatives:
- Apache (if you need advanced routing)
- Docker container with nginx
- Caddy (modern, automatic HTTPS)

## Step-by-Step Setup Guides

### 1) Testing the Existing Code (As-Is)

#### Local Testing (Simplest Method)
```bash
# Navigate to your built web app directory
cd "e:\Data\FRITZ\Projects\BlueBubbles\bluebubbles-web"

# FIX: Update the base href in index.html for local testing
# Change <base href="/web/"> to <base href="/">
# This is needed because the app was built for deployment at /web/ path

# Start local web server (Python 3)
python -m http.server 8080

# Alternative: Node.js serve
npx serve .

# Alternative: PHP
php -S localhost:8080
```

Then open `http://localhost:8080` in your browser.

#### ProxMox Server Setup

1. **Install nginx:**
   ```bash
   apt update
   apt install nginx
   ```

2. **Create app directory:**
   ```bash
   mkdir -p /var/www/bluebubbles-web
   ```

3. **Copy web app files:**
   ```bash
   # From your Windows machine to ProxMox server
   scp -r e:\Data\FRITZ\Projects\BlueBubbles\bluebubbles-web/* user@your-proxmox-server:/var/www/bluebubbles-web/
   ```

4. **Configure nginx site:**
   ```bash
   nano /etc/nginx/sites-available/bluebubbles-web
   ```
   
   Add this configuration:
   ```nginx
   server {
       listen 80;
       server_name your-domain.com;  # Or use IP address
       
       root /var/www/bluebubbles-web;
       index index.html;
       
       # Enable gzip compression
       gzip on;
       gzip_types text/css application/javascript text/javascript application/json;
       
       # Cache static assets
       location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
           expires 1y;
           add_header Cache-Control "public, immutable";
       }
       
       # Handle Flutter routing (SPA)
       location / {
           try_files $uri $uri/ /index.html;
       }
   }
   ```

5. **Enable the site:**
   ```bash
   ln -s /etc/nginx/sites-available/bluebubbles-web /etc/nginx/sites-enabled/
   nginx -t  # Test configuration
   systemctl reload nginx
   ```

6. **Access your app:**
   - Visit `http://your-proxmox-server-ip` or your configured domain
   - The app should connect to your existing server at `https://bluebubbles.fritzhurst.com`

### 2) Development Environment Setup (For Making Changes)

#### Prerequisites
1. **Flutter Installation:**
   - Download from https://flutter.dev/docs/get-started/install
   - Enable web support: `flutter config --enable-web`

2. **Clone your forked repository:**
   ```bash
   git clone https://github.com/fritzhurst/bluebubbles-app.git
   cd bluebubbles-app
   ```

3. **Install dependencies:**
   ```bash
   flutter pub get
   ```

4. **Install build system:**
   ```bash
   flutter pub global activate peanut
   ```

#### Implementing Local Storage (Core Fix)

The main issue is in the `lib/database/html/` files. Here's the required implementation:

**Modify `lib/database/html/message.dart`:**
```dart
import 'package:idb_shim/idb.dart' as idb;

class MessageCache {
  static idb.Database? _db;
  
  static Future<idb.Database> get database async {
    if (_db != null) return _db!;
    _db = await idb.idbFactory.open('BlueBubblesMessages', version: 1, onUpgradeNeeded: (event) {
      final db = event.database;
      if (!db.objectStoreNames.contains('messages')) {
        db.createObjectStore('messages', keyPath: 'guid');
      }
    });
    return _db!;
  }
  
  static Future<void> saveMessage(Message message) async {
    final db = await database;
    final txn = db.transaction('messages', idb.idbModeReadWrite);
    final store = txn.objectStore('messages');
    await store.put(message.toMap());
    await txn.completed;
  }
  
  static Future<List<Message>> loadMessages(String chatGuid) async {
    final db = await database;
    final txn = db.transaction('messages', idb.idbModeReadOnly);
    final store = txn.objectStore('messages');
    final index = store.index('chatGuid'); // Add this index to store
    final results = await index.getAll(chatGuid);
    await txn.completed;
    return results.map((m) => Message.fromMap(m)).toList();
  }
}

// Update Message.save() to call MessageCache.saveMessage(this)
```

**Similar implementations needed for:**
- `lib/database/html/chat.dart` (chat persistence)
- `lib/database/html/handle.dart` (contact/handle caching)
- `lib/database/html/contact.dart` (contact data caching)

#### Build and Deploy Process

1. **Make your code changes** in the `bluebubbles-app` repository

2. **Build the web app:**
   ```bash
   flutter pub global run peanut:peanut --web-renderer=canvaskit
   ```

3. **Deploy to ProxMox:**
   ```bash
   # Copy build output to server
   scp -r build/web/* user@proxmox-server:/var/www/bluebubbles-web/
   ```

4. **Reload nginx:**
   ```bash
   systemctl reload nginx
   ```

#### Testing Your Changes

1. **Data Persistence Test:**
   - Load messages in web app
   - Refresh browser
   - Verify messages appear instantly (from cache)

2. **Contact Persistence Test:**
   - Ensure contact names appear immediately on refresh

3. **Sync Functionality Test:**
   - Send message from another device
   - Check if it appears in web app without full refresh

4. **Performance Test:**
   - Time page load after refresh (should be instant with caching)

## Next Steps

1. **Set up ProxMox environment** using the nginx configuration above
2. **Test current behavior** to confirm the issues
3. **Implement local storage** in your forked repository
4. **Build and deploy** updated version
5. **Test fixes** thoroughly

## Additional Notes

- The web app uses CanvasKit renderer for better performance
- Service worker provides PWA capabilities but doesn't cache data
- All data persistence logic exists in desktop code - needs adaptation for IndexedDB
- Consider implementing a hybrid approach: cache recent messages, fetch older ones as needed

## Quick Reference: What to Expect When Testing

### On ProxMox Deployment:
- ✅ **OAuth should work** (configured for production domains)
- ✅ **App loads and connects** to your server at `https://bluebubbles.fritzhurst.com`
- ❌ **Data will be lost** on browser refresh (this is the issue we're fixing)
- ❌ **Contacts will disappear** until server responds again
- ❌ **Messages will show stale data** from last sync

### After Implementing Local Storage:
- ✅ **Messages persist** across browser refreshes
- ✅ **Contacts load instantly** from cache
- ✅ **App feels responsive** like the desktop/mobile versions
- ✅ **Incremental sync** works properly with cached data

### Testing Checklist:
- [ ] Deploy to ProxMox with nginx
- [ ] Access via domain/IP
- [ ] Log in with Google
- [ ] Load messages and contacts
- [ ] Refresh browser → observe data loss
- [ ] Implement IndexedDB caching
- [ ] Re-deploy and test persistence