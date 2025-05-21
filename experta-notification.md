# Experta Platform - Notification Module Documentation

## 1. Module Overview

The Notification Module is a critical component of the Experta platform responsible for delivering timely and relevant push notifications to users across both mobile (Flutter) and web (React/Next.js) applications. This module leverages Firebase Cloud Messaging (FCM) as the primary notification delivery service due to its cross-platform capabilities, reliability, and scalability.

### Purpose

The Notification Module serves as a centralized system for:
- Receiving notification triggers from various microservices
- Formatting notification content according to standardized templates
- Delivering notifications to the appropriate user devices
- Tracking notification delivery status
- Managing notification preferences and user subscriptions

### Integration Points

The Notification Module integrates with the following microservices within the Experta ecosystem:

- **Wallet MS**: For financial transaction notifications (top-ups, debits, withdrawals)
- **Booking MS**: For appointment and session-related notifications
- **Call MS**: For real-time and scheduled call notifications
- **Feedback MS**: For review and rating notifications
- **Posts MS**: For social interaction notifications
- **Profiles MS**: For user profile updates and verification notifications
- **Feeds MS**: For content recommendation notifications

## 2. Integration Architecture

The Notification Module implements an event-driven architecture using a pub/sub pattern to ensure loose coupling between services and reliable notification delivery.

### Architecture Diagram

```
┌────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌──────────────────┐
│                │     │                 │     │                 │     │                  │
│  Microservices │────▶│  Event Bus/Queue│────▶│ Notification MS │────▶│ Firebase Cloud   │
│                │     │                 │     │                 │     │ Messaging (FCM)  │
└────────────────┘     └─────────────────┘     └─────────────────┘     └──────────────────┘
                                                       │                         │
                                                       │                         │
                                                       ▼                         ▼
                                              ┌─────────────────┐      ┌──────────────────┐
                                              │                 │      │                  │
                                              │ Notification DB │      │ Client Devices   │
                                              │                 │      │                  │
                                              └─────────────────┘      └──────────────────┘
```

### Implementation Details

1. **Event Publication**:
   - Each microservice publishes notification events to a centralized message bus/queue (e.g., RabbitMQ, Kafka, or AWS SNS)
   - Events contain all necessary data for constructing the notification

2. **Event Consumption**:
   - The Notification MS subscribes to relevant notification events
   - Upon receiving an event, it processes and transforms it into an appropriate FCM message

3. **Notification Dispatch**:
   - Formatted notifications are sent to FCM for delivery
   - Delivery attempts are logged in the Notification DB

4. **Client Reception**:
   - Mobile and web clients receive and process notifications according to platform-specific guidelines

### Technical Specifications

- **NodeJS Backend**: The Notification MS is implemented in Node.js for optimal performance with Firebase
- **Database**: MongoDB for storing notification templates, delivery logs, and user device tokens
- **Message Queue**: RabbitMQ for reliable event distribution between services
- **FCM Admin SDK**: For server-side integration with Firebase Cloud Messaging

## 3. Event-to-Notification Mapping

| Triggering Service | Event                      | Notification Title                  | Notification Body                                           | Receiver               | Timing   | Priority | Retry Strategy | Fallback Channels | TTL    | User Preference Category |
|--------------------|----------------------------|-------------------------------------|-------------------------------------------------------------|------------------------|----------|----------|----------------|-------------------|--------|--------------------------|
| **Wallet MS**      | Wallet top-up completed    | "Wallet Topped Up"                 | "Your wallet has been topped up with {amount} {currency}."  | Account owner          | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Financial              |
| **Wallet MS**      | Wallet debited             | "Payment Processed"                | "Your wallet has been charged {amount} {currency} for {service}." | Account owner          | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Financial              |
| **Wallet MS**      | Withdraw request submitted | "Withdrawal Request Received"      | "Your withdrawal request for {amount} {currency} is being processed." | Account owner          | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Financial              |
| **Wallet MS**      | Withdraw approved          | "Withdrawal Approved"              | "Your withdrawal of {amount} {currency} has been approved and will be processed shortly." | Account owner          | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Financial              |
| **Wallet MS**      | Withdraw completed         | "Withdrawal Successful"            | "Your withdrawal of {amount} {currency} has been processed successfully." | Account owner          | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Financial              |
| **Call MS**        | Incoming instant call      | "Incoming Call"                    | "{caller_name} is calling you now."                         | Call recipient         | Instant  | Critical | 5 attempts, 5s backoff | SMS | 30 sec | Calls (Cannot opt-out)  |
| **Call MS**        | Incoming scheduled call    | "Scheduled Call Starting"          | "Your scheduled call with {expert/client_name} is starting now." | Both parties           | Instant  | Critical | 5 attempts, 5s backoff | SMS, Email | 2 min | Calls (Cannot opt-out)  |
| **Call MS**        | Call joined                | "Call Joined"                      | "{participant_name} has joined the call."                   | Other participants     | Instant  | Medium   | 2 attempts, 10s backoff | None | 1 min  | Calls                   |
| **Call MS**        | Call ended                 | "Call Ended"                       | "Your call with {expert/client_name} has ended. Duration: {duration}." | Both parties           | Instant  | Medium   | 2 attempts, 10s backoff | None | 1 hour | Calls                   |
| **Call MS**        | Call missed                | "Missed Call"                      | "You missed a call from {caller_name}."                     | Call recipient         | Instant  | High     | 3 attempts, 30s backoff | Email | 1 day  | Calls                   |
| **Booking MS**     | Session reminder           | "Upcoming Session Reminder"        | "Your session with {expert/client_name} starts in {time_until} minutes." | Both parties           | Scheduled | High     | 3 attempts, 1min backoff | Email, SMS | 15 min | Sessions (Cannot opt-out) |
| **Booking MS**     | Session booked             | "New Session Booked"               | "A new session has been booked with you for {date} at {time}." | Expert                 | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Sessions                |
| **Booking MS**     | Session confirmed          | "Session Confirmed"                | "Your session on {date} at {time} has been confirmed."      | Client                 | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Sessions                |
| **Posts MS**       | Like on post               | "New Like"                         | "{user_name} liked your post."                              | Post owner             | Batched  | Low      | 1 attempt, no backoff | None | 3 days | Social                  |
| **Posts MS**       | Comment on post            | "New Comment"                      | "{user_name} commented on your post: '{comment_preview}'"   | Post owner             | Instant  | Medium   | 2 attempts, 5min backoff | None | 3 days | Social                  |
| **Posts MS**       | Share of post              | "Post Shared"                      | "{user_name} shared your post."                             | Post owner             | Batched  | Low      | 1 attempt, no backoff | None | 3 days | Social                  |
| **Feedback MS**    | Expert receives feedback   | "New Feedback Received"            | "You received new feedback from {client_name}. Rating: {rating}/5" | Expert                 | Instant  | High     | 3 attempts, 1min backoff | Email | 2 days | Feedback                |
| **Feedback MS**    | Client receives feedback   | "Expert Feedback Received"         | "{expert_name} has shared feedback about your session."     | Client                 | Instant  | Medium   | 2 attempts, 5min backoff | Email | 2 days | Feedback                |
| **Profiles MS**    | Profile verification       | "Profile Verification Update"      | "Your profile verification status has been updated to {status}." | Account owner          | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Account                 |
| **Expert Ranking MS** | Ranking change          | "Ranking Update"                   | "Your expert ranking has {increased/decreased} to {new_rank}." | Expert                 | Scheduled | Medium   | 2 attempts, 5min backoff | Email | 1 day  | Professional            |

## 4. Sample Payloads

### FCM Message Structure

All notifications follow a standardized FCM message format:

```json
{
  "message": {
    "token": "USER_DEVICE_TOKEN",
    "notification": {
      "title": "Notification Title",
      "body": "Notification Body"
    },
    "data": {
      "type": "NOTIFICATION_TYPE",
      "entityId": "RELATED_ENTITY_ID",
      "metadata": {
        // Additional contextual data specific to the notification type
      }
    },
    "android": {
      "priority": "high",
      "notification": {
        "channel_id": "CHANNEL_ID"
      }
    },
    "apns": {
      "payload": {
        "aps": {
          "sound": "default",
          "category": "CATEGORY"
        }
      }
    },
    "webpush": {
      "notification": {
        "icon": "https://experta.com/favicon.ico"
      },
      "fcm_options": {
        "link": "DEEP_LINK_URL"
      }
    }
  }
}
```

### Sample Notification Examples

#### Wallet Top-up Notification

```json
{
  "message": {
    "token": "user_device_token_123",
    "notification": {
      "title": "Wallet Topped Up",
      "body": "Your wallet has been topped up with $50.00 USD."
    },
    "data": {
      "type": "WALLET_TOPUP",
      "entityId": "transaction_123456",
      "metadata": {
        "amount": "50.00",
        "currency": "USD",
        "balanceAfter": "250.00",
        "timestamp": "2025-04-02T10:15:30Z"
      }
    },
    "android": {
      "priority": "high",
      "notification": {
        "channel_id": "financial_updates"
      }
    },
    "apns": {
      "payload": {
        "aps": {
          "sound": "default",
          "category": "FINANCIAL"
        }
      }
    },
    "webpush": {
      "notification": {
        "icon": "https://experta.com/assets/icons/wallet.png"
      },
      "fcm_options": {
        "link": "https://experta.com/dashboard/wallet/transactions"
      }
    }
  }
}
```

#### Incoming Call Notification

