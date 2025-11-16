# Advanced Push Notifications - SvelteKit PWAs

**Advanced notification patterns, service worker handlers, and testing**

## Service Worker Push Handling

```javascript
// src/service-worker.js - Push notification handlers

// Push event - receive notification
self.addEventListener('push', (event) => {
  if (!event.data) {
    console.log('Push event but no data');
    return;
  }
  
  const data = event.data.json();
  const options = {
    body: data.notification.body,
    icon: data.notification.icon || '/icon-192.png',
    badge: data.notification.badge || '/badge-72.png',
    vibrate: data.notification.vibrate || [200, 100, 200],
    data: data.notification.data,
    actions: data.notification.actions || [],
    tag: data.notification.tag || 'default',
    requireInteraction: data.notification.requireInteraction || false,
    renotify: data.notification.renotify || false,
    silent: data.notification.silent || false,
    timestamp: data.notification.timestamp || Date.now(),
    image: data.notification.image // Large image
  };
  
  event.waitUntil(
    self.registration.showNotification(
      data.notification.title,
      options
    )
  );
  
  // Update app badge if supported
  if ('setAppBadge' in navigator) {
    const unreadCount = data.notification.data?.unreadCount;
    if (unreadCount) {
      navigator.setAppBadge(unreadCount);
    }
  }
});

// Notification click - handle interaction
self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  
  const data = event.notification.data;
  let url = '/';
  
  // Handle action clicks
  if (event.action) {
    switch (event.action) {
      case 'view':
        url = data.viewUrl || '/';
        break;
      case 'dismiss':
        return; // Just close
      default:
        url = `/action/${event.action}`;
    }
  } else if (data.primaryKey) {
    url = `/item/${data.primaryKey}`;
  }
  
  event.waitUntil(
    clients.matchAll({ type: 'window' }).then((windowClients) => {
      // Check if app is already open
      for (const client of windowClients) {
        if (client.url === url && 'focus' in client) {
          return client.focus();
        }
      }
      
      // Open new window if not
      if (clients.openWindow) {
        return clients.openWindow(url);
      }
    })
  );
  
  // Clear badge when notification is clicked
  if ('clearAppBadge' in navigator) {
    navigator.clearAppBadge();
  }
});

// Push subscription change
self.addEventListener('pushsubscriptionchange', (event) => {
  event.waitUntil(
    self.registration.pushManager.subscribe(
      event.oldSubscription.options
    ).then((subscription) => {
      // Update subscription on server
      return fetch('/api/push/resubscribe', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          oldEndpoint: event.oldSubscription.endpoint,
          newSubscription: subscription
        })
      });
    })
  );
});
```

## Advanced Notification Features

### Rich Notifications

```javascript
// Notification with actions and image
const notification = {
  title: 'New Message from Alice',
  body: 'Hey, check out this photo from our trip!',
  icon: '/alice-avatar.png',
  badge: '/badge.png',
  image: '/trip-photo.jpg', // Large image
  actions: [
    {
      action: 'reply',
      title: 'Reply',
      icon: '/reply-icon.png'
    },
    {
      action: 'like',
      title: 'Like',
      icon: '/heart-icon.png'
    }
  ],
  data: {
    messageId: '123',
    senderId: 'alice',
    timestamp: Date.now()
  },
  tag: 'message-alice', // Groups notifications
  requireInteraction: true // Doesn't auto-dismiss
};
```

### Notification Patterns

```javascript
// lib/utils/notification-patterns.js

export class NotificationPatterns {
  // Silent notification (updates UI without disturbing)
  static silent(data) {
    return {
      ...data,
      silent: true,
      vibrate: []
    };
  }
  
  // Urgent notification
  static urgent(data) {
    return {
      ...data,
      requireInteraction: true,
      vibrate: [200, 100, 200, 100, 200],
      tag: 'urgent',
      renotify: true
    };
  }
  
  // Progress notification
  static progress(title, progress, max = 100) {
    return {
      title,
      body: `Progress: ${progress}/${max}`,
      silent: true,
      tag: 'progress',
      data: { progress, max }
    };
  }
  
  // Grouped notifications
  static grouped(messages) {
    if (messages.length === 1) {
      return {
        title: messages[0].sender,
        body: messages[0].text,
        tag: 'messages'
      };
    }
    
    return {
      title: `${messages.length} new messages`,
      body: messages.map(m => `${m.sender}: ${m.text}`).join('\n'),
      tag: 'messages',
      data: { messageIds: messages.map(m => m.id) }
    };
  }
}
```

### Background Notification Processing

