# Interfaces, DTOs, and Models

This document outlines the data structures used throughout Text Commander, including Laravel models, request DTOs, response schemas, and TypeScript interfaces.

## Table of Contents

- [Laravel Models](#laravel-models)
- [Request DTOs](#request-dtos)
- [Response Schemas](#response-schemas)
- [TypeScript Interfaces](#typescript-interfaces)
- [Validation Rules](#validation-rules)

---

## Laravel Models

### Group Model

**Location:** `app/Models/Group.php`

**Table:** `groups`

**Schema:**
```php
Schema::create('groups', function (Blueprint $table) {
    $table->id();
    $table->string('name')->unique();
    $table->text('description')->nullable();
    $table->timestamps();
    $table->softDeletes();
});
```

**Model:**
```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Group extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'name',
        'description',
    ];

    protected $casts = [
        'created_at' => 'datetime',
        'updated_at' => 'datetime',
        'deleted_at' => 'datetime',
    ];

    public function contacts(): BelongsToMany
    {
        return $this->belongsToMany(Contact::class, 'contact_group')
            ->withTimestamps();
    }
}
```

---

### Contact Model

**Location:** `app/Models/Contact.php`

**Package:** Uses `lbhurtado/contact` package

**Table:** `contacts` (provided by package migration)

**Schema:**
```php
// From lbhurtado/contact package migration
Schema::create('contacts', function (Blueprint $table) {
    $table->id();
    $table->string('mobile');  // formatForMobileDialingInCountry format
    $table->string('country');
    $table->string('bank_account')->nullable();
    $table->schemalessAttributes('extra_attributes'); // JSON column
    $table->timestamps();
});

// Application-specific pivot table
Schema::create('contact_group', function (Blueprint $table) {
    $table->id();
    $table->foreignId('contact_id')->constrained()->onDelete('cascade');
    $table->foreignId('group_id')->constrained()->onDelete('cascade');
    $table->timestamps();
    
    $table->unique(['contact_id', 'group_id']);
});
```

**Model:**
```php
namespace App\Models;

use LBHurtado\Contact\Models\Contact as BaseContact;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Contact extends BaseContact
{
    /**
     * Groups this contact belongs to
     */
    public function groups(): BelongsToMany
    {
        return $this->belongsToMany(Group::class, 'contact_group')
            ->withTimestamps();
    }

    /**
     * SMS logs for this contact
     */
    public function smsLogs()
    {
        return $this->hasMany(SMSLog::class, 'mobile', 'mobile');
    }

    /**
     * Get formatted mobile for SMS sending (E.164)
     */
    public function getE164MobileAttribute(): string
    {
        return phone($this->mobile, $this->country)->formatE164();
    }

    /**
     * Set tags helper
     */
    public function setTags(array $tags): self
    {
        $this->extra_attributes->set('tags', $tags);
        return $this;
    }

    /**
     * Get tags helper
     */
    public function getTags(): array
    {
        return $this->extra_attributes->get('tags', []);
    }

    /**
     * Check if contact has tag
     */
    public function hasTag(string $tag): bool
    {
        return in_array($tag, $this->getTags());
    }
}
```

> **Note:** The Contact model extends `LBHurtado\Contact\Models\Contact` from the `lbhurtado/contact` package. This provides automatic phone normalization, schemaless attributes, bank account management, and meta data storage. See [Contact Package](contact-package.md) for complete documentation.

---

### Campaign Model

**Location:** `app/Models/Campaign.php`

**Table:** `campaigns`

**Schema:**
```php
Schema::create('campaigns', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->text('message');
    $table->string('sender_id')->default('TXTCMDR');
    $table->enum('status', ['pending', 'sending', 'completed', 'failed'])->default('pending');
    $table->timestamp('scheduled_at')->nullable();
    $table->timestamps();
    $table->softDeletes();
});

Schema::create('campaign_group', function (Blueprint $table) {
    $table->id();
    $table->foreignId('campaign_id')->constrained()->onDelete('cascade');
    $table->foreignId('group_id')->constrained()->onDelete('cascade');
    $table->timestamps();
});
```

**Model:**
```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Campaign extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'name',
        'message',
        'sender_id',
        'status',
        'scheduled_at',
    ];

    protected $casts = [
        'scheduled_at' => 'datetime',
        'created_at' => 'datetime',
        'updated_at' => 'datetime',
        'deleted_at' => 'datetime',
    ];

    public function groups(): BelongsToMany
    {
        return $this->belongsToMany(Group::class, 'campaign_group')
            ->withTimestamps();
    }
}
```

---

### SMSLog Model

**Location:** `app/Models/SMSLog.php`

**Table:** `sms_logs`

**Schema:**
```php
Schema::create('sms_logs', function (Blueprint $table) {
    $table->id();
    $table->string('message_id')->unique()->nullable();
    $table->string('mobile');
    $table->text('message');
    $table->string('sender_id');
    $table->enum('status', ['queued', 'sent', 'delivered', 'failed'])->default('queued');
    $table->timestamp('sent_at')->nullable();
    $table->timestamp('delivered_at')->nullable();
    $table->text('error_message')->nullable();
    $table->timestamps();
});
```

**Model:**
```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class SMSLog extends Model
{
    protected $table = 'sms_logs';

    protected $fillable = [
        'message_id',
        'mobile',
        'message',
        'sender_id',
        'status',
        'sent_at',
        'delivered_at',
        'error_message',
    ];

    protected $casts = [
        'sent_at' => 'datetime',
        'delivered_at' => 'datetime',
        'created_at' => 'datetime',
        'updated_at' => 'datetime',
    ];
}
```

---

## Request DTOs

### SendToMultipleRecipientsRequest

**Endpoint:** `POST /api/send`

**JSON Schema:**
```json
{
  "recipients": "+639171234567,+639189876543",
  "message": "Hello Quezon City!",
  "sender_id": "QUEZON_CITY"
}
```

**TypeScript Interface:**
```typescript
interface SendToMultipleRecipientsRequest {
  recipients: string | string[]; // Comma-delimited or array
  message: string;
  sender_id?: string;
}
```

**Laravel Validation:**
```php
[
    'recipients' => 'required',
    'message' => 'required|string|max:1600',
    'sender_id' => 'nullable|string|max:11',
]
```

---

### SendToMultipleGroupsRequest

**Endpoint:** `POST /api/groups/send`

**JSON Schema:**
```json
{
  "groups": "barangay-leaders,health-workers",
  "message": "Please attend the emergency meeting.",
  "sender_id": "QUEZON_CITY"
}
```

**TypeScript Interface:**
```typescript
interface SendToMultipleGroupsRequest {
  groups: string | string[] | number[]; // Names, IDs, or comma-delimited
  message: string;
  sender_id?: string;
}
```

**Laravel Validation:**
```php
[
    'groups' => 'required',
    'message' => 'required|string|max:1600',
    'sender_id' => 'nullable|string|max:11',
]
```

---

### CreateGroupRequest

**Endpoint:** `POST /api/groups`

**JSON Schema:**
```json
{
  "name": "barangay-leaders",
  "description": "Group of active barangay officials"
}
```

**TypeScript Interface:**
```typescript
interface CreateGroupRequest {
  name: string;
  description?: string;
}
```

**Laravel Validation:**
```php
[
    'name' => 'required|string|max:255|unique:groups,name',
    'description' => 'nullable|string|max:500',
]
```

---

### UpdateGroupRequest

**Endpoint:** `PUT /api/groups/{id}`

**JSON Schema:**
```json
{
  "name": "barangay-leaders",
  "description": "Updated description"
}
```

**TypeScript Interface:**
```typescript
interface UpdateGroupRequest {
  name?: string;
  description?: string;
}
```

**Laravel Validation:**
```php
[
    'name' => 'sometimes|string|max:255|unique:groups,name,' . $groupId,
    'description' => 'nullable|string|max:500',
]
```

---

### AddContactToGroupRequest

**Endpoint:** `POST /api/groups/{id}/contacts`

**JSON Schema:**
```json
{
  "mobile": "+639171234567",
  "name": "Juan Dela Cruz"
}
```

**TypeScript Interface:**
```typescript
interface AddContactToGroupRequest {
  mobile: string; // E.164 format: +63XXXXXXXXXX
  name?: string;
}
```

**Laravel Validation:**
```php
[
    'mobile' => 'required|string|regex:/^\+63[0-9]{10}$/',
    'name' => 'nullable|string|max:255',
]
```

---

### UpdateContactRequest

**Endpoint:** `PUT /api/groups/{group_id}/contacts/{contact_id}`

**JSON Schema:**
```json
{
  "mobile": "+639189876543",
  "name": "Juan Dela Cruz"
}
```

**TypeScript Interface:**
```typescript
interface UpdateContactRequest {
  mobile?: string;
  name?: string;
}
```

**Laravel Validation:**
```php
[
    'mobile' => 'sometimes|string|regex:/^\+63[0-9]{10}$/',
    'name' => 'nullable|string|max:255',
]
```

---

## Response Schemas

### SendToMultipleRecipientsResponse

```json
{
  "status": "queued",
  "count": 2,
  "recipients": [
    "+639171234567",
    "+639189876543"
  ]
}
```

**TypeScript Interface:**
```typescript
interface SendToMultipleRecipientsResponse {
  status: 'queued';
  count: number;
  recipients: string[];
}
```

---

### SendToMultipleGroupsResponse

```json
{
  "status": "queued",
  "groups": [
    {
      "id": 1,
      "name": "barangay-leaders",
      "contacts": 50
    },
    {
      "id": 2,
      "name": "health-workers",
      "contacts": 30
    }
  ],
  "total_contacts": 80
}
```

**TypeScript Interface:**
```typescript
interface SendToMultipleGroupsResponse {
  status: 'queued';
  groups: Array<{
    id: number;
    name: string;
    contacts: number;
  }>;
  total_contacts: number;
}
```

---

### GroupResource

```json
{
  "id": 1,
  "name": "barangay-leaders",
  "description": "Group of active barangay officials",
  "contacts_count": 50,
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

**TypeScript Interface:**
```typescript
interface Group {
  id: number;
  name: string;
  description: string | null;
  contacts_count?: number;
  created_at: string;
  updated_at: string;
}
```

**Laravel Resource:**
```php
namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class GroupResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'description' => $this->description,
            'contacts_count' => $this->whenCounted('contacts'),
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }
}
```

---

### ContactResource

```json
{
  "id": 1,
  "mobile": "+639171234567",
  "name": "Juan Dela Cruz",
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

**TypeScript Interface:**
```typescript
interface Contact {
  id: number;
  mobile: string;
  name: string | null;
  created_at: string;
  updated_at: string;
}
```

**Laravel Resource:**
```php
namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class ContactResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'mobile' => $this->mobile,
            'name' => $this->name,
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }
}
```

---

### ErrorResponse

```json
{
  "message": "Validation failed",
  "errors": {
    "mobile": [
      "The mobile field is required."
    ]
  }
}
```

**TypeScript Interface:**
```typescript
interface ErrorResponse {
  message: string;
  errors?: Record<string, string[]>;
}
```

---

## TypeScript Interfaces

Complete TypeScript type definitions for frontend consumption.

```typescript
// types/api.ts

/** SMS Broadcasting */
export interface SendToMultipleRecipientsRequest {
  recipients: string | string[];
  message: string;
  sender_id?: string;
}

export interface SendToMultipleRecipientsResponse {
  status: 'queued';
  count: number;
  recipients: string[];
}

export interface SendToMultipleGroupsRequest {
  groups: string | string[] | number[];
  message: string;
  sender_id?: string;
}

export interface SendToMultipleGroupsResponse {
  status: 'queued';
  groups: Array<{
    id: number;
    name: string;
    contacts: number;
  }>;
  total_contacts: number;
}

/** Group Management */
export interface Group {
  id: number;
  name: string;
  description: string | null;
  contacts_count?: number;
  created_at: string;
  updated_at: string;
}

export interface CreateGroupRequest {
  name: string;
  description?: string;
}

export interface UpdateGroupRequest {
  name?: string;
  description?: string;
}

/** Contact Management */
export interface Contact {
  id: number;
  mobile: string;
  name: string | null;
  created_at: string;
  updated_at: string;
}

export interface AddContactToGroupRequest {
  mobile: string;
  name?: string;
}

export interface UpdateContactRequest {
  mobile?: string;
  name?: string;
}

/** Common */
export interface PaginatedResponse<T> {
  data: T[];
  links: {
    first: string;
    last: string;
    prev: string | null;
    next: string | null;
  };
  meta: {
    current_page: number;
    from: number;
    last_page: number;
    per_page: number;
    to: number;
    total: number;
  };
}

export interface ErrorResponse {
  message: string;
  errors?: Record<string, string[]>;
}
```

---

## Validation Rules

### Phone Number Validation

**Philippine Mobile Number Format:**
- Must start with `+63`
- Followed by 10 digits
- Example: `+639171234567`

**Regex:** `/^\+63[0-9]{10}$/`

**Laravel Custom Rule:**
```php
namespace App\Rules;

use Illuminate\Contracts\Validation\Rule;

class PhilippineMobileNumber implements Rule
{
    public function passes($attribute, $value)
    {
        return preg_match('/^\+63[0-9]{10}$/', $value) === 1;
    }

    public function message()
    {
        return 'The :attribute must be a valid Philippine mobile number (+63XXXXXXXXXX).';
    }
}
```

**Usage:**
```php
use App\Rules\PhilippineMobileNumber;

[
    'mobile' => ['required', new PhilippineMobileNumber],
]
```

---

### SMS Message Validation

**Rules:**
- Maximum 1600 characters (10 SMS segments)
- GSM 7-bit character set recommended
- Unicode characters count as 2 characters

**Laravel Validation:**
```php
[
    'message' => 'required|string|max:1600',
]
```

---

### Sender ID Validation

**Rules:**
- Maximum 11 characters
- Alphanumeric only
- Must be pre-approved by engageSPARK

**Laravel Validation:**
```php
[
    'sender_id' => 'nullable|string|max:11|alpha_dash',
]
```

---

## API Client Example (TypeScript)

```typescript
// services/api.ts
import axios, { AxiosInstance } from 'axios';
import type {
  SendToMultipleRecipientsRequest,
  SendToMultipleRecipientsResponse,
  SendToMultipleGroupsRequest,
  SendToMultipleGroupsResponse,
  Group,
  Contact,
} from '../types/api';

class TextCommanderAPI {
  private client: AxiosInstance;

  constructor(baseURL: string, apiToken?: string) {
    this.client = axios.create({
      baseURL,
      headers: {
        'Content-Type': 'application/json',
        ...(apiToken && { Authorization: `Bearer ${apiToken}` }),
      },
    });
  }

  // SMS Broadcasting
  async sendToRecipients(
    data: SendToMultipleRecipientsRequest
  ): Promise<SendToMultipleRecipientsResponse> {
    const response = await this.client.post('/send', data);
    return response.data;
  }

  async sendToGroups(
    data: SendToMultipleGroupsRequest
  ): Promise<SendToMultipleGroupsResponse> {
    const response = await this.client.post('/groups/send', data);
    return response.data;
  }

  // Group Management
  async getGroups(): Promise<Group[]> {
    const response = await this.client.get('/groups');
    return response.data;
  }

  async getGroup(id: number): Promise<Group> {
    const response = await this.client.get(`/groups/${id}`);
    return response.data;
  }

  async createGroup(data: { name: string; description?: string }): Promise<Group> {
    const response = await this.client.post('/groups', data);
    return response.data;
  }

  async updateGroup(id: number, data: Partial<Group>): Promise<Group> {
    const response = await this.client.put(`/groups/${id}`, data);
    return response.data;
  }

  async deleteGroup(id: number): Promise<void> {
    await this.client.delete(`/groups/${id}`);
  }

  // Contact Management
  async getGroupContacts(groupId: number): Promise<Contact[]> {
    const response = await this.client.get(`/groups/${groupId}/contacts`);
    return response.data;
  }

  async addContactToGroup(
    groupId: number,
    mobile: string,
    name?: string
  ): Promise<Contact> {
    const response = await this.client.post(`/groups/${groupId}/contacts`, {
      mobile,
      name,
    });
    return response.data;
  }

  async updateContact(
    groupId: number,
    contactId: number,
    data: Partial<Contact>
  ): Promise<Contact> {
    const response = await this.client.put(
      `/groups/${groupId}/contacts/${contactId}`,
      data
    );
    return response.data;
  }

  async deleteContact(groupId: number, contactId: number): Promise<void> {
    await this.client.delete(`/groups/${groupId}/contacts/${contactId}`);
  }
}

export default TextCommanderAPI;
```

---

## Related Documentation

- [API Documentation](api-documentation.md) - HTTP API endpoints
- [Backend Services](backend-services.md) - Actions and Controllers
- [SMS Integration](sms-integration.md) - SMS packages and drivers
- [Development Plan](development-plan.md) - Implementation roadmap