```json
{
  "message": {
    "token": "user_device_token_456",
    "notification": {
      "title": "Incoming Call",
      "body": "John Doe is calling you now."
    },
    "data": {
      "type": "INCOMING_CALL",
      "entityId": "call_789012",
      "metadata": {
        "callerId": "user_123",
        "callerName": "John Doe",
        "callerAvatar": "https://experta.com/avatars/johndoe.jpg",
        "callType": "video",
        "timestamp": "2025-04-02T14:30:45Z",
        "roomId": "room_345678"
      }
    },
    "android": {
      "priority": "high",
      "notification": {
        "channel_id": "calls",
        "sound": "ringtone"
      }
    },
    "apns": {
      "payload": {
        "aps": {
          "sound": "ringtone.caf",
          "category": "CALL",
          "contentAvailable": true,
          "mutableContent": true
        }
      }
    },
    "webpush": {
      "notification": {
        "icon": "https://experta.com/assets/icons/call.png",
        "actions": [
          {
            "action": "accept",
            "title": "Accept"
          },
          {
            "action": "decline",
            "title": "Decline"
          }
        ]
      },
      "fcm_options": {
        "link": "https://experta.com/call/join/room_345678"
      }
    }
  }
}
```

#### Post Interaction Notification

```json
{
  "message": {
    "token": "user_device_token_789",
    "notification": {
      "title": "New Comment",
      "body": "Maria Garcia commented on your post: 'This is really insightful, thanks for sharing!'"
    },
    "data": {
      "type": "POST_COMMENT",
      "entityId": "comment_234567",
      "metadata": {
        "postId": "post_123456",
        "commenterId": "user_456",
        "commenterName": "Maria Garcia",
        "commenterAvatar": "https://experta.com/avatars/mariagarcia.jpg",
        "commentText": "This is really insightful, thanks for sharing!",
        "timestamp": "2025-04-02T09:45:15Z"
      }
    },
    "android": {
      "priority": "normal",
      "notification": {
        "channel_id": "social_interactions"
      }
    },
    "apns": {
      "payload": {
        "aps": {
          "sound": "default",
          "category": "SOCIAL"
        }
      }
    },
    "webpush": {
      "notification": {
        "icon": "https://experta.com/assets/icons/comment.png"
      },
      "fcm_options": {
        "link": "https://experta.com/posts/post_123456?comment=comment_234567"
      }
    }
  }
}
```

## 5. Targeting Methods

The Notification Module employs three primary targeting methods to ensure notifications reach the intended recipients:

### Device Tokens

- **Description**: Direct targeting to specific devices using FCM registration tokens
- **Use Cases**:
  - User-specific notifications (e.g., wallet updates, incoming calls)
  - Critical notifications requiring delivery to all user devices
- **Implementation**:
  - Tokens are collected during app initialization and stored in the Profiles MS
  - Tokens are refreshed periodically and on user re-login
  - Multiple tokens per user are supported (for multi-device scenarios)

### Topics

- **Description**: Publish/subscribe model for group-based notifications
- **Use Cases**:
  - Category-based notifications (e.g., new content in specific expertise areas)
  - Broadcast announcements (e.g., platform updates, new features)
  - Expert-level specific notifications (e.g., all platinum experts)
- **Implementation**:
  - Topics follow the format: `{category}_{subCategory}_{optionalParameters}`
  - Examples:
    - `expertise_finance_tax`
    - `expertLevel_platinum`
    - `region_europe`
  - Users are automatically subscribed to relevant topics based on their profile settings

### User ID-based Routing

- **Description**: Sending to all devices associated with a specific user ID
- **Use Cases**:
  - Ensuring delivery across all user devices without managing individual tokens
  - Account-level notifications
- **Implementation**:
  - Requires additional configuration with Firebase User Management
  - Maps Experta user IDs to Firebase user IDs
  - Useful for multi-device and cross-platform scenarios

### Notification Registration Flow

1. Client app generates FCM token on startup
2. Token is sent to Profiles MS with device metadata
3. Profiles MS stores token in user's profile document
4. When sending notifications, Notification MS retrieves tokens from Profiles MS
5. For topic-based notifications, clients subscribe directly with FCM

## 6. Frontend Behavior

### Mobile App (Flutter)

#### Notification Handling

1. **Foreground Handling**:
   - Display in-app toast/banner for non-critical notifications
   - Show full-screen interface for critical notifications (calls)
   - Provide interactive elements (buttons) for actionable notifications

2. **Background Handling**:
   - Display system notification with appropriate icon and sound
   - Implement notification channels on Android for categorization
   - Use notification categories on iOS for grouping and actions

3. **Deep Linking**:
   - Define URI schemes for all notification types
   - Implement navigation to relevant screens when notifications are tapped
   - Preserve context and state during navigation

#### Code Example (Flutter)

```dart
// Initialize Firebase Messaging
Future<void> initFirebaseMessaging() async {
  final FirebaseMessaging messaging = FirebaseMessaging.instance;
  
  // Request permission (iOS)
  NotificationSettings settings = await messaging.requestPermission(
    alert: true,
    badge: true,
    sound: true,
  );
  
  // Get token
  String? token = await messaging.getToken();
  
  // Send token to backend
  if (token != null) {
    await ProfilesService.updateDeviceToken(token);
  }
  
  // Configure handlers
  FirebaseMessaging.onMessage.listen(_handleForegroundMessage);
  FirebaseMessaging.onMessageOpenedApp.listen(_handleNotificationTap);
  
  // Check for initial notification (cold start)
  RemoteMessage? initialMessage = await FirebaseMessaging.instance.getInitialMessage();
  if (initialMessage != null) {
    _handleNotificationTap(initialMessage);
  }
}

// Handle foreground message
void _handleForegroundMessage(RemoteMessage message) {
  String type = message.data['type'] ?? '';
  
  switch (type) {
    case 'INCOMING_CALL':
      // Show full-screen call UI
      NavigationService.pushNamed(
        Routes.incomingCall,
        arguments: CallArguments.fromNotification(message),
      );
      break;
    case 'WALLET_TOPUP':
    case 'WALLET_DEBIT':
      // Show toast notification
      NotificationService.showToast(
        message.notification?.title ?? '',
        message.notification?.body ?? '',
      );
      break;
    default:
      // Show in-app banner
      NotificationService.showBanner(
        title: message.notification?.title ?? '',
        body: message.notification?.body ?? '',
        onTap: () => _handleNotificationTap(message),
      );
  }
}

// Handle notification tap
void _handleNotificationTap(RemoteMessage message) {
  String type = message.data['type'] ?? '';
  String entityId = message.data['entityId'] ?? '';
  
  // Navigate based on notification type
  switch (type) {
    case 'INCOMING_CALL':
    case 'SCHEDULED_CALL':
      NavigationService.pushNamed(
        Routes.callRoom,
        arguments: CallRoomArguments(
          roomId: message.data['metadata']['roomId'],
        ),
      );
      break;
    case 'POST_COMMENT':
    case 'POST_LIKE':
    case 'POST_SHARE':
      NavigationService.pushNamed(
        Routes.postDetail,
        arguments: PostDetailArguments(
          postId: message.data['metadata']['postId'],
          commentId: type == 'POST_COMMENT' ? entityId : null,
        ),
      );
      break;
    // Additional cases for other notification types
  }
}
```

### Web App (React/Next.js)

#### Notification Handling

1. **Service Worker Configuration**:
   - Register FCM service worker for handling background notifications
   - Configure web push certificates in Firebase console
   - Implement notification click handlers in the service worker

2. **UI Integration**:
   - Display browser notifications for background messages
   - Show in-app notifications for foreground messages
   - Implement a notification center for historical notifications

3. **Permission Management**:
   - Request notification permissions on appropriate user action
   - Provide clear UX for enabling/disabling notifications
   - Respect user preferences across sessions

#### Code Example (React/Next.js)

```javascript
// firebase-messaging-sw.js (Service Worker)
importScripts('https://www.gstatic.com/firebasejs/9.0.0/firebase-app-compat.js');
importScripts('https://www.gstatic.com/firebasejs/9.0.0/firebase-messaging-compat.js');

firebase.initializeApp({
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "..."
});

const messaging = firebase.messaging();

// Handle background messages
messaging.onBackgroundMessage((payload) => {
  const notificationTitle = payload.notification.title;
  const notificationOptions = {
    body: payload.notification.body,
    icon: '/logo.png',
    data: payload.data,
  };

  self.registration.showNotification(notificationTitle, notificationOptions);
});

// Handle notification click
self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  
  const notificationData = event.notification.data;
  let url = '/dashboard';
  
  // Determine URL based on notification type
  switch (notificationData.type) {
    case 'INCOMING_CALL':
      url = `/call/join/${notificationData.metadata.roomId}`;
      break;
    case 'POST_COMMENT':
      url = `/posts/${notificationData.metadata.postId}?comment=${notificationData.entityId}`;
      break;
    // Additional cases for other notification types
  }
  
  // Focus or open a new window
  event.waitUntil(
    clients.matchAll({type: 'window'}).then((clientList) => {
      for (const client of clientList) {
        if (client.url === url && 'focus' in client) {
          return client.focus();
        }
      }
      if (clients.openWindow) {
        return clients.openWindow(url);
      }
    })
  );
});

// React Component for FCM setup
import { useEffect, useState } from 'react';
import { getMessaging, getToken, onMessage } from 'firebase/messaging';
import { initializeApp } from 'firebase/app';
import { firebaseConfig } from '../config/firebase';
import { useNotificationStore } from '../stores/notificationStore';
import { apiClient } from '../services/api';

export default function NotificationManager() {
  const [permission, setPermission] = useState('default');
  const { addNotification } = useNotificationStore();
  
  useEffect(() => {
    // Initialize Firebase
    const app = initializeApp(firebaseConfig);
    const messaging = getMessaging(app);
    
    // Check permission
    if ('Notification' in window) {
      setPermission(Notification.permission);
    }
    
    // Handle foreground messages
    const unsubscribe = onMessage(messaging, (payload) => {
      addNotification({
        id: payload.data.entityId,
        title: payload.notification.title,
        body: payload.notification.body,
        type: payload.data.type,
        metadata: JSON.parse(payload.data.metadata),
        read: false,
        timestamp: new Date().toISOString(),
      });
    });
    
    return () => unsubscribe();
  }, []);
  
  const requestPermission = async () => {
    if ('Notification' in window) {
      const permission = await Notification.requestPermission();
      setPermission(permission);
      
      if (permission === 'granted') {
        // Get token and send to backend
        const app = initializeApp(firebaseConfig);
        const messaging = getMessaging(app);
        
        getToken(messaging, { 
          vapidKey: 'YOUR_VAPID_KEY' 
        }).then((token) => {
          if (token) {
            apiClient.post('/profile/device-token', { token });
          }
        });
      }
    }
  };
  
  return (
    <div>
      {permission !== 'granted' && (
        <button onClick={requestPermission}>
          Enable Notifications
        </button>
      )}
    </div>
  );
}
```