```javascript
// Service worker - process notifications in background
self.addEventListener('push', async (event) => {
  const data = event.data.json();
  
  // Process based on type
  switch (data.type) {
    case 'chat':
      event.waitUntil(handleChatNotification(data));
      break;
    case 'sync':
      event.waitUntil(handleSyncNotification(data));
      break;
    case 'update':
      event.waitUntil(handleUpdateNotification(data));
      break;
    default:
      event.waitUntil(handleDefaultNotification(data));
  }
});

async function handleChatNotification(data) {
  // Get all notifications with same tag
  const notifications = await self.registration.getNotifications({
    tag: 'chat'
  });
  
  // Close old notifications
  notifications.forEach(n => n.close());
  
  // Group messages
  const messages = notifications.map(n => n.data).concat(data);
  
  // Show grouped notification
  await self.registration.showNotification(
    `${messages.length} new messages`,
    {
      body: 'Tap to view all messages',
      tag: 'chat',
      data: { messages }
    }
  );
}

async function handleSyncNotification(data) {
  // Silent sync - don't show notification
  const clients = await self.clients.matchAll();
  
  // Send message to all clients
  clients.forEach(client => {
    client.postMessage({
      type: 'BACKGROUND_SYNC',
      data: data.payload
    });
  });
}
```

## Testing Push Notifications

### Manual Testing

```javascript
// Debug component for testing
<!-- lib/components/NotificationTester.svelte -->
<script>
  async function sendTestNotification() {
    const registration = await navigator.serviceWorker.ready;
    
    // Local notification (no server required)
    registration.showNotification('Test Notification', {
      body: 'This is a test notification',
      icon: '/icon-192.png',
      badge: '/badge-72.png',
      actions: [
        { action: 'action1', title: 'Action 1' },
        { action: 'action2', title: 'Action 2' }
      ]
    });
  }
  
  async function testPushEvent() {
    // Simulate push event
    const registration = await navigator.serviceWorker.ready;
    
    registration.active.postMessage({
      type: 'PUSH_EVENT',
      data: {
        notification: {
          title: 'Simulated Push',
          body: 'This simulates a server push',
          data: { test: true }
        }
      }
    });
  }
</script>

<div class="notification-tester">
  <button on:click={sendTestNotification}>
    Test Local Notification
  </button>
  
  <button on:click={testPushEvent}>
    Test Push Event
  </button>
</div>
```

### Automated Testing

```javascript
// tests/push-notifications.test.js
import { test, expect } from '@playwright/test';

test.describe('Push Notifications', () => {
  test('requests permission when prompted', async ({ page, context }) => {
    // Grant permission in test
    await context.grantPermissions(['notifications']);
    
    await page.goto('/');
    
    // Click enable notifications
    await page.click('button:has-text("Enable Notifications")');
    
    // Check subscription saved
    const response = await page.waitForResponse('/api/push/subscribe');
    expect(response.status()).toBe(200);
  });
  
  test('shows notification on push', async ({ page, context }) => {
    await context.grantPermissions(['notifications']);
    
    await page.goto('/');
    
    // Inject service worker test
    await page.evaluate(() => {
      navigator.serviceWorker.controller.postMessage({
        type: 'SHOW_NOTIFICATION',
        data: {
          title: 'Test',
          body: 'Test notification'
        }
      });
    });
    
    // Check notification appears
    // Note: Browser automation of actual notifications is limited
  });
});
```

## Best Practices

### Permission Request Timing

```javascript
// Good: Request after user action or engagement
function requestAfterEngagement() {
  // Track user engagement
  let engagementScore = 0;
  
  // Increment on meaningful actions
  document.addEventListener('click', () => {
    engagementScore++;
    
    if (engagementScore > 5 && !notificationRequested) {
      showNotificationPrompt();
    }
  });
}

// Bad: Request immediately on page load
// This creates poor UX and likely denial
```

### Notification Content

```javascript
// Good: Relevant, timely, actionable
{
  title: 'Your order has shipped!',
  body: 'Track your package or change delivery',
  actions: [
    { action: 'track', title: 'Track Package' },
    { action: 'change', title: 'Change Delivery' }
  ]
}

// Bad: Generic, spammy, not actionable
{
  title: 'Check our app!',
  body: 'We have updates'
}
```

### Frequency Management

```javascript
// Implement notification throttling
class NotificationThrottle {
  constructor() {
    this.sent = new Map();
  }
  
  canSend(userId, type) {
    const key = `${userId}-${type}`;
    const lastSent = this.sent.get(key);
    
    if (!lastSent) return true;
    
    const limits = {
      marketing: 24 * 60 * 60 * 1000, // 1 day
      transaction: 0, // No limit
      reminder: 60 * 60 * 1000, // 1 hour
      social: 30 * 60 * 1000 // 30 minutes
    };
    
    const limit = limits[type] || 60 * 60 * 1000;
    return Date.now() - lastSent > limit;
  }
  
  markSent(userId, type) {
    this.sent.set(`${userId}-${type}`, Date.now());
  }
}
```

## Troubleshooting

### Common Issues

1. **Notifications not showing**
   - Check permission status
   - Verify HTTPS (required for production)
   - Check browser notification settings
   - Ensure service worker is active

2. **Subscription fails**
   - Verify VAPID keys match
   - Check applicationServerKey format
   - Ensure service worker registered

3. **Push not received**
   - Check server-side implementation
   - Verify subscription is current
   - Check push service status

4. **iOS/Safari limitations**
   - Requires iOS 16.4+
   - Must be installed as PWA
   - Limited API support

## References

- [Web Push Protocol](https://datatracker.ietf.org/doc/html/rfc8030)
- [Notification API](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API)
- [Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)
- [web-push library](https://github.com/web-push-libs/web-push)
