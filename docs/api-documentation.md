# API Documentation (Updated)

## 1. Send to Multiple Recipients

### POST /api/send

Send an SMS message to one or more mobile numbers.

**Request Body**
```json
{
  "recipients": "+639171234567,+639189876543",
  "message": "Hello Quezon City!",
  "sender_id": "Quezon City"
}
```

- `recipients`: comma-delimited string or array of mobile numbers
- `message`: the message content
- `sender_id` (optional): override default sender ID

---

## 2. Send to Multiple Groups

### POST /api/groups/send

Send a message to one or more groups.

**Request Body**
```json
{
  "groups": "barangay-leaders,health-workers",
  "message": "Please attend the emergency meeting.",
  "sender_id": "Quezon City"
}
```

- `groups`: comma-delimited string or array of group identifiers
- `message`: the message content
- `sender_id` (optional): override default sender ID

---

## 3. Group Management (CRUD)

### Create a Group
**POST /api/groups**
```json
{
  "name": "barangay-leaders",
  "description": "Group of active barangay officials"
}
```

### Get All Groups
**GET /api/groups**

### Get a Single Group
**GET /api/groups/{id}**

### Update a Group
**PUT /api/groups/{id}**
```json
{
  "name": "barangay-leaders",
  "description": "Updated group description"
}
```

### Delete a Group
**DELETE /api/groups/{id}**

---

## 4. Contact Management (Within Groups)

### Add Contact to a Group
**POST /api/groups/{id}/contacts**
```json
{
  "mobile": "+639171234567"
}
```

### Update Contact in a Group
**PUT /api/groups/{group_id}/contacts/{contact_id}**
```json
{
  "mobile": "+639189876543"
}
```

### Delete Contact from a Group
**DELETE /api/groups/{group_id}/contacts/{contact_id}**

### List Contacts in a Group
**GET /api/groups/{id}/contacts**

---

## Notes

- All send operations allow `sender_id` override.
- Input validation is enforced (e.g., mobile number format, group existence).
- Both comma-delimited strings and arrays are accepted for bulk operations.
- Mobile numbers should be normalized (e.g., in `+63` format) before storage.