## 7. Error Handling

### Retry Logic

1. **Per-Event Retry Strategy**:
   - Each notification type has a defined retry policy based on its priority and urgency
   - Retry configurations are stored alongside templates and include:
     - Maximum retry attempts (1-5 based on priority)
     - Initial backoff delay (5s-5min based on urgency)
     - Backoff multiplier (typically 1.5-2.0)
     - Maximum backoff cap
   - Examples:
     - **Critical (Calls)**: 5 attempts, 5s initial delay, 1.5x multiplier, 1min cap
     - **High (Wallet)**: 3 attempts, 1min initial delay, 2x multiplier, 5min cap
     - **Medium (Comments)**: 2 attempts, 5min initial delay, 2x multiplier, 15min cap
     - **Low (Likes)**: 1 attempt, no retry

2. **Circuit Breaker Implementation**:
   - Prevents overwhelming FCM during service degradation
   - Triggers when failure rate exceeds threshold (typically 25%)
   - Reduces notification volume for non-critical messages
   - Auto-resets after cooldown period (typically 15 minutes)
   - Critical notifications bypass circuit breaker

3. **Persistent Storage and Monitoring**:
   - All notification requests are persisted with complete retry history
   - Status tracked through complete lifecycle: pending → sent → delivered/failed
   - Retry stats aggregated for monitoring and alerting
   - Failed notification dashboard for support team visibility

### Token Invalidation

1. **Detection**:
   - FCM response monitoring for token invalidation errors
   - Error codes: `registration-token-not-registered`, `invalid-registration-token`
   - Batch token validation process runs weekly

2. **Handling**:
   - Invalid tokens are marked in the database
   - Notification to alternative user devices about token invalidation
   - Removal of invalid tokens after 30 days of inactivity

3. **Recovery**:
   - Re-registration prompts sent via alternative channels
   - Silent push to trigger token refresh on valid devices
   - User notification preferences preserved across token changes

### Delivery Failure Fallbacks

1. **Structured Fallback Chain**:
   - Each notification type has a defined fallback chain configuration
   - Fallback decisions based on:
     - Notification priority (Critical/High/Medium/Low)
     - User contact details availability
     - Delivery urgency and TTL
   - Fallback channels and rules:
     - **Critical (Calls)**: FCM → SMS (immediate) → Email (if SMS fails)
     - **High (Financial/Booking)**: FCM → Email (immediate)
     - **Medium (Social/Feedback)**: FCM → Email (delayed, batched)
     - **Low (Likes)**: FCM only, no fallback
   - Fallback throttling to prevent notification fatigue
     - SMS limited to 5 per day per user
     - Email fallbacks batched hourly for medium priority

2. **Notification Center as Guaranteed Delivery**:
   - In-app notification center serves as delivery guarantee
   - All notifications persisted regardless of push delivery status
   - Reads/views tracked and synchronized across devices
   - Pull-to-refresh mechanism to force notification sync
   - Background sync upon app open to ensure consistency

3. **Cross-Device State Synchronization**:
   - Notification state (read/unread/actioned) synchronized across all user devices
   - Central state management via Profiles MS
   - Read receipts sent back to Notification MS
   - Conflict resolution strategy:
     - Server timestamp as source of truth
     - Last-write-wins for conflicting updates
     - Periodic full sync for eventual consistency

### Error Handling Implementation

```javascript
// Node.js error handling for FCM
async function sendNotification(deviceToken, payload, options = {}) {
  const { 
    priority = 'high',
    retryCount = 3,
    retryDelay = 5000,
    fallbackChannels = ['email']
  } = options;
  
  const notificationRecord = await NotificationModel.create({
    token: deviceToken,
    payload,
    status: 'pending',
    priority,
    attempts: 0,
    maxAttempts: retryCount
  });
  
  try {
    // Send to FCM
    const response = await admin.messaging().send({
      token: deviceToken,
      ...payload
    });
    
    // Update status
    await NotificationModel.findByIdAndUpdate(notificationRecord._id, {
      status: 'sent',
      fcmMessageId: response.messageId,
      sentAt: new Date()
    });
    
    return response;
  } catch (error) {
    await handleFCMError(error, notificationRecord, fallbackChannels);
  }
}

async function handleFCMError(error, notificationRecord, fallbackChannels) {
  // Increment attempt counter
  notificationRecord.attempts += 1;
  
  if (error.code === 'messaging/registration-token-not-registered' || 
      error.code === 'messaging/invalid-registration-token') {
    // Token is invalid - mark for removal
    await DeviceTokenModel.findOneAndUpdate(
      { token: notificationRecord.token },
      { isValid: false, invalidatedAt: new Date() }
    );
    
    // Try alternative devices for same user
    if (notificationRecord.userId) {
      const alternativeTokens = await DeviceTokenModel.find({
        userId: notificationRecord.userId,
        isValid: true,
        token: { $ne: notificationRecord.token }
      });
      
      if (alternativeTokens.length > 0) {
        // Clone notification to alternative device
        const newNotification = new NotificationModel({
          ...notificationRecord.toObject(),
          _id: undefined,
          token: alternativeTokens[0].token,
          attempts: 0,
          status: 'pending',
          previousNotificationId: notificationRecord._id
        });
        await newNotification.save();
        
        // Mark original as redirected
        notificationRecord.status = 'redirected';
        await notificationRecord.save();
        return;
      }
    }
  } else if (error.code === 'messaging/server-unavailable' || 
             error.code === 'messaging/internal-error') {
    // Server error - retry if attempts remain
    if (notificationRecord.attempts < notificationRecord.maxAttempts) {
      notificationRecord.status = 'retry_scheduled';
      notificationRecord.next

## 8. Template Management

### Template Structure

Templates are stored in the database with the following enhanced structure:

```json
{
  "templateId": "wallet_topup",
  "version": "1.2.0",
  "status": "active",
  "testGroups": ["control", "variant_a", "variant_b"],
  "ttl": 86400,  // Time-to-live in seconds (1 day)
  "title": {
    "en": "Wallet Topped Up",
    "es": "Saldo Añadido a la Billetera",
    "fr": "Portefeuille Rechargé",
    "hi": "वॉलेट में राशि जमा की गई"
  },
  "body": {
    "en": "Your wallet has been topped up with {{amount}} {{currency}}.",
    "es": "Se han añadido {{amount}} {{currency}} a tu billetera.",
    "fr": "Votre portefeuille a été rechargé de {{amount}} {{currency}}.",
    "hi": "आपके वॉलेट में {{amount}} {{currency}} जमा किए गए हैं।"
  },
  "languageFallbackChain": ["user_preferred", "en", "es", "fr"],
  "metadata": {
    "category": "financial",
    "priority": "high",
    "userPreferenceCategory": "Financial",
    "android": {
      "channelId": "financial_updates",
      "smallIcon": "ic_wallet_notification",
      "color": "#4CAF50"
    },
    "ios": {
      "sound": "default",
      "badge": 1,
      "category": "FINANCIAL",
      "interruptionLevel": "timeSensitive"
    },
    "web": {
      "icon": "/assets/icons/wallet.png",
      "badge": "/assets/badges/wallet.png",
      "actions": [
        {
          "action": "view_transaction",
          "title": {
            "en": "View",
            "es": "Ver",
            "fr": "Voir",
            "hi": "देखें"
          }
        }
      ]
    }
  },
  "deepLinks": {
    "android": "experta://wallet/transactions?id={{transactionId}}",
    "ios": "experta://wallet/transactions?id={{transactionId}}",
    "web": "/dashboard/wallet/transactions?id={{transactionId}}"
  },
  "retryPolicy": {
    "maxAttempts": 3,
    "initialDelayMs": 60000,
    "backoffMultiplier": 2,
    "maxDelayMs": 300000
  },
  "fallbackChannels": ["email"],
  "conditions": {
    "userType": ["client", "expert"],
    "minAmount": 1.0,
    "checkUserPreference": true
  },
  "abTestConfig": {
    "enabled": true,
    "testId": "wallet_topup_template_test_01",
    "variants": {
      "control": {
        "weight": 0.6,
        "title": {
          "en": "Wallet Topped Up"
        },
        "body": {
          "en": "Your wallet has been topped up with {{amount}} {{currency}}."
        }
      },
      "variant_a": {
        "weight": 0.2,
        "title": {
          "en": "Funds Added!"
        },
        "body": {
          "en": "{{amount}} {{currency}} has been added to your wallet."
        }
      },
      "variant_b": {
        "weight": 0.2,
        "title": {
          "en": "Wallet Balance Increased"
        },
        "body": {
          "en": "Your wallet balance increased by {{amount}} {{currency}}."
        }
      }
    }
  },
  "analytics": {
    "trackOpen": true,
    "trackClick": true,
    "conversionEvents": ["wallet_viewed", "transaction_details_viewed"],
    "conversionTimeWindowMinutes": 30
  }
}
```

### Template Manager Service

The Template Manager Service provides the following enhanced functionalities:

1. **Template Rendering with Language Fallback Chain**:
   - Mustache-style variable interpolation with sanitization
   - Intelligent language selection based on fallback chain:
     1. User's preferred language (from profile)
     2. Custom fallback chain defined in template
     3. Default system fallback (e.g., hi → en)
   - Graceful handling of missing translations
   - Runtime error handling for variable interpolation
   - Personalization based on user context and preferences

2. **Template Validation and Testing**:
   - Strict schema validation for template structure
   - Variable validation against available context
   - Required fields enforcement per platform (iOS, Android, Web)
   - Preview generation with mock data
   - Content length validation (character limits per platform)
   - Automated test delivery to development devices

3. **A/B Testing Framework**:
   - Multiple template variants with weighted distribution
   - User segmentation capabilities (by cohort, region, etc.)
   - Performance tracking metrics:
     - Delivery rate
     - Open rate
     - Click-through rate (CTR)
     - Conversion events
   - Automated statistical analysis on variant performance
   - Winner auto-selection based on conversion goals
   - Gradual rollout of winning variants

4. **Template Administration**:
   - Comprehensive admin dashboard for template management
   - Version control with full history tracking
   - Visual editor with live preview
   - Scheduled publishing and expiration
   - Template duplication and bulk editing
   - Multi-language management interface
   - Role-based access control
   
5. **User Preference Enforcement**:
   - Integration with user notification preferences
   - Category-based opt-out respect
   - Handling of mandatory notifications (calls, critical updates)
   - Time-window preferences (quiet hours)
   - Frequency capping to prevent notification fatigue
   - Per-device preference management

### Implementation Example

```javascript
class NotificationTemplateManager {
  async renderTemplate(templateId, data, userPreferences) {
    // Get template from database
    const template = await TemplateModel.findOne({ templateId });
    if (!template) {
      throw new Error(`Template not found: ${templateId}`);
    }
    
    // Handle A/B testing if enabled
    let selectedTemplate = template;
    if (template.abTestConfig && template.abTestConfig.enabled) {
      selectedTemplate = await this.selectTemplateVariant(template, userPreferences.userId);
    }
    
    // Determine language based on fallback chain
    const userLanguage = userPreferences.language || 'en';
    const availableLanguages = Object.keys(selectedTemplate.title);
    
    // Apply language fallback chain
    let language = userLanguage;
    if (!availableLanguages.includes(userLanguage)) {
      // User language not available, try fallback chain
      for (const fallbackLang of selectedTemplate.languageFallbackChain) {
        if (fallbackLang === 'user_preferred') continue; // Skip as we already tried user's preference
        if (availableLanguages.includes(fallbackLang)) {
          language = fallbackLang;
          break;
        }
      }
      
      // If still no match, default to English or first available language
      if (!availableLanguages.includes(language)) {
        language = availableLanguages.includes('en') ? 'en' : availableLanguages[0];
      }
    }
    
    // Get title and body in appropriate language
    const title = selectedTemplate.title[language];
    const body = selectedTemplate.body[language];
    
    // Replace variables in title and body with sanitization
    const renderedTitle = this.interpolateVariables(title, data, true);
    const renderedBody = this.interpolateVariables(body, data, true);
    
    // Prepare platform-specific configuration
    const platformConfig = this.getPlatformConfig(selectedTemplate, userPreferences.deviceType);
    
    // Build deep link
    const deepLink = this.buildDeepLink(selectedTemplate.deepLinks, userPreferences.deviceType, data);
    
    // Track template variant for analytics
    if (template.abTestConfig && template.abTestConfig.enabled) {
      await this.trackVariantImpression(template.abTestConfig.testId, selectedTemplate.variantId, userPreferences.userId);
    }
    
    return {
      title: renderedTitle,
      body: renderedBody,
      metadata: selectedTemplate.metadata,
      platformConfig,
      deepLink,
      ttl: selectedTemplate.ttl || 86400, // Default to 24 hours if not specified
      retryPolicy: selectedTemplate.retryPolicy,
      fallbackChannels: selectedTemplate.fallbackChannels,
      abTestInfo: template.abTestConfig ? {
        testId: template.abTestConfig.testId,
        variantId: selectedTemplate.variantId
      } : null
    };
  }
  
