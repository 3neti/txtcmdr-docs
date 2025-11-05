# API Documentation

Comprehensive API reference for Text Commander SMS broadcasting system.

---

## Table of Contents

- [Authentication](#authentication)
- [SMS Endpoints](#sms-endpoints)
- [Scheduled Messages](#scheduled-messages)
- [Group Management](#group-management)
- [Contact Management](#contact-management)
- [Blacklist Management](#blacklist-management)
- [Error Responses](#error-responses)
- [Route Middleware](#route-middleware)

---

## Authentication

All API endpoints require authentication via Laravel Sanctum.

**Header:**
```
Authorization: Bearer {token}
```

**Obtaining a Token:**
```bash
POST /api/login
Content-Type: application/json

{
  "email": "admin@txtcmdr.local",
  "password": "password"
}
```

**Response:**
```json
{
  "token": "1|abc123...",
  "user": {
    "id": 1,
    "name": "Admin",
    "email": "admin@txtcmdr.local"
  }
}
```

---

## SMS Endpoints

### Send to Multiple Recipients

**Endpoint:** `POST /api/send`  
**Middleware:** `auth:sanctum`  
**Job Middleware:** `CheckBlacklist` (automatically filters blacklisted numbers)

Send SMS to one or more mobile numbers.

**Request:**
```json
{
  "recipients": "+639171234567,+639189876543",
  "message": "Hello Quezon City!",
  "sender_id": "QUEZON_CITY"
}
```

**Parameters:**
- `recipients` (required) - Comma-delimited string or array of mobile numbers
- `message` (required, max: 1600) - SMS content
- `sender_id` (optional, max: 11) - Branded sender ID

**Response (200 OK):**
```json
{
  "status": "queued",
  "count": 2,
  "recipients": ["+639171234567", "+639189876543"],
  "invalid_count": 0
}
```

**Note:** Blacklisted numbers are automatically filtered by `CheckBlacklist` job middleware before sending.

---

### Send to Multiple Groups

**Endpoint:** `POST /api/groups/send`  
**Middleware:** `auth:sanctum`  
**Job Middleware:** `CheckBlacklist` (cascades to all contacts)

Send SMS to all contacts in one or more groups.

**Request:**
```json
{
  "groups": "barangay-leaders,health-workers",
  "message": "Please attend the emergency meeting.",
  "sender_id": "QUEZON_CITY"
}
```

**Parameters:**
- `groups` (required) - Comma-delimited string or array of group names/IDs
- `message` (required, max: 1600) - SMS content
- `sender_id` (optional, max: 11) - Branded sender ID

**Response (200 OK):**
```json
{
  "status": "queued",
  "groups": [
    {
      "id": 1,
      "name": "barangay-leaders",
      "contacts": 25
    },
    {
      "id": 2,
      "name": "health-workers",
      "contacts": 15
    }
  ],
  "total_contacts": 40
}
```

---

## Scheduled Messages

### Schedule a Message

**Endpoint:** `POST /api/send/schedule`  
**Middleware:** `auth:sanctum`

Schedule an SMS for future delivery.

**Request:**
```json
{
  "recipients": "+639171234567,barangay-leaders",
  "message": "Reminder: Meeting tomorrow at 2PM",
  "scheduled_at": "2024-01-20T14:00:00+08:00",
  "sender_id": "QUEZON_CITY"
}
```

**Parameters:**
- `recipients` (required) - Phone numbers or group names (comma-delimited)
- `message` (required, max: 1600) - SMS content
- `scheduled_at` (required) - ISO 8601 datetime (must be in the future)
- `sender_id` (optional, max: 11) - Branded sender ID

**Response (201 Created):**
```json
{
  "id": 123,
  "message": "Reminder: Meeting tomorrow at 2PM",
  "sender_id": "QUEZON_CITY",
  "recipient_type": "mixed",
  "recipient_data": {
    "numbers": ["+639171234567"],
    "groups": [
      {
        "id": 1,
        "name": "barangay-leaders",
        "count": 25
      }
    ]
  },
  "scheduled_at": "2024-01-20T14:00:00+08:00",
  "status": "pending",
  "total_recipients": 26
}
```

---

### List Scheduled Messages

**Endpoint:** `GET /api/scheduled-messages`  
**Middleware:** `auth:sanctum`

**Query Parameters:**
- `status` (optional) - Filter by status: `pending`, `sent`, `failed`, `cancelled`
- `page` (optional) - Page number for pagination

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": 123,
      "message": "Reminder: Meeting tomorrow",
      "scheduled_at": "2024-01-20T14:00:00+08:00",
      "status": "pending",
      "total_recipients": 26
    }
  ],
  "current_page": 1,
  "total": 10
}
```

---

### Update Scheduled Message

**Endpoint:** `PUT /api/scheduled-messages/{id}`  
**Middleware:** `auth:sanctum`

Update a pending scheduled message (only if not yet sent).

**Request:**
```json
{
  "message": "Updated message",
  "scheduled_at": "2024-01-21T10:00:00+08:00"
}
```

**Response (200 OK):**
```json
{
  "id": 123,
  "message": "Updated message",
  "scheduled_at": "2024-01-21T10:00:00+08:00",
  "status": "pending"
}
```

---

### Cancel Scheduled Message

**Endpoint:** `POST /api/scheduled-messages/{id}/cancel`  
**Middleware:** `auth:sanctum`

**Response (200 OK):**
```json
{
  "message": "Scheduled message cancelled",
  "id": 123,
  "status": "cancelled"
}
```

---

## Group Management

### Create Group

**Endpoint:** `POST /api/groups`  
**Middleware:** `auth:sanctum`

**Request:**
```json
{
  "name": "barangay-leaders",
  "description": "Group of active barangay officials"
}
```

**Response (201 Created):**
```json
{
  "id": 1,
  "name": "barangay-leaders",
  "description": "Group of active barangay officials",
  "contacts_count": 0,
  "created_at": "2024-01-15T10:30:00Z"
}
```

---

### List Groups

**Endpoint:** `GET /api/groups`  
**Middleware:** `auth:sanctum`

**Response (200 OK):**
```json
[
  {
    "id": 1,
    "name": "barangay-leaders",
    "description": "Group of active barangay officials",
    "contacts_count": 25
  }
]
```

---

### Get Group

**Endpoint:** `GET /api/groups/{id}`  
**Middleware:** `auth:sanctum`

**Response (200 OK):**
```json
{
  "id": 1,
  "name": "barangay-leaders",
  "description": "Group of active barangay officials",
  "contacts_count": 25,
  "contacts": [
    {
      "id": 1,
      "mobile": "+639171234567",
      "name": "Juan Dela Cruz"
    }
  ]
}
```

---

### Update Group

**Endpoint:** `PUT /api/groups/{id}`  
**Middleware:** `auth:sanctum`

**Request:**
```json
{
  "name": "barangay-leaders-updated",
  "description": "Updated description"
}
```

**Response (200 OK):**
```json
{
  "id": 1,
  "name": "barangay-leaders-updated",
  "description": "Updated description",
  "contacts_count": 25
}
```

---

### Delete Group

**Endpoint:** `DELETE /api/groups/{id}`  
**Middleware:** `auth:sanctum`

**Response (200 OK):**
```json
{
  "message": "Group deleted successfully"
}
```

---

## Contact Management

### Add Contact to Group

**Endpoint:** `POST /api/groups/{id}/contacts`  
**Middleware:** `auth:sanctum`

**Request:**
```json
{
  "mobile": "0917 123 4567",
  "name": "Juan Dela Cruz",
  "tags": ["leader", "volunteer"]
}
```

**Response (201 Created):**
```json
{
  "id": 1,
  "mobile": "09171234567",
  "mobile_e164": "+639171234567",
  "name": "Juan Dela Cruz",
  "tags": ["leader", "volunteer"]
}
```

---

### List Contacts in Group

**Endpoint:** `GET /api/groups/{id}/contacts`  
**Middleware:** `auth:sanctum`

**Response (200 OK):**
```json
[
  {
    "id": 1,
    "mobile": "09171234567",
    "mobile_e164": "+639171234567",
    "name": "Juan Dela Cruz",
    "tags": ["leader"]
  }
]
```

---

### Update Contact

**Endpoint:** `PUT /api/groups/{group_id}/contacts/{contact_id}`  
**Middleware:** `auth:sanctum`

**Request:**
```json
{
  "mobile": "0918 765 4321",
  "name": "Juan Updated",
  "tags": ["leader", "volunteer"]
}
```

**Response (200 OK):**
```json
{
  "id": 1,
  "mobile": "09187654321",
  "mobile_e164": "+639187654321",
  "name": "Juan Updated",
  "tags": ["leader", "volunteer"]
}
```

---

### Delete Contact from Group

**Endpoint:** `DELETE /api/groups/{group_id}/contacts/{contact_id}`  
**Middleware:** `auth:sanctum`

**Response (200 OK):**
```json
{
  "message": "Contact removed from group"
}
```

---

## Blacklist Management

### Add to Blacklist (Admin)

**Endpoint:** `POST /api/blacklist`  
**Middleware:** `auth:sanctum`

Add a phone number to the blacklist/no-send list.

**Request:**
```json
{
  "mobile": "0917 123 4567",
  "reason": "User opted out"
}
```

**Response (201 Created):**
```json
{
  "message": "Number added to blacklist",
  "blacklisted": {
    "id": 1,
    "mobile": "+639171234567",
    "reason": "User opted out",
    "added_by": "Admin",
    "blocked_at": "2024-01-15T10:30:00Z"
  }
}
```

---

### Opt-Out (Public Endpoint)

**Endpoint:** `POST /api/optout`  
**Middleware:** NONE (public endpoint)

Allow recipients to opt-out of SMS broadcasts by adding their number to the blacklist.

**Request:**
```json
{
  "mobile": "0917 123 4567"
}
```

**Response (200 OK):**
```json
{
  "message": "You have been successfully opted out of SMS broadcasts",
  "mobile": "+639171234567"
}
```

**Usage Example:**
Include opt-out instructions in your SMS messages:
```
"Your message here. Reply STOP or visit https://txtcmdr.com/optout to unsubscribe."
```

**Note:** This endpoint does NOT require authentication to allow recipients to self-service opt-out.

---

### List Blacklisted Numbers

**Endpoint:** `GET /api/blacklist`  
**Middleware:** `auth:sanctum`

**Query Parameters:**
- `search` (optional) - Search by mobile number
- `reason` (optional) - Filter by reason: `opt-out`, `complaint`, `invalid`, `manual`

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "mobile": "+639171234567",
      "reason": "opt-out",
      "added_by": "System",
      "blocked_at": "2024-01-15T10:30:00Z"
    }
  ],
  "current_page": 1,
  "total": 10
}
```

---

### Remove from Blacklist

**Endpoint:** `DELETE /api/blacklist`  
**Middleware:** `auth:sanctum`

**Request:**
```json
{
  "mobile": "0917 123 4567"
}
```

**Response (200 OK):**
```json
{
  "message": "Number removed from blacklist"
}
```

**Response (404 Not Found):**
```json
{
  "message": "Number not found in blacklist"
}
```

---

### Check if Blacklisted

**Endpoint:** `POST /api/blacklist/check`  
**Middleware:** `auth:sanctum`

**Request:**
```json
{
  "mobile": "0917 123 4567"
}
```

**Response (200 OK):**
```json
{
  "is_blacklisted": true,
  "mobile": "+639171234567",
  "record": {
    "id": 1,
    "reason": "opt-out",
    "blocked_at": "2024-01-15T10:30:00Z"
  }
}
```

---

## Error Responses

### 401 Unauthorized

```json
{
  "message": "Unauthenticated."
}
```

### 422 Validation Error

```json
{
  "message": "The recipients field is required.",
  "errors": {
    "recipients": [
      "The recipients field is required."
    ],
    "message": [
      "The message must not exceed 1600 characters."
    ]
  }
}
```

### 404 Not Found

```json
{
  "message": "Group not found."
}
```

### 500 Internal Server Error

```json
{
  "message": "Server error",
  "error": "Detailed error message"
}
```

---

## Route Middleware

### Authentication Middleware

All API routes (except `/api/optout`) require authentication:

```php
// routes/api.php
Route::middleware('auth:sanctum')->group(function () {
    // SMS Broadcasting
    Route::post('/send', SendToMultipleRecipients::class);
    Route::post('/groups/send', SendToMultipleGroups::class);
    Route::post('/send/schedule', ScheduleMessage::class);
    
    // Scheduled Messages
    Route::get('/scheduled-messages', ListScheduledMessages::class);
    Route::put('/scheduled-messages/{id}', UpdateScheduledMessage::class);
    Route::post('/scheduled-messages/{id}/cancel', CancelScheduledMessage::class);
    
    // Group Management
    Route::get('/groups', ListGroups::class);
    Route::post('/groups', CreateGroup::class);
    Route::get('/groups/{id}', GetGroup::class);
    Route::put('/groups/{id}', UpdateGroup::class);
    Route::delete('/groups/{id}', DeleteGroup::class);
    
    // Contact Management
    Route::get('/groups/{id}/contacts', ListGroupContacts::class);
    Route::post('/groups/{id}/contacts', AddContactToGroup::class);
    Route::put('/groups/{group_id}/contacts/{contact_id}', UpdateContactInGroup::class);
    Route::delete('/groups/{group_id}/contacts/{contact_id}', DeleteContactFromGroup::class);
    
    // Blacklist Management (Admin)
    Route::get('/blacklist', ListBlacklistedNumbers::class);
    Route::post('/blacklist', AddToBlacklist::class);
    Route::delete('/blacklist', RemoveFromBlacklist::class);
    Route::post('/blacklist/check', CheckIfBlacklisted::class);
});

// Public opt-out endpoint (no authentication required)
Route::post('/optout', OptOut::class);
```

### Job Middleware

**CheckBlacklist** middleware is automatically applied to all SMS jobs:

```php
// app/Jobs/SendSMSJob.php
public function middleware(): array
{
    return [new CheckBlacklist];
}
```

This ensures:
- ✅ Every SMS is checked against blacklist before sending
- ✅ Blacklisted numbers are filtered automatically
- ✅ No code changes needed in Actions or Jobs
- ✅ Opt-out requests are immediately effective

---

## Related Documentation

- [Backend Services](backend-services.md) - Complete Action implementations
- [Blacklist Feature](blacklist-feature.md) - Detailed blacklist architecture
- [Scheduled Messaging](scheduled-messaging.md) - Scheduling system details
- [Frontend Scaffolding](frontend-scaffolding.md) - TypeScript types and Vue components