  async selectTemplateVariant(template, userId) {
    // Deterministic variant selection based on userId to ensure consistency
    const variantNames = Object.keys(template.abTestConfig.variants);
    const hash = this.hashUserId(userId);
    
    // Weighted selection based on variant weights
    const weights = variantNames.map(name => template.abTestConfig.variants[name].weight);
    const variantName = this.weightedSelection(variantNames, weights, hash);
    
    // Merge base template with variant-specific overrides
    const variant = template.abTestConfig.variants[variantName];
    const mergedTemplate = { ...template };
    
    // Override title and body with variant-specific values where available
    if (variant.title) {
      mergedTemplate.title = { ...template.title, ...variant.title };
    }
    
    if (variant.body) {
      mergedTemplate.body = { ...template.body, ...variant.body };
    }
    
    // Add variant metadata for tracking
    mergedTemplate.variantId = variantName;
    
    return mergedTemplate;
  }
  
  interpolateVariables(text, data, sanitize = true) {
    if (!text) return '';
    
    try {
      return text.replace(/\{\{(\w+)\}\}/g, (match, variable) => {
        const value = data[variable];
        
        if (value === undefined) {
          return match; // Keep placeholder if variable not provided
        }
        
        // Sanitize value to prevent injection if requested
        return sanitize ? this.sanitizeValue(value) : value;
      });
    } catch (error) {
      console.error('Error interpolating variables:', error);
      return text; // Return original text on error
    }
  }
  
  sanitizeValue(value) {
    // Basic sanitization to prevent injection in notifications
    if (typeof value !== 'string') {
      return String(value);
    }
    
    return value
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#039;');
  }
  
  getPlatformConfig(template, deviceType) {
    switch (deviceType) {
      case 'android':
        return template.metadata.android;
      case 'ios':
        return template.metadata.ios;
      case 'web':
        return template.metadata.web;
      default:
        return {};
    }
  }
  
  buildDeepLink(deepLinks, deviceType, data) {
    if (!deepLinks) return null;
    
    const linkTemplate = deepLinks[deviceType] || deepLinks.web;
    if (!linkTemplate) return null;
    
    return this.interpolateVariables(linkTemplate, data);
  }
  
  hashUserId(userId) {
    // Simple hash function for deterministic variant selection
    let hash = 0;
    for (let i = 0; i < userId.length; i++) {
      hash = ((hash << 5) - hash) + userId.charCodeAt(i);
      hash |= 0; // Convert to 32bit integer
    }
    return Math.abs(hash);
  }
  
  weightedSelection(items, weights, seed) {
    // Weighted random selection with seed for determinism
    const totalWeight = weights.reduce((sum, weight) => sum + weight, 0);
    const threshold = (seed % 1000) / 1000 * totalWeight;
    
    let sum = 0;
    for (let i = 0; i < items.length; i++) {
      sum += weights[i];
      if (sum >= threshold) {
        return items[i];
      }
    }
    
    return items[items.length - 1]; // Fallback
  }
  
  async trackVariantImpression(testId, variantId, userId) {
    // Record variant impression for A/B test analysis
    await ABTestImpressionModel.create({
      testId,
      variantId,
      userId,
      timestamp: new Date()
    });
  }
}
```

## 9. Flow Diagrams

### Notification Lifecycle

```
sequenceDiagram
    participant MS as Microservice
    participant NMS as Notification MS
    participant FCM as Firebase Cloud Messaging
    participant Device as User Device
    
    MS->>NMS: Publish notification event
    Note over NMS: Validate event data
    NMS->>NMS: Fetch recipient device tokens
    NMS->>NMS: Apply notification template
    
    alt High Priority
        NMS->>FCM: Send notification (immediate)
    else Standard Priority
        NMS->>NMS: Add to batch queue
        Note over NMS: Process in batches
        NMS->>FCM: Send batch notification
    end
    
    FCM->>Device: Deliver notification
    
    alt Device Online
        Device->>FCM: Delivery receipt
        FCM->>NMS: Delivery status update
    else Device Offline
        Note over FCM: Store notification
        Note over Device: Receive on reconnect
    end
    
    Device->>NMS: Read/Interaction event
    NMS->>MS: Notify originating service (optional)
```

## 10. Technical Dependencies

### Required Libraries and Services

| Component | Technology | Purpose |
|-----------|------------|---------|
| Server Framework | Node.js/Express | Core notification service implementation |
| Database | MongoDB | Storing templates, logs, and notification status |
| Message Queue | RabbitMQ | Event distribution between microservices |
| Push Notifications | Firebase Cloud Messaging | Cross-platform notification delivery |
| Mobile SDK | Firebase SDK for Flutter | Client-side notification handling |
| Web SDK | Firebase JS SDK | Web push notification implementation |
| Monitoring | Prometheus/Grafana | Service monitoring and alerting |
| Logging | ELK Stack | Centralized logging for troubleshooting |

### API Dependencies

The Notification MS requires the following APIs from other microservices:

1. **Profiles MS**:
   - `GET /profiles/{userId}/devices` - Fetch user device tokens
   - `POST /profiles/device-token` - Register new FCM tokens

2. **Wallet MS**:
   - `GET /wallet/transactions/{transactionId}` - Get transaction details for deep links

3. **Call MS**:
   - `GET /calls/{callId}` - Get call details for incoming call notifications

4. **Feedback MS**:
   - `GET /feedback/{feedbackId}` - Get feedback details for notification content

### Configuration Requirements

```json
{
  "fcm": {
    "projectId": "experta-platform",
    "privateKeyPath": "/path/to/firebase-admin-sdk.json",
    "databaseURL": "https://experta-platform.firebaseio.com"
  },
  "rabbitmq": {
    "host": "amqp://rabbitmq:5672",
    "queues": {
      "notifications": "notification.events",
      "retries": "notification.retries"
    },
    "exchanges": {
      "events": "platform.events"
    },
    "routingKeys": {
      "wallet": "wallet.#",
      "call": "call.#",
      "booking": "booking.#",
      "post": "post.#",
      "feedback": "feedback.#",
      "profile": "profile.#"
    }
  },
  "mongodb": {
    "uri": "mongodb://mongodb:27017/notification-ms",
    "options": {
      "useNewUrlParser": true,
      "useUnifiedTopology": true
    }
  },
  "retryPolicy": {
    "critical": {
      "attempts": 5,
      "initialDelay": 5000,
      "maxDelay": 60000,
      "multiplier": 2
    },
    "standard": {
      "attempts": 3,
      "initialDelay": 60000,
      "maxDelay": 3600000,
      "multiplier": 4
    }
  },
  "fallbackChannels": {
    "enableEmail": true,
    "enableSMS": true,
    "smsThrottleLimit": 5
  }
}
```

## 11. Implementation Guidelines

### Setup and Configuration

1. **Firebase Project Setup**:
   - Create a Firebase project in the Firebase Console
   - Download admin SDK credentials
   - Configure FCM in Flutter and Web applications
   - Set up FCM Analytics for delivery tracking
   - Configure APNs certificates for iOS

2. **Microservice Integration**:
   - Implement event publishers in all relevant microservices
   - Standardize event payload structure across services
   - Add notification triggers to critical user flows
   - Implement fallback channel integrations (email, SMS)

3. **Database Collections**:

| Collection | Purpose | Key Fields |
|------------|---------|------------|
| templates | Notification templates | templateId, version, title, body, metadata, ttl, languageFallbackChain |
| templateVersions | Historical template versions | templateId, version, createdAt, createdBy, changeLog |
| notifications | Notification history | userId, type, status, timestamp, deviceTokens, readStatus, retryHistory |
| notificationDelivery | Delivery tracking | notificationId, deviceToken, status, deliveredAt, openedAt |
| deviceTokens | User device mappings | userId, token, platform, lastActive, isValid, appVersion |
| topics | Topic subscriptions | topicId, subscribedUsers, description |
| preferences | User notification settings | userId, categories, enabledStatus, quietHours, deviceSpecificOverrides |
| abTests | A/B test configurations | testId, templateId, variants, startDate, endDate, status |
| testResults | A/B test performance data | testId, variantId, impressions, opens, clicks, conversions |

4. **Admin Dashboard and Tools**:
   - **Notification Testing Console**:
     - Web-based tool for sending test notifications
     - Template preview with variable substitution
     - Device selector for targeted testing
     - Delivery status tracking
   
   - **Notification Monitoring Dashboard**:
     - Real-time delivery status visualization
     - Failure rate monitoring with alerting
     - Performance metrics by notification type
     - User engagement analytics
     
   - **Notification Replay Tool**:
     - CLI utility for developers to replay failed notifications
     - Batch replay capabilities for incident recovery
     - Delivery validation and confirmation
     
   - **Template Management UI**:
     - Web-based template editor with version control
     - Multi-language support with translation assistance
     - A/B test creation and monitoring
     - Template performance analytics

### Security Considerations

1. **Data Protection**:
   - Encrypt sensitive notification content
   - Avoid PII in notification payloads
   - Implement TTL for notification history

2. **Authentication**:
   - Secure token registration with user authentication
   - Validate topic subscriptions against user permissions
   - Use HTTPS for all FCM communication

3. **Rate Limiting**:
   - Implement per-user notification rate limits
   - Batch non-critical notifications
   - Monitor and prevent notification spam

### Monitoring and Analytics

1. **Key Performance Indicators**:
   - Notification delivery success rate
   - End-to-end delivery latency
   - User engagement metrics (open rate, action rate)

2. **Logging Strategy**:
   - Log all notification events
   - Track delivery status changes
   - Record user interactions with notifications

3. **Alerting Thresholds**:
   - Alert on sustained delivery failure rate > 5%
   - Alert on critical notification retry failures
   - Alert on FCM quota approaching limits

## 12. Conclusion

The Notification Module serves as a critical component of the Experta platform, enabling real-time communication and updates across all client applications. By leveraging Firebase Cloud Messaging, the module provides a reliable, scalable solution for delivering notifications to users across different devices and platforms.

This documentation provides a comprehensive overview of the module's architecture, implementation details, and integration points. Developers should follow these guidelines to ensure consistent, reliable notification delivery throughout the platform.

For additional information or questions, please contact the Notification Module team at `notification-team@experta.com`.

## 13. Appendix

### Common FCM Error Codes and Handling

| Error Code | Description | Handling Strategy |
|------------|-------------|-------------------|
| messaging/invalid-registration-token | Token is not valid for FCM | Remove token from database, request app to re-register |
| messaging/registration-token-not-registered | Token is no longer valid | Remove token from database, request app to re-register |
| messaging/message-rate-exceeded | Too many messages sent to a device | Implement exponential backoff, batch non-critical notifications |
| messaging/server-unavailable | FCM servers are temporarily unavailable | Retry with exponential backoff |
| messaging/internal-error | Internal error in FCM | Retry with exponential backoff |
| messaging/quota-exceeded | Project quota exceeded | Implement rate limiting, alert dev team |

### Android Notification Channels

Android 8.0+ requires notification channels. The following channels should be implemented:

| Channel ID | Name | Description | Importance |
|------------|------|-------------|------------|
| calls | Calls | Call notifications | HIGH (makes sound and appears as heads-up) |
| financial_updates | Financial Updates | Wallet transactions and financial alerts | DEFAULT (makes sound) |
| social_interactions | Social Interactions | Likes, comments, and shares on posts | LOW (no sound) |
| booking_updates | Booking Updates | Session bookings and reminders | DEFAULT (makes sound) |
| system_updates | System Updates | Platform updates and announcements | LOW (no sound) |

### iOS Notification Categories

For iOS, implement the following notification categories:

| Category ID | Purpose | Actions |
|-------------|---------|---------|
| CALL | Call notifications | Accept, Decline |
| FINANCIAL | Financial updates | View Transaction |
| SOCIAL | Social interactions | View, Reply |
| BOOKING | Session notifications | Confirm, Reschedule, Cancel |
| FEEDBACK | Feedback notifications | View, Respond |

### Web Push Implementation

For Progressive Web Apps, implement the following:

1. **Service Worker Registration**:
   ```javascript
   if ('serviceWorker' in navigator) {
     navigator.serviceWorker.register('/firebase-messaging-sw.js');
   }
   ```

2. **Web App Manifest**:
   ```json
   {
     "name": "Experta Platform",
     "short_name": "Experta",
     "start_url": "/",
     "display": "standalone",
     "gcm_sender_id": "103953800507"
   }
   ```

3. **VAPID Key Configuration**:
   Generate VAPID keys in Firebase Console and configure in the web application.

## 14. Additional Implementation Features

### Multi-Device Notification State Synchronization

To address the challenge of maintaining consistent notification state across multiple user devices, the Notification Module implements a robust synchronization mechanism:

1. **Centralized State Management**:
   - All notification states (delivered, read, actioned) stored in Notification MS
   - State changes timestamped with server time as source of truth
   - Read/action receipts sent from client apps to server

2. **Real-time Sync Mechanism**:
   - WebSocket connection for real-time state updates
   - Batched state updates for offline devices upon reconnection
   - Periodic full state sync to ensure eventual consistency

3. **Conflict Resolution**:
   - Last-write-wins based on server timestamp
   - Conflict detection based on state version tracking
   - Force sync to all devices when conflicts are resolved

4. **Implementation Details**:
   - Optimistic UI updates to prevent perceived latency
   - Sync queue for offline operations
   - Intelligent batching to reduce network overhead

### A/B Testing and Performance Optimization

The Notification Module includes built-in A/B testing capabilities to optimize notification effectiveness:

1. **Test Creation and Management**:
   - Create tests with multiple template variants
   - Assign distribution weights to variants
   - Set test duration and success metrics
   - Target specific user segments

2. **Variant Assignment**:
   - Deterministic user-to-variant assignment using userId hash
   - Consistent variant assignment across multiple notifications in same test
   - Override capability for testing and troubleshooting

3. **Performance Tracking**:
   - Delivery rate tracking by variant
   - Open rate comparison
   - Click-through rate (CTR) analysis
   - Conversion event correlation
   - Engagement time analysis

4. **Statistical Analysis**:
   - Automated significance testing
   - Confidence interval calculation
   - Conversion lift calculation
   - Platform and demographic segmentation

5. **Continuous Optimization**:
   - Automated promotion of winning variants
   - Learnings documentation and template updates
   - Failing variant early termination

### Notification Admin Dashboard

A comprehensive admin dashboard provides tools for developers, testers, and support staff:

1. **Real-time Monitoring**:
   - FCM delivery status tracking
   - Error rate monitoring
   - Service health indicators
   - Queue depth visualization

2. **Test and Debug Tools**:
   - Send test notifications to specific devices
   - Delivery verification
   - Template preview with variable substitution
   - Notification replay for failed messages

3. **Analytics and Reporting**:
   - Delivery success rates by notification type
   - User engagement metrics
   - A/B test performance visualization
   - Conversion tracking
   - Notification volume trends

4. **User Support Tools**:
   - Notification audit trail by user
   - Delivery validation for support cases
   - User preference management
   - Device token management

### Time-to-Live (TTL) Management

Notifications are configured with appropriate Time-to-Live settings to ensure timely delivery:

1. **TTL Configuration**:
   - Each notification type has a defined TTL value
   - TTL controls how long FCM attempts delivery if device is offline
   - Values range from 30 seconds (calls) to 3 days (social notifications)

2. **Implementation**:
   - TTL set in FCM payload
   - Stored in notification record for tracking
   - Auto-expiry of notifications in notification center

3. **TTL Adjustment Logic**:
   - Dynamic TTL extension for critical notifications
   - TTL reduction for batched notifications to prevent stale delivery
   - TTL override capability for specific scenarios

4. **Expired Notification Handling**:
   - Automatic cleanup of expired notifications
   - Fallback delivery for critical notifications
   - Digest creation for expired low-priority notifications# Experta Platform - Notification Module Documentation

## 1. Module Overview

The Notification Module is a critical component of the Experta platform responsible for delivering timely and relevant push notifications to users across both mobile (Flutter) and web (React/Next.js) applications. This module leverages Firebase Cloud Messaging (FCM) as the primary notification delivery service due to its cross-platform capabilities, reliability, and scalability.

### Purpose

The Notification Module serves as a centralized system for:
- Receiving notification triggers from various microservices
- Formatting notification content according to standardized templates
- Delivering notifications to the appropriate user devices
- Tracking notification delivery status
- Managing notification preferences and user subscriptions

### Integration Points

The Notification Module integrates with the following microservices within the Experta ecosystem:

- **Wallet MS**: For financial transaction notifications (top-ups, debits, withdrawals)
- **Booking MS**: For appointment and session-related notifications
- **Call MS**: For real-time and scheduled call notifications
- **Feedback MS**: For review and rating notifications
- **Posts MS**: For social interaction notifications
- **Profiles MS**: For user profile updates and verification notifications
- **Feeds MS**: For content recommendation notifications

## 2. Integration Architecture

The Notification Module implements an event-driven architecture using a pub/sub pattern to ensure loose coupling between services and reliable notification delivery.

### Architecture Diagram

```
┌────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌──────────────────┐
│                │     │                 │     │                 │     │                  │
│  Microservices │────▶│  Event Bus/Queue│────▶│ Notification MS │────▶│ Firebase Cloud   │
│                │     │                 │     │                 │     │ Messaging (FCM)  │
└────────────────┘     └─────────────────┘     └─────────────────┘     └──────────────────┘
                                                       │                         │
                                                       │                         │
                                                       ▼                         ▼
                                              ┌─────────────────┐      ┌──────────────────┐
                                              │                 │      │                  │
                                              │ Notification DB │      │ Client Devices   │
                                              │                 │      │                  │
                                              └─────────────────┘      └──────────────────┘
```

### Implementation Details

1. **Event Publication**:
   - Each microservice publishes notification events to a centralized message bus/queue (e.g., RabbitMQ, Kafka, or AWS SNS)
   - Events contain all necessary data for constructing the notification

2. **Event Consumption**:
   - The Notification MS subscribes to relevant notification events
   - Upon receiving an event, it processes and transforms it into an appropriate FCM message

3. **Notification Dispatch**:
   - Formatted notifications are sent to FCM for delivery
   - Delivery attempts are logged in the Notification DB

4. **Client Reception**:
   - Mobile and web clients receive and process notifications according to platform-specific guidelines

### Technical Specifications

- **NodeJS Backend**: The Notification MS is implemented in Node.js for optimal performance with Firebase
- **Database**: MongoDB for storing notification templates, delivery logs, and user device tokens
- **Message Queue**: RabbitMQ for reliable event distribution between services
- **FCM Admin SDK**: For server-side integration with Firebase Cloud Messaging

## 3. Event-to-Notification Mapping

| Triggering Service | Event                      | Notification Title                  | Notification Body                                           | Receiver               | Timing   | Priority | Retry Strategy | Fallback Channels | TTL    | User Preference Category |
|--------------------|----------------------------|-------------------------------------|-------------------------------------------------------------|------------------------|----------|----------|----------------|-------------------|--------|--------------------------|
| **Wallet MS**      | Wallet top-up completed    | "Wallet Topped Up"                 | "Your wallet has been topped up with {amount} {currency}."  | Account owner          | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Financial              |
| **Wallet MS**      | Wallet debited             | "Payment Processed"                | "Your wallet has been charged {amount} {currency} for {service}." | Account owner          | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Financial              |
| **Wallet MS**      | Withdraw request submitted | "Withdrawal Request Received"      | "Your withdrawal request for {amount} {currency} is being processed." | Account owner          | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Financial              |
| **Wallet MS**      | Withdraw approved          | "Withdrawal Approved"              | "Your withdrawal of {amount} {currency} has been approved and will be processed shortly." | Account owner          | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Financial              |
| **Wallet MS**      | Withdraw completed         | "Withdrawal Successful"            | "Your withdrawal of {amount} {currency} has been processed successfully." | Account owner          | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Financial              |
| **Call MS**        | Incoming instant call      | "Incoming Call"                    | "{caller_name} is calling you now."                         | Call recipient         | Instant  | Critical | 5 attempts, 5s backoff | SMS | 30 sec | Calls (Cannot opt-out)  |
| **Call MS**        | Incoming scheduled call    | "Scheduled Call Starting"          | "Your scheduled call with {expert/client_name} is starting now." | Both parties           | Instant  | Critical | 5 attempts, 5s backoff | SMS, Email | 2 min | Calls (Cannot opt-out)  |
| **Call MS**        | Call joined                | "Call Joined"                      | "{participant_name} has joined the call."                   | Other participants     | Instant  | Medium   | 2 attempts, 10s backoff | None | 1 min  | Calls                   |
| **Call MS**        | Call ended                 | "Call Ended"                       | "Your call with {expert/client_name} has ended. Duration: {duration}." | Both parties           | Instant  | Medium   | 2 attempts, 10s backoff | None | 1 hour | Calls                   |
| **Call MS**        | Call missed                | "Missed Call"                      | "You missed a call from {caller_name}."                     | Call recipient         | Instant  | High     | 3 attempts, 30s backoff | Email | 1 day  | Calls                   |
| **Booking MS**     | Session reminder           | "Upcoming Session Reminder"        | "Your session with {expert/client_name} starts in {time_until} minutes." | Both parties           | Scheduled | High     | 3 attempts, 1min backoff | Email, SMS | 15 min | Sessions (Cannot opt-out) |
| **Booking MS**     | Session booked             | "New Session Booked"               | "A new session has been booked with you for {date} at {time}." | Expert                 | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Sessions                |
| **Booking MS**     | Session confirmed          | "Session Confirmed"                | "Your session on {date} at {time} has been confirmed."      | Client                 | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Sessions                |
| **Posts MS**       | Like on post               | "New Like"                         | "{user_name} liked your post."                              | Post owner             | Batched  | Low      | 1 attempt, no backoff | None | 3 days | Social                  |
| **Posts MS**       | Comment on post            | "New Comment"                      | "{user_name} commented on your post: '{comment_preview}'"   | Post owner             | Instant  | Medium   | 2 attempts, 5min backoff | None | 3 days | Social                  |
| **Posts MS**       | Share of post              | "Post Shared"                      | "{user_name} shared your post."                             | Post owner             | Batched  | Low      | 1 attempt, no backoff | None | 3 days | Social                  |
| **Feedback MS**    | Expert receives feedback   | "New Feedback Received"            | "You received new feedback from {client_name}. Rating: {rating}/5" | Expert                 | Instant  | High     | 3 attempts, 1min backoff | Email | 2 days | Feedback                |
| **Feedback MS**    | Client receives feedback   | "Expert Feedback Received"         | "{expert_name} has shared feedback about your session."     | Client                 | Instant  | Medium   | 2 attempts, 5min backoff | Email | 2 days | Feedback                |
| **Profiles MS**    | Profile verification       | "Profile Verification Update"      | "Your profile verification status has been updated to {status}." | Account owner          | Instant  | High     | 3 attempts, 1min backoff | Email | 1 day  | Account                 |
| **Expert Ranking MS** | Ranking change          | "Ranking Update"                   | "Your expert ranking has {increased/decreased} to {new_rank}." | Expert                 | Scheduled | Medium   | 2 attempts, 5min backoff | Email | 1 day  | Professional            |

## 4. Sample Payloads

### FCM Message Structure

All notifications follow a standardized FCM message format:

```json
{
  "message": {
    "token": "USER_DEVICE_TOKEN",
    "notification": {
      "title": "Notification Title",
      "body": "Notification Body"
    },
    "data": {
      "type": "NOTIFICATION_TYPE",
      "entityId": "RELATED_ENTITY_ID",
      "metadata": {
        // Additional contextual data specific to the notification type
      }
    },
    "android": {
      "priority": "high",
      "notification": {
        "channel_id": "CHANNEL_ID"
      }
    },
    "apns": {
      "payload": {
        "aps": {
          "sound": "default",
          "category": "CATEGORY"
        }
      }
    },
    "webpush": {
      "notification": {
        "icon": "https://experta.com/favicon.ico"
      },
      "fcm_options": {
        "link": "DEEP_LINK_URL"
      }
    }
  }
}
```

### Sample Notification Examples

#### Wallet Top-up Notification

```json
{
  "message": {
    "token": "user_device_token_123",
    "notification": {
      "title": "Wallet Topped Up",
      "body": "Your wallet has been topped up with $50.00 USD."
    },
    "data": {
      "type": "WALLET_TOPUP",
      "entityId": "transaction_123456",
      "metadata": {
        "amount": "50.00",
        "currency": "USD",
        "balanceAfter": "250.00",
        "timestamp": "2025-04-02T10:15:30Z"
      }
    },
    "android": {
      "priority": "high",
      "notification": {
        "channel_id": "financial_updates"
      }
    },
    "apns": {
      "payload": {
        "aps": {
          "sound": "default",
          "category": "FINANCIAL"
        }
      }
    },
    "webpush": {
      "notification": {
        "icon": "https://experta.com/assets/icons/wallet.png"
      },
      "fcm_options": {
        "link": "https://experta.com/dashboard/wallet/transactions"
      }
    }
  }
}
```

#### Incoming Call Notification

```json
{
  "message": {
    "token": "user_device_token_456",
    "notification": {
      "title": "Incoming Call",
      "body": "John Doe is calling you now."
    },
    "data": {
      "type": "INCOMING_CALL",
      "entityId": "call_789012",
      "metadata": {
        "callerId": "user_123",
        "callerName": "John Doe",
        "callerAvatar": "https://experta.com/avatars/johndoe.jpg",
        "callType": "video",
        "timestamp": "2025-04-02T14:30:45Z",
        "roomId": "room_345678"
      }
    },
    "android": {
      "priority": "high",
      "notification": {
        "channel_id": "calls",
        "sound": "ringtone"
      }
    },
    "apns": {
      "payload": {
        "aps": {
          "sound": "ringtone.caf",
          "category": "CALL",
          "contentAvailable": true,
          "mutableContent": true
        }
      }
    },
    "webpush": {
      "notification": {
        "icon": "https://experta.com/assets/icons/call.png",
        "actions": [
          {
            "action": "accept",
            "title": "Accept"
          },
          {
            "action": "decline",
            "title": "Decline"
          }
        ]
      },
      "fcm_options": {
        "link": "https://experta.com/call/join/room_345678"
      }
    }
  }
}
```

#### Post Interaction Notification

```json
{
  "message": {
    "token": "user_device_token_789",
    "notification": {
      "title": "New Comment",
      "body": "Maria Garcia commented on your post: 'This is really insightful, thanks for sharing!'"
    },
    "data": {
      "type": "POST_COMMENT",
      "entityId": "comment_234567",
      "metadata": {
        "postId": "post_123456",
        "commenterId": "user_456",
        "commenterName": "Maria Garcia",
        "commenterAvatar": "https://experta.com/avatars/mariagarcia.jpg",
        "commentText": "This is really insightful, thanks for sharing!",
        "timestamp": "2025-04-02T09:45:15Z"
      }
    },
    "android": {
      "priority": "normal",
      "notification": {
        "channel_id": "social_interactions"
      }
    },
    "apns": {
      "payload": {
        "aps": {
          "sound": "default",
          "category": "SOCIAL"
        }
      }
    },
    "webpush": {
      "notification": {
        "icon": "https://experta.com/assets/icons/comment.png"
      },
      "fcm_options": {
        "link": "https://experta.com/posts/post_123456?comment=comment_234567"
      }
    }
  }
}
```

## 5. Targeting Methods

The Notification Module employs three primary targeting methods to ensure notifications reach the intended recipients:

### Device Tokens

- **Description**: Direct targeting to specific devices using FCM registration tokens
- **Use Cases**:
  - User-specific notifications (e.g., wallet updates, incoming calls)
  - Critical notifications requiring delivery to all user devices
- **Implementation**:
  - Tokens are collected during app initialization and stored in the Profiles MS
  - Tokens are refreshed periodically and on user re-login
  - Multiple tokens per user are supported (for multi-device scenarios)

### Topics

- **Description**: Publish/subscribe model for group-based notifications
- **Use Cases**:
  - Category-based notifications (e.g., new content in specific expertise areas)
  - Broadcast announcements (e.g., platform updates, new features)
  - Expert-level specific notifications (e.g., all platinum experts)
- **Implementation**:
  - Topics follow the format: `{category}_{subCategory}_{optionalParameters}`
  - Examples:
    - `expertise_finance_tax`
    - `expertLevel_platinum`
    - `region_europe`
  - Users are automatically subscribed to relevant topics based on their profile settings

### User ID-based Routing

- **Description**: Sending to all devices associated with a specific user ID
- **Use Cases**:
  - Ensuring delivery across all user devices without managing individual tokens
  - Account-level notifications
- **Implementation**:
  - Requires additional configuration with Firebase User Management
  - Maps Experta user IDs to Firebase user IDs
  - Useful for multi-device and cross-platform scenarios

### Notification Registration Flow

1. Client app generates FCM token on startup
2. Token is sent to Profiles MS with device metadata
3. Profiles MS stores token in user's profile document
4. When sending notifications, Notification MS retrieves tokens from Profiles MS
5. For topic-based notifications, clients subscribe directly with FCM

## 6. Frontend Behavior

### Mobile App (Flutter)

#### Notification Handling

1. **Foreground Handling**:
   - Display in-app toast/banner for non-critical notifications
   - Show full-screen interface for critical notifications (calls)
   - Provide interactive elements (buttons) for actionable notifications

2. **Background Handling**:
   - Display system notification with appropriate icon and sound
   - Implement notification channels on Android for categorization
   - Use notification categories on iOS for grouping and actions

3. **Deep Linking**:
   - Define URI schemes for all notification types
   - Implement navigation to relevant screens when notifications are tapped
   - Preserve context and state during navigation

#### Code Example (Flutter)

```dart
// Initialize Firebase Messaging
Future<void> initFirebaseMessaging() async {
  final FirebaseMessaging messaging = FirebaseMessaging.instance;
  
  // Request permission (iOS)
  NotificationSettings settings = await messaging.requestPermission(
    alert: true,
    badge: true,
    sound: true,
  );
  
  // Get token
  String? token = await messaging.getToken();
  
  // Send token to backend
  if (token != null) {
    await ProfilesService.updateDeviceToken(token);
  }
  
  // Configure handlers
  FirebaseMessaging.onMessage.listen(_handleForegroundMessage);
  FirebaseMessaging.onMessageOpenedApp.listen(_handleNotificationTap);
  
  // Check for initial notification (cold start)
  RemoteMessage? initialMessage = await FirebaseMessaging.instance.getInitialMessage();
  if (initialMessage != null) {
    _handleNotificationTap(initialMessage);
  }
}

// Handle foreground message
void _handleForegroundMessage(RemoteMessage message) {
  String type = message.data['type'] ?? '';
  
  switch (type) {
    case 'INCOMING_CALL':
      // Show full-screen call UI
      NavigationService.pushNamed(
        Routes.incomingCall,
        arguments: CallArguments.fromNotification(message),
      );
      break;
    case 'WALLET_TOPUP':
    case 'WALLET_DEBIT':
      // Show toast notification
      NotificationService.showToast(
        message.notification?.title ?? '',
        message.notification?.body ?? '',
      );
      break;
    default:
      // Show in-app banner
      NotificationService.showBanner(
        title: message.notification?.title ?? '',
        body: message.notification?.body ?? '',
        onTap: () => _handleNotificationTap(message),
      );
  }
}

// Handle notification tap
void _handleNotificationTap(RemoteMessage message) {
  String type = message.data['type'] ?? '';
  String entityId = message.data['entityId'] ?? '';
  
  // Navigate based on notification type
  switch (type) {
    case 'INCOMING_CALL':
    case 'SCHEDULED_CALL':
      NavigationService.pushNamed(
        Routes.callRoom,
        arguments: CallRoomArguments(
          roomId: message.data['metadata']['roomId'],
        ),
      );
      break;
    case 'POST_COMMENT':
    case 'POST_LIKE':
    case 'POST_SHARE':
      NavigationService.pushNamed(
        Routes.postDetail,
        arguments: PostDetailArguments(
          postId: message.data['metadata']['postId'],
          commentId: type == 'POST_COMMENT' ? entityId : null,
        ),
      );
      break;
    // Additional cases for other notification types
  }
}
```

### Web App (React/Next.js)

#### Notification Handling

1. **Service Worker Configuration**:
   - Register FCM service worker for handling background notifications
   - Configure web push certificates in Firebase console
   - Implement notification click handlers in the service worker

2. **UI Integration**:
   - Display browser notifications for background messages
   - Show in-app notifications for foreground messages
   - Implement a notification center for historical notifications

3. **Permission Management**:
   - Request notification permissions on appropriate user action
   - Provide clear UX for enabling/disabling notifications
   - Respect user preferences across sessions

#### Code Example (React/Next.js)

```javascript
// firebase-messaging-sw.js (Service Worker)
importScripts('https://www.gstatic.com/firebasejs/9.0.0/firebase-app-compat.js');
importScripts('https://www.gstatic.com/firebasejs/9.0.0/firebase-messaging-compat.js');

firebase.initializeApp({
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "..."
});

const messaging = firebase.messaging();

// Handle background messages
messaging.onBackgroundMessage((payload) => {
  const notificationTitle = payload.notification.title;
  const notificationOptions = {
    body: payload.notification.body,
    icon: '/logo.png',
    data: payload.data,
  };

  self.registration.showNotification(notificationTitle, notificationOptions);
});

// Handle notification click
self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  
  const notificationData = event.notification.data;
  let url = '/dashboard';
  
  // Determine URL based on notification type
  switch (notificationData.type) {
    case 'INCOMING_CALL':
      url = `/call/join/${notificationData.metadata.roomId}`;
      break;
    case 'POST_COMMENT':
      url = `/posts/${notificationData.metadata.postId}?comment=${notificationData.entityId}`;
      break;
    // Additional cases for other notification types
  }
  
  // Focus or open a new window
  event.waitUntil(
    clients.matchAll({type: 'window'}).then((clientList) => {
      for (const client of clientList) {
        if (client.url === url && 'focus' in client) {
          return client.focus();
        }
      }
      if (clients.openWindow) {
        return clients.openWindow(url);
      }
    })
  );
});

// React Component for FCM setup
import { useEffect, useState } from 'react';
import { getMessaging, getToken, onMessage } from 'firebase/messaging';
import { initializeApp } from 'firebase/app';
import { firebaseConfig } from '../config/firebase';
import { useNotificationStore } from '../stores/notificationStore';
import { apiClient } from '../services/api';

export default function NotificationManager() {
  const [permission, setPermission] = useState('default');
  const { addNotification } = useNotificationStore();
  
  useEffect(() => {
    // Initialize Firebase
    const app = initializeApp(firebaseConfig);
    const messaging = getMessaging(app);
    
    // Check permission
    if ('Notification' in window) {
      setPermission(Notification.permission);
    }
    
    // Handle foreground messages
    const unsubscribe = onMessage(messaging, (payload) => {
      addNotification({
        id: payload.data.entityId,
        title: payload.notification.title,
        body: payload.notification.body,
        type: payload.data.type,
        metadata: JSON.parse(payload.data.metadata),
        read: false,
        timestamp: new Date().toISOString(),
      });
    });
    
    return () => unsubscribe();
  }, []);
  
  const requestPermission = async () => {
    if ('Notification' in window) {
      const permission = await Notification.requestPermission();
      setPermission(permission);
      
      if (permission === 'granted') {
        // Get token and send to backend
        const app = initializeApp(firebaseConfig);
        const messaging = getMessaging(app);
        
        getToken(messaging, { 
          vapidKey: 'YOUR_VAPID_KEY' 
        }).then((token) => {
          if (token) {
            apiClient.post('/profile/device-token', { token });
          }
        });
      }
    }
  };
  
  return (
    <div>
      {permission !== 'granted' && (
        <button onClick={requestPermission}>
          Enable Notifications
        </button>
      )}
    </div>
  );
}
```

## 7. Error Handling

### Retry Logic

1. **Per-Event Retry Strategy**:
   - Each notification type has a defined retry policy based on its priority and urgency
   - Retry configurations are stored alongside templates and include:
     - Maximum retry attempts (1-5 based on priority)
     - Initial backoff delay (5s-5min based on urgency)
     - Backoff multiplier (typically 1.5-2.0)
     - Maximum backoff cap
   - Examples:
     - **Critical (Calls)**: 5 attempts, 5s initial delay, 1.5x multiplier, 1min cap
     - **High (Wallet)**: 3 attempts, 1min initial delay, 2x multiplier, 5min cap
     - **Medium (Comments)**: 2 attempts, 5min initial delay, 2x multiplier, 15min cap
     - **Low (Likes)**: 1 attempt, no retry

2. **Circuit Breaker Implementation**:
   - Prevents overwhelming FCM during service degradation
   - Triggers when failure rate exceeds threshold (typically 25%)
   - Reduces notification volume for non-critical messages
   - Auto-resets after cooldown period (typically 15 minutes)
   - Critical notifications bypass circuit breaker

3. **Persistent Storage and Monitoring**:
   - All notification requests are persisted with complete retry history
   - Status tracked through complete lifecycle: pending → sent → delivered/failed
   - Retry stats aggregated for monitoring and alerting
   - Failed notification dashboard for support team visibility

### Token Invalidation

1. **Detection**:
   - FCM response monitoring for token invalidation errors
   - Error codes: `registration-token-not-registered`, `invalid-registration-token`
   - Batch token validation process runs weekly

2. **Handling**:
   - Invalid tokens are marked in the database
   - Notification to alternative user devices about token invalidation
   - Removal of invalid tokens after 30 days of inactivity

3. **Recovery**:
   - Re-registration prompts sent via alternative channels
   - Silent push to trigger token refresh on valid devices
   - User notification preferences preserved across token changes

### Delivery Failure Fallbacks

1. **Structured Fallback Chain**:
   - Each notification type has a defined fallback chain configuration
   - Fallback decisions based on:
     - Notification priority (Critical/High/Medium/Low)
     - User contact details availability
     - Delivery urgency and TTL
   - Fallback channels and rules:
     - **Critical (Calls)**: FCM → SMS (immediate) → Email (if SMS fails)
     - **High (Financial/Booking)**: FCM → Email (immediate)
     - **Medium (Social/Feedback)**: FCM → Email (delayed, batched)
     - **Low (Likes)**: FCM only, no fallback
   - Fallback throttling to prevent notification fatigue
     - SMS limited to 5 per day per user
     - Email fallbacks batched hourly for medium priority

2. **Notification Center as Guaranteed Delivery**:
   - In-app notification center serves as delivery guarantee
   - All notifications persisted regardless of push delivery status
   - Reads/views tracked and synchronized across devices
   - Pull-to-refresh mechanism to force notification sync
   - Background sync upon app open to ensure consistency

3. **Cross-Device State Synchronization**:
   - Notification state (read/unread/actioned) synchronized across all user devices
   - Central state management via Profiles MS
   - Read receipts sent back to Notification MS
   - Conflict resolution strategy:
     - Server timestamp as source of truth
     - Last-write-wins for conflicting updates
     - Periodic full sync for eventual consistency

### Error Handling Implementation

```javascript
// Node.js error handling for FCM
async function sendNotification(deviceToken, payload, options = {}) {
  const { 
    priority = 'high',
    retryCount = 3,
    retryDelay = 5000,
    fallbackChannels = ['email']
  } = options;
  
  const notificationRecord = await NotificationModel.create({
    token: deviceToken,
    payload,
    status: 'pending',
    priority,
    attempts: 0,
    maxAttempts: retryCount
  });
  
  try {
    // Send to FCM
    const response = await admin.messaging().send({
      token: deviceToken,
      ...payload
    });
    
    // Update status
    await NotificationModel.findByIdAndUpdate(notificationRecord._id, {
      status: 'sent',
      fcmMessageId: response.messageId,
      sentAt: new Date()
    });
    
    return response;
  } catch (error) {
    await handleFCMError(error, notificationRecord, fallbackChannels);
  }
}

async function handleFCMError(error, notificationRecord, fallbackChannels) {
  // Increment attempt counter
  notificationRecord.attempts += 1;
  
  if (error.code === 'messaging/registration-token-not-registered' || 
      error.code === 'messaging/invalid-registration-token') {
    // Token is invalid - mark for removal
    await DeviceTokenModel.findOneAndUpdate(
      { token: notificationRecord.token },
      { isValid: false, invalidatedAt: new Date() }
    );
    
    // Try alternative devices for same user
    if (notificationRecord.userId) {
      const alternativeTokens = await DeviceTokenModel.find({
        userId: notificationRecord.userId,
        isValid: true,
        token: { $ne: notificationRecord.token }
      });
      
      if (alternativeTokens.length > 0) {
        // Clone notification to alternative device
        const newNotification = new NotificationModel({
          ...notificationRecord.toObject(),
          _id: undefined,
          token: alternativeTokens[0].token,
          attempts: 0,
          status: 'pending',
          previousNotificationId: notificationRecord._id
        });
        await newNotification.save();
        
        // Mark original as redirected
        notificationRecord.status = 'redirected';
        await notificationRecord.save();
        return;
      }
    }
  } else if (error.code === 'messaging/server-unavailable' || 
             error.code === 'messaging/internal-error') {
    // Server error - retry if attempts remain
    if (notificationRecord.attempts < notificationRecord.maxAttempts) {
      notificationRecord.status = 'retry_scheduled';
      notificationRecord.nextRetryAt = new Date(Date.now() + exponentialBackoff(notificationRecord.attempts));
      await notificationRecord.save();
      return;
    }
  }
  
  // All FCM attempts failed - try fallback channels
  notificationRecord.status = 'fcm_failed';
  await notificationRecord.save();
  
  if (fallbackChannels.includes('email') && notificationRecord.userId) {
    await sendEmailFallback(notificationRecord);
  }
  
  if (fallbackChannels.includes('sms') && 
      notificationRecord.priority === 'high' && 
      notificationRecord.userPhone) {
    await sendSMSFallback(notificationRecord);
  }
}

function exponentialBackoff(attempt) {
  return Math.min(
    (Math.pow(2, attempt) * 5000) + (Math.random() * 1000), 
    300000 // Max 5 minutes
  );
}


## 7. Resilience & Retry Logic for Notification Delivery (Updated)

> ❌ Previous Gap: No guaranteed delivery or retry for critical alerts

### ✅ New Implementation

To ensure critical alerts are never silently dropped, the notification module now includes:

### 7.1 Retry Queue with RabbitMQ
- All outgoing notifications are pushed to **RabbitMQ** with delivery metadata.
- Retry logic uses **priority-based configurations**:
  - `critical`: Retry up to 5 times with 1m, 5m, 15m, 30m, 60m intervals.
  - `high`: Retry 3 times with exponential backoff.
  - `medium/low`: Retry once or fire-and-forget.

### 7.2 Fallback Logic Chain
- Based on notification type:
  - **Critical** (e.g., session reminders, OTP, wallet alerts): FCM → SMS → Email
  - **High** (e.g., KYC, booking updates): FCM → Email
  - **Medium**: FCM → batched Email (optional)
  - **Low**: FCM only
- SMS fallback uses transactional sender ID with country-aware formatting.

### 7.3 DB Logging and Audit Trail
- MongoDB stores each notification event with:
  - `status`: pending, sent, failed, retried
  - `type`, `target`, `channel`, `attempts`, `fallbackUsed`
  - `createdAt`, `lastAttemptAt`, `deliveredAt`
- Dashboard for support team to investigate failed alerts

### 7.4 In-App Notification Center as Backup
- All notifications, regardless of channel status, are logged to the **in-app inbox**
- Ensures fallback visibility even when push or email fail

### 7.5 Developer Hooks
- Retry queue emits lifecycle events:
  - `notification.sent`, `notification.failed`, `notification.retried`, `notification.fallback`
- Devs can hook for escalation, logs, or alerts

---

✅ This update ensures that critical platform flows (sessions, payments, auth) always reach the user and that failures are observable and recoverable.
