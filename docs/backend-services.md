# Backend Services

This document outlines the Laravel Actions, Controllers, and Services that power the Text Commander API.

> **Architecture Note:** Text Commander uses [lorisleiva/laravel-actions](https://laravelactions.com/) to combine controllers and business logic in single Action classes. Each Action can be invoked as a controller (`asController`), job (`asJob`), listener (`asListener`), or command (`asCommand`).

## Table of Contents

- [SMS Broadcasting Actions](#sms-broadcasting-actions)
  - [SendToMultipleRecipients](#sendtomultiplerecipients)
  - [SendToMultipleGroups](#sendtomultiplegroups)
- [Group Management Actions](#group-management-actions)
  - [CreateGroup](#creategroup)
  - [ListGroups](#listgroups)
  - [GetGroup](#getgroup)
  - [UpdateGroup](#updategroup)
  - [DeleteGroup](#deletegroup)
- [Contact Management Actions](#contact-management-actions)
  - [AddContactToGroup](#addcontacttogroup)
  - [UpdateContactInGroup](#updatecontactingroup)
  - [DeleteContactFromGroup](#deletecontactfromgroup)
  - [ListGroupContacts](#listgroupcontacts)
  - [ContactResource](#contactresource)
- [Blacklist Management Actions](#blacklist-management-actions)
  - [AddToBlacklist](#addtoblacklist)
  - [RemoveFromBlacklist](#removefromblacklist)
  - [ListBlacklistedNumbers](#listblacklistednumbers)
  - [CheckIfBlacklisted](#checkifblacklisted)
- [Models](#models)
  - [BlacklistedNumber](#blacklistednumber)
- [Supporting Services](#supporting-services)
  - [SMSService](#smsservice)
  - [ContactNormalizationService](#contactnormalizationservice)
- [Job Middleware](#job-middleware)
  - [CheckBlacklist](#checkblacklist)
- [Jobs](#jobs)
  - [SendSMSJob](#sendsmsjob)
  - [BroadcastToGroupJob](#broadcasttogroupjob)
  - [ProcessScheduledMessage](#processscheduledmessage)
  - [ContactImportJob](#contactimportjob)
- [Console Commands](#console-commands)
  - [ProcessScheduledMessages](#processscheduledmessages)
  - [SendScheduledBroadcasts](#sendscheduledbroadcasts)

---

## SMS Broadcasting Actions

### SendToMultipleRecipients

**Endpoint:** `POST /api/send`

**Location:** `app/Actions/SendToMultipleRecipients.php`

**Description:** Send SMS to one or more mobile numbers (comma-delimited or array).

**Request:**
```json
{
  "recipients": "+639171234567,+639189876543",
  "message": "Hello Quezon City!",
  "sender_id": "QUEZON_CITY"
}
```

**Implementation:**
```php
namespace App\Actions;

use App\Jobs\SendSMSJob;
use App\Models\Contact;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;
use Propaganistas\LaravelPhone\PhoneNumber;

class SendToMultipleRecipients
{
    use AsAction;

    public function handle(array|string $recipients, string $message, ?string $senderId = null): array
    {
        $senderId = $senderId ?? config('sms.default_sender_id', 'TXTCMDR');
        
        // Normalize to array
        $recipientArray = is_string($recipients) 
            ? explode(',', $recipients) 
            : $recipients;
        
        $normalizedRecipients = [];
        $dispatchedCount = 0;
        
        foreach ($recipientArray as $mobile) {
            try {
                // Create PhoneNumber object and Contact
                $phone = new PhoneNumber(trim($mobile), 'PH');
                $contact = Contact::fromPhoneNumber($phone);
                
                // Get E.164 format for SMS sending
                $e164Mobile = $contact->e164_mobile;
                
                SendSMSJob::dispatch($e164Mobile, $message, $senderId);
                
                $normalizedRecipients[] = $e164Mobile;
                $dispatchedCount++;
            } catch (\Exception $e) {
                // Skip invalid numbers
                continue;
            }
        }
        
        return [
            'status' => 'queued',
            'count' => $dispatchedCount,
            'recipients' => $normalizedRecipients,
            'invalid_count' => count($recipientArray) - count($normalizedRecipients),
        ];
    }

    public function rules(): array
    {
        return [
            'recipients' => 'required',
            'message' => 'required|string|max:1600',
            'sender_id' => 'nullable|string|max:11',
        ];
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $result = $this->handle(
            $request->recipients,
            $request->message,
            $request->sender_id ?? null
        );

        return response()->json($result, 200);
    }
}
```

**Routes:**
```php
// routes/api.php
use App\Actions\SendToMultipleRecipients;

Route::post('/send', SendToMultipleRecipients::class);
```

---

### SendToMultipleGroups

**Endpoint:** `POST /api/groups/send`

**Location:** `app/Actions/SendToMultipleGroups.php`

**Description:** Send SMS to one or more groups (comma-delimited or array).

**Request:**
```json
{
  "groups": "barangay-leaders,health-workers",
  "message": "Please attend the emergency meeting.",
  "sender_id": "QUEZON_CITY"
}
```

**Implementation:**
```php
namespace App\Actions;

use App\Jobs\BroadcastToGroupJob;
use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class SendToMultipleGroups
{
    use AsAction;

    public function handle(array|string $groups, string $message, ?string $senderId = null): array
    {
        $senderId = $senderId ?? config('sms.default_sender_id', 'TXTCMDR');
        
        // Normalize to array
        $groupArray = is_string($groups) 
            ? explode(',', $groups) 
            : $groups;
        
        // Trim and filter
        $groupArray = array_filter(
            array_map('trim', $groupArray)
        );
        
        $dispatchedGroups = [];
        $totalContacts = 0;
        
        foreach ($groupArray as $groupIdentifier) {
            $group = Group::where('name', $groupIdentifier)
                ->orWhere('id', $groupIdentifier)
                ->first();
            
            if ($group) {
                BroadcastToGroupJob::dispatch(
                    $group->id,
                    $message,
                    $senderId
                );
                
                $contactCount = $group->contacts()->count();
                $totalContacts += $contactCount;
                
                $dispatchedGroups[] = [
                    'id' => $group->id,
                    'name' => $group->name,
                    'contacts' => $contactCount,
                ];
            }
        }
        
        return [
            'status' => 'queued',
            'groups' => $dispatchedGroups,
            'total_contacts' => $totalContacts,
        ];
    }

    public function rules(): array
    {
        return [
            'groups' => 'required',
            'message' => 'required|string|max:1600',
            'sender_id' => 'nullable|string|max:11',
        ];
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $result = $this->handle(
            $request->groups,
            $request->message,
            $request->sender_id ?? null
        );

        return response()->json($result, 200);
    }
}
```

**Routes:**
```php
// routes/api.php
use App\Actions\SendToMultipleGroups;

Route::post('/groups/send', SendToMultipleGroups::class);
```

---

## Group Management Actions

### CreateGroup

**Endpoint:** `POST /api/groups`

**Location:** `app/Actions/Groups/CreateGroup.php`

**Request:**
```json
{
  "name": "barangay-leaders",
  "description": "Group of active barangay officials"
}
```

**Implementation:**
```php
namespace App\Actions\Groups;

use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class CreateGroup
{
    use AsAction;

    public function handle(string $name, ?string $description = null): Group
    {
        return Group::create([
            'name' => $name,
            'description' => $description,
        ]);
    }

    public function rules(): array
    {
        return [
            'name' => 'required|string|max:255|unique:groups,name',
            'description' => 'nullable|string|max:500',
        ];
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $group = $this->handle(
            $request->name,
            $request->description ?? null
        );

        return response()->json($group, 201);
    }
}
```

---

### ListGroups

**Endpoint:** `GET /api/groups`

**Location:** `app/Actions/Groups/ListGroups.php`

**Implementation:**
```php
namespace App\Actions\Groups;

use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class ListGroups
{
    use AsAction;

    public function handle()
    {
        return Group::withCount('contacts')
            ->orderBy('name')
            ->get();
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $groups = $this->handle();

        return response()->json($groups, 200);
    }
}
```

---

### GetGroup

**Endpoint:** `GET /api/groups/{id}`

**Location:** `app/Actions/Groups/GetGroup.php`

**Implementation:**
```php
namespace App\Actions\Groups;

use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\Concerns\AsAction;

class GetGroup
{
    use AsAction;

    public function handle(int $id): Group
    {
        return Group::with('contacts')
            ->withCount('contacts')
            ->findOrFail($id);
    }

    public function asController(int $id): JsonResponse
    {
        $group = $this->handle($id);

        return response()->json($group, 200);
    }
}
```

---

### UpdateGroup

**Endpoint:** `PUT /api/groups/{id}`

**Location:** `app/Actions/Groups/UpdateGroup.php`

**Request:**
```json
{
  "name": "barangay-leaders",
  "description": "Updated description"
}
```

**Implementation:**
```php
namespace App\Actions\Groups;

use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class UpdateGroup
{
    use AsAction;

    public function handle(int $id, array $data): Group
    {
        $group = Group::findOrFail($id);
        $group->update($data);

        return $group->fresh();
    }

    public function rules(): array
    {
        return [
            'name' => 'sometimes|string|max:255|unique:groups,name,' . $this->route('id'),
            'description' => 'nullable|string|max:500',
        ];
    }

    public function asController(ActionRequest $request, int $id): JsonResponse
    {
        $group = $this->handle($id, $request->validated());

        return response()->json($group, 200);
    }
}
```

---

### DeleteGroup

**Endpoint:** `DELETE /api/groups/{id}`

**Location:** `app/Actions/Groups/DeleteGroup.php`

**Implementation:**
```php
namespace App\Actions\Groups;

use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\Concerns\AsAction;

class DeleteGroup
{
    use AsAction;

    public function handle(int $id): bool
    {
        $group = Group::findOrFail($id);
        
        return $group->delete();
    }

    public function asController(int $id): JsonResponse
    {
        $this->handle($id);

        return response()->json([
            'message' => 'Group deleted successfully',
        ], 200);
    }
}
```

---

## Contact Management Actions

### AddContactToGroup

**Endpoint:** `POST /api/groups/{id}/contacts`

**Location:** `app/Actions/Contacts/AddContactToGroup.php`

**Request:**
```json
{
  "mobile": "0917 123 4567",
  "name": "Juan Dela Cruz",
  "tags": ["barangay-leader", "volunteer"]
}
```

**Implementation:**
```php
namespace App\Actions\Contacts;

use App\Models\Contact;
use App\Models\Group;
use Illuminate\Http\JsonResponse;
use LBHurtado\Contact\Data\ContactData;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;
use Propaganistas\LaravelPhone\PhoneNumber;

class AddContactToGroup
{
    use AsAction;

    public function handle(
        int $groupId, 
        string $mobile, 
        ?string $name = null,
        array $tags = []
    ): Contact {
        $group = Group::findOrFail($groupId);
        
        // Create PhoneNumber and Contact using package method
        $phone = new PhoneNumber($mobile, 'PH');
        $contact = Contact::fromPhoneNumber($phone);
        
        // Set name if provided
        if ($name) {
            $contact->setMeta('name', $name);
        }
        
        // Set tags if provided
        if (!empty($tags)) {
            $contact->setTags($tags);
        }
        
        $contact->save();
        
        // Attach to group if not already attached
        if (!$group->contacts->contains($contact)) {
            $group->contacts()->attach($contact->id);
        }
        
        return $contact;
    }

    public function rules(): array
    {
        return [
            'mobile' => 'required|phone:PH',
            'name' => 'nullable|string|max:255',
            'tags' => 'nullable|array',
            'tags.*' => 'string',
        ];
    }

    public function asController(ActionRequest $request, int $id): JsonResponse
    {
        $contact = $this->handle(
            $id,
            $request->mobile,
            $request->name ?? null,
            $request->tags ?? []
        );

        return response()->json(
            ContactData::fromModel($contact),
            201
        );
    }
}
```

---

### UpdateContactInGroup

**Endpoint:** `PUT /api/groups/{group_id}/contacts/{contact_id}`

**Location:** `app/Actions/Contacts/UpdateContactInGroup.php`

**Request:**
```json
{
  "mobile": "0918 765 4321",
  "name": "Juan Dela Cruz",
  "tags": ["barangay-leader", "health-worker"]
}
```

**Implementation:**
```php
namespace App\Actions\Contacts;

use App\Models\Contact;
use App\Models\Group;
use Illuminate\Http\JsonResponse;
use LBHurtado\Contact\Data\ContactData;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;
use Propaganistas\LaravelPhone\PhoneNumber;

class UpdateContactInGroup
{
    use AsAction;

    public function handle(
        int $groupId, 
        int $contactId, 
        ?string $mobile = null,
        ?string $name = null,
        ?array $tags = null
    ): Contact {
        $group = Group::findOrFail($groupId);
        $contact = Contact::findOrFail($contactId);
        
        // Verify contact belongs to group
        if (!$group->contacts->contains($contact)) {
            abort(404, 'Contact not found in this group');
        }
        
        // Update mobile if provided
        if ($mobile) {
            $phone = new PhoneNumber($mobile, 'PH');
            $contact->mobile = $phone->formatForMobileDialingInCountry('PH');
            $contact->country = 'PH';
        }
        
        // Update name if provided
        if ($name !== null) {
            $contact->setMeta('name', $name);
        }
        
        // Update tags if provided
        if ($tags !== null) {
            $contact->setTags($tags);
        }
        
        $contact->save();

        return $contact->fresh();
    }

    public function rules(): array
    {
        return [
            'mobile' => 'sometimes|phone:PH',
            'name' => 'nullable|string|max:255',
            'tags' => 'nullable|array',
            'tags.*' => 'string',
        ];
    }

    public function asController(ActionRequest $request, int $groupId, int $contactId): JsonResponse
    {
        $contact = $this->handle(
            $groupId,
            $contactId,
            $request->mobile ?? null,
            $request->name ?? null,
            $request->tags ?? null
        );

        return response()->json(
            ContactData::fromModel($contact),
            200
        );
    }
}
```

---

### DeleteContactFromGroup

**Endpoint:** `DELETE /api/groups/{group_id}/contacts/{contact_id}`

**Location:** `app/Actions/Contacts/DeleteContactFromGroup.php`

**Implementation:**
```php
namespace App\Actions\Contacts;

use App\Models\Contact;
use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\Concerns\AsAction;

class DeleteContactFromGroup
{
    use AsAction;

    public function handle(int $groupId, int $contactId): bool
    {
        $group = Group::findOrFail($groupId);
        $contact = Contact::findOrFail($contactId);
        
        // Detach from group
        $group->contacts()->detach($contact->id);
        
        // Optionally delete contact if not in any other groups
        if ($contact->groups()->count() === 0) {
            $contact->delete();
        }
        
        return true;
    }

    public function asController(int $groupId, int $contactId): JsonResponse
    {
        $this->handle($groupId, $contactId);

        return response()->json([
            'message' => 'Contact removed from group',
        ], 200);
    }
}
```

---

### ListGroupContacts

**Endpoint:** `GET /api/groups/{id}/contacts`

**Location:** `app/Actions/Contacts/ListGroupContacts.php`

**Implementation:**
```php
namespace App\Actions\Contacts;

use App\Http\Resources\ContactResource;
use App\Models\Group;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;
use Lorisleiva\Actions\Concerns\AsAction;

class ListGroupContacts
{
    use AsAction;

    public function handle(int $groupId)
    {
        $group = Group::findOrFail($groupId);
        
        return $group->contacts()
            ->orderBy('mobile')
            ->get();
    }

    public function asController(int $id): AnonymousResourceCollection
    {
        $contacts = $this->handle($id);

        return ContactResource::collection($contacts);
    }
}
```

---

### ContactResource

**Location:** `app/Http/Resources/ContactResource.php`

**Description:** API resource for Contact model using lbhurtado/contact package features.

**Implementation:**
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
            'mobile_e164' => $this->e164_mobile,
            'country' => $this->country,
            'name' => $this->name,
            'bank_account' => $this->bank_account,
            'bank_code' => $this->bank_code,
            'account_number' => $this->account_number,
            'tags' => $this->getTags(),
            'extra_attributes' => $this->extra_attributes->all(),
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }
}
```

**Example Response:**
```json
{
  "id": 1,
  "mobile": "09171234567",
  "mobile_e164": "+639171234567",
  "country": "PH",
  "name": "Juan Dela Cruz",
  "bank_account": "GXCHPHM2XXX:09171234567",
  "bank_code": "GXCHPHM2XXX",
  "account_number": "09171234567",
  "tags": ["barangay-leader", "volunteer"],
  "extra_attributes": {
    "address": "Quezon City",
    "notes": "Active volunteer"
  },
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

> **Note:** The Contact model extends `LBHurtado\Contact\Models\Contact`. See [Contact Package](contact-package.md) for complete documentation on package features including phone normalization, schemaless attributes, and bank account management.

---

## Blacklist Management Actions

### AddToBlacklist

**Endpoint:** `POST /api/blacklist`

**Location:** `app/Actions/Blacklist/AddToBlacklist.php`

**Description:** Add a phone number to the blacklist/no-send list.

**Request:**
```json
{
  "mobile": "0917 123 4567",
  "reason": "User opted out"
}
```

**Implementation:**
```php
namespace App\Actions\Blacklist;

use App\Models\BlacklistedNumber;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class AddToBlacklist
{
    use AsAction;

    public function handle(
        string $mobile,
        ?string $reason = null,
        ?string $addedBy = null
    ): BlacklistedNumber {
        return BlacklistedNumber::add($mobile, $reason, $addedBy);
    }

    public function rules(): array
    {
        return [
            'mobile' => 'required|phone:PH',
            'reason' => 'nullable|string|max:255',
        ];
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $blacklisted = $this->handle(
            $request->mobile,
            $request->reason ?? 'Manual addition',
            auth()->user()?->name
        );

        return response()->json([
            'message' => 'Number added to blacklist',
            'blacklisted' => $blacklisted,
        ], 201);
    }
}
```

---

### RemoveFromBlacklist

**Endpoint:** `DELETE /api/blacklist`

**Location:** `app/Actions/Blacklist/RemoveFromBlacklist.php`

**Implementation:**
```php
namespace App\Actions\Blacklist;

use App\Models\BlacklistedNumber;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class RemoveFromBlacklist
{
    use AsAction;

    public function handle(string $mobile): bool
    {
        return BlacklistedNumber::remove($mobile);
    }

    public function rules(): array
    {
        return [
            'mobile' => 'required|phone:PH',
        ];
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $removed = $this->handle($request->mobile);

        if (!$removed) {
            return response()->json([
                'message' => 'Number not found in blacklist',
            ], 404);
        }

        return response()->json([
            'message' => 'Number removed from blacklist',
        ], 200);
    }
}
```

---

### ListBlacklistedNumbers

**Endpoint:** `GET /api/blacklist`

**Location:** `app/Actions/Blacklist/ListBlacklistedNumbers.php`

**Implementation:**
```php
namespace App\Actions\Blacklist;

use App\Models\BlacklistedNumber;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class ListBlacklistedNumbers
{
    use AsAction;

    public function handle(?string $search = null, ?string $reason = null)
    {
        $query = BlacklistedNumber::query();

        if ($search) {
            $query->where('mobile', 'like', "%{$search}%");
        }

        if ($reason) {
            $query->byReason($reason);
        }

        return $query->orderBy('blocked_at', 'desc')->paginate(50);
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $blacklisted = $this->handle(
            $request->search,
            $request->reason
        );

        return response()->json($blacklisted);
    }
}
```

---

### CheckIfBlacklisted

**Endpoint:** `POST /api/blacklist/check`

**Location:** `app/Actions/Blacklist/CheckIfBlacklisted.php`

**Implementation:**
```php
namespace App\Actions\Blacklist;

use App\Models\BlacklistedNumber;
use Illuminate\Http\JsonResponse;
use Lorisleiva\Actions\ActionRequest;
use Lorisleiva\Actions\Concerns\AsAction;

class CheckIfBlacklisted
{
    use AsAction;

    public function handle(string $mobile): array
    {
        $isBlacklisted = BlacklistedNumber::isBlacklisted($mobile);
        
        $record = null;
        if ($isBlacklisted) {
            $record = BlacklistedNumber::where('mobile', $mobile)->first();
        }

        return [
            'is_blacklisted' => $isBlacklisted,
            'mobile' => $mobile,
            'record' => $record,
        ];
    }

    public function rules(): array
    {
        return [
            'mobile' => 'required|phone:PH',
        ];
    }

    public function asController(ActionRequest $request): JsonResponse
    {
        $result = $this->handle($request->mobile);

        return response()->json($result);
    }
}
```

---

### OptOut (Public Endpoint)

**Endpoint:** `POST /api/optout`

**Location:** `app/Actions/Blacklist/OptOut.php`

**Description:** Public endpoint allowing recipients to opt-out of SMS broadcasts. No authentication required.

**Implementation:**
```php
namespace App\Actions\Blacklist;

use App\Models\BlacklistedNumber;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Lorisleiva\Actions\Concerns\AsAction;

class OptOut
{
    use AsAction;

    public function handle(string $mobile): BlacklistedNumber
    {
        return BlacklistedNumber::add(
            mobile: $mobile,
            reason: 'opt-out',
            addedBy: 'Self-service opt-out'
        );
    }

    public function asController(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'mobile' => 'required|phone:PH',
        ]);

        $this->handle($validated['mobile']);

        return response()->json([
            'message' => 'You have been successfully opted out of SMS broadcasts',
            'mobile' => $validated['mobile'],
        ], 200);
    }
}
```

**Usage in SMS Messages:**
```
Your message content here. Reply STOP or visit https://txtcmdr.com/optout to unsubscribe.
```

**Note:** This action does NOT use `ActionRequest` since it must be publicly accessible without authentication.

---

## Models

### BlacklistedNumber

**Location:** `app/Models/BlacklistedNumber.php`

**Description:** Model for managing the blacklist/no-send list.

**Database Schema:**
```php
Schema::create('blacklisted_numbers', function (Blueprint $table) {
    $table->id();
    $table->string('mobile')->unique(); // E.164 format
    $table->string('reason')->nullable();
    $table->string('added_by')->nullable();
    $table->timestamp('blocked_at')->useCurrent();
    $table->timestamps();
    
    $table->index('mobile');
});
```

**Implementation:**
```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Propaganistas\LaravelPhone\PhoneNumber;

class BlacklistedNumber extends Model
{
    protected $fillable = [
        'mobile',
        'reason',
        'added_by',
        'blocked_at',
    ];

    protected $casts = [
        'blocked_at' => 'datetime',
    ];

    /**
     * Check if a phone number is blacklisted
     */
    public static function isBlacklisted(string $mobile): bool
    {
        try {
            $phone = new PhoneNumber($mobile, 'PH');
            $normalized = $phone->formatE164();
        } catch (\Exception $e) {
            return false;
        }

        return self::where('mobile', $normalized)->exists();
    }

    /**
     * Add a number to blacklist
     */
    public static function add(
        string $mobile, 
        ?string $reason = null, 
        ?string $addedBy = null
    ): self {
        $phone = new PhoneNumber($mobile, 'PH');
        $normalized = $phone->formatE164();

        return self::firstOrCreate(
            ['mobile' => $normalized],
            [
                'reason' => $reason ?? 'Manual addition',
                'added_by' => $addedBy,
                'blocked_at' => now(),
            ]
        );
    }

    /**
     * Remove a number from blacklist
     */
    public static function remove(string $mobile): bool
    {
        try {
            $phone = new PhoneNumber($mobile, 'PH');
            $normalized = $phone->formatE164();
            
            return self::where('mobile', $normalized)->delete() > 0;
        } catch (\Exception $e) {
            return false;
        }
    }

    /**
     * Scope: Recent blacklisted numbers
     */
    public function scopeRecent($query, int $days = 30)
    {
        return $query->where('blocked_at', '>=', now()->subDays($days));
    }

    /**
     * Scope: By reason
     */
    public function scopeByReason($query, string $reason)
    {
        return $query->where('reason', $reason);
    }
}
```

---

## Supporting Services

### SMSService

**Location:** `app/Services/SMSService.php`

**Description:** Wrapper service for SMS facade with logging and error handling.

**Implementation:**
```php
namespace App\Services;

use LBHurtado\SMS\Facades\SMS;
use Illuminate\Support\Facades\Log;

class SMSService
{
    public function send(string $mobile, string $message, string $senderId): bool
    {
        try {
            SMS::channel('engagespark')
                ->from($senderId)
                ->to($mobile)
                ->content($message)
                ->send();
            
            Log::info('SMS sent', [
                'mobile' => $mobile,
                'sender_id' => $senderId,
            ]);
            
            return true;
        } catch (\Exception $e) {
            Log::error('SMS send failed', [
                'mobile' => $mobile,
                'error' => $e->getMessage(),
            ]);
            
            return false;
        }
    }
    
    public function sendWithTopup(string $mobile, string $message, string $senderId, int $amount): bool
    {
        try {
            SMS::channel('engagespark')
                ->from($senderId)
                ->to($mobile)
                ->content($message)
                ->send()
                ->topup($amount);
            
            Log::info('SMS with topup sent', [
                'mobile' => $mobile,
                'sender_id' => $senderId,
                'topup_amount' => $amount,
            ]);
            
            return true;
        } catch (\Exception $e) {
            Log::error('SMS with topup failed', [
                'mobile' => $mobile,
                'error' => $e->getMessage(),
            ]);
            
            return false;
        }
    }
}
```

---

### ContactNormalizationService

**Location:** `app/Services/ContactNormalizationService.php`

**Description:** Normalize phone numbers to +63 format.

**Implementation:**
```php
namespace App\Services;

class ContactNormalizationService
{
    public function normalize(string $mobile): string
    {
        // Remove all non-numeric characters
        $mobile = preg_replace('/[^0-9]/', '', $mobile);
        
        // Handle different formats
        if (str_starts_with($mobile, '63')) {
            return '+' . $mobile;
        }
        
        if (str_starts_with($mobile, '0')) {
            return '+63' . substr($mobile, 1);
        }
        
        // Assume 10-digit local number
        if (strlen($mobile) === 10) {
            return '+63' . $mobile;
        }
        
        return '+' . $mobile;
    }
    
    public function validate(string $mobile): bool
    {
        $normalized = $this->normalize($mobile);
        
        // Must be +63 followed by 10 digits
        return preg_match('/^\+63[0-9]{10}$/', $normalized) === 1;
    }
}
```

---

## Job Middleware

### CheckBlacklist

**Location:** `app/Jobs/Middleware/CheckBlacklist.php`

**Description:** Job middleware that intercepts SMS jobs and checks if the recipient's phone number is blacklisted.

**Architecture:** Applied to `SendSMSJob` via the `middleware()` method. Automatically cascades to all jobs that dispatch `SendSMSJob`.

**Implementation:**
```php
namespace App\Jobs\Middleware;

use App\Models\BlacklistedNumber;
use Illuminate\Support\Facades\Log;

class CheckBlacklist
{
    /**
     * Process the queued job
     */
    public function handle(object $job, callable $next): void
    {
        // Only check jobs that have a 'mobile' property
        if (!property_exists($job, 'mobile')) {
            $next($job);
            return;
        }

        $mobile = $job->mobile;

        // Check if number is blacklisted
        if (BlacklistedNumber::isBlacklisted($mobile)) {
            Log::warning('SMS blocked - number is blacklisted', [
                'mobile' => $mobile,
                'job' => get_class($job),
                'message' => property_exists($job, 'message') ? substr($job->message, 0, 50) : null,
            ]);

            // Mark job as handled (don't retry)
            $job->delete();
            return;
        }

        // Number is not blacklisted, proceed with job
        $next($job);
    }
}
```

**How It Works:**
1. Job is dispatched (e.g., `SendSMSJob::dispatch($mobile, $message, $senderId)`)
2. Middleware intercepts before `handle()` executes
3. Checks if `$mobile` is in `blacklisted_numbers` table
4. If blacklisted: Deletes job, logs warning, returns
5. If not blacklisted: Proceeds to job's `handle()` method

---

## Jobs

### SendSMSJob

**Location:** `app/Jobs/SendSMSJob.php`

**Description:** Queued job for sending individual SMS messages.

**Queue:** `default`

**Retry:** 3 attempts

**Properties:**
- `$mobile` (string) - Recipient phone number in E.164 format
- `$message` (string) - SMS content
- `$senderId` (string) - Branded sender ID

**Implementation:**
```php
namespace App\Jobs;

use App\Jobs\Middleware\CheckBlacklist;
use App\Services\SMSService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class SendSMSJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 30;

    public function __construct(
        public string $mobile,
        public string $message,
        public string $senderId
    ) {}

    /**
     * Get the middleware the job should pass through
     */
    public function middleware(): array
    {
        return [new CheckBlacklist];
    }

    public function handle(SMSService $smsService): void
    {
        try {
            $smsService->send(
                $this->mobile,
                $this->message,
                $this->senderId
            );

            Log::info('SMS sent successfully', [
                'mobile' => $this->mobile,
                'sender_id' => $this->senderId,
            ]);
        } catch (\Exception $e) {
            Log::error('SMS send failed', [
                'mobile' => $this->mobile,
                'error' => $e->getMessage(),
            ]);

            throw $e; // Retry
        }
    }

    public function failed(\Throwable $exception): void
    {
        Log::error('SMS job failed after all retries', [
            'mobile' => $this->mobile,
            'exception' => $exception->getMessage(),
        ]);
    }
}
```

> **Note:** The `CheckBlacklist` middleware automatically applies to all `SendSMSJob` dispatches, including those from `BroadcastToGroupJob` and `ProcessScheduledMessage`.

**Usage:**
```php
use App\Jobs\SendSMSJob;

SendSMSJob::dispatch(
    '+639171234567',
    'Your message here',
    'TXTCMDR'
);

// Delay sending
SendSMSJob::dispatch($mobile, $message, $senderId)
    ->delay(now()->addMinutes(5));

// Set specific queue
SendSMSJob::dispatch($mobile, $message, $senderId)
    ->onQueue('high-priority');
```

---

### BroadcastToGroupJob

**Location:** `app/Jobs/BroadcastToGroupJob.php`

**Description:** Queued job for broadcasting SMS to all contacts in a group.

**Queue:** `broadcasts`

**Retry:** 2 attempts

**Properties:**
- `$groupId` (int) - Group ID
- `$message` (string) - SMS content
- `$senderId` (string) - Branded sender ID

**Implementation:**
```php
namespace App\Jobs;

use App\Models\Group;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;

class BroadcastToGroupJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 2;
    public int $timeout = 300; // 5 minutes for large groups

    public function __construct(
        public int $groupId,
        public string $message,
        public string $senderId
    ) {}

    public function handle(): void
    {
        $group = Group::with('contacts')->findOrFail($this->groupId);

        $contactCount = $group->contacts->count();

        Log::info('Starting group broadcast', [
            'group_id' => $this->groupId,
            'group_name' => $group->name,
            'contact_count' => $contactCount,
        ]);

        foreach ($group->contacts as $contact) {
            SendSMSJob::dispatch(
                $contact->e164_mobile,
                $this->message,
                $this->senderId
            );
        }

        Log::info('Group broadcast dispatched', [
            'group_id' => $this->groupId,
            'messages_dispatched' => $contactCount,
        ]);
    }

    public function failed(\Throwable $exception): void
    {
        Log::error('Broadcast job failed', [
            'group_id' => $this->groupId,
            'exception' => $exception->getMessage(),
        ]);
    }
}
```

**Usage:**
```php
use App\Jobs\BroadcastToGroupJob;

BroadcastToGroupJob::dispatch(
    groupId: 1,
    message: 'Emergency alert',
    senderId: 'QUEZON_CITY'
);

// Dispatch to specific queue
BroadcastToGroupJob::dispatch($groupId, $message, $senderId)
    ->onQueue('broadcasts');
```

---

### ProcessScheduledMessage

**Location:** `app/Jobs/ProcessScheduledMessage.php`

**Description:** Processes a single scheduled message by dispatching SMS jobs to all recipients.

**Queue:** `scheduled`

**Retry:** 1 attempt (non-retryable to prevent duplicate sends)

**Properties:**
- `$scheduledMessageId` (int) - ScheduledMessage ID

**Implementation:**
```php
namespace App\Jobs;

use App\Models\Group;
use App\Models\ScheduledMessage;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Propaganistas\LaravelPhone\PhoneNumber;

class ProcessScheduledMessage implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 1; // Don't retry to prevent duplicates
    public int $timeout = 300;

    public function __construct(
        public int $scheduledMessageId
    ) {}

    public function handle(): void
    {
        $message = ScheduledMessage::findOrFail($this->scheduledMessageId);

        // Verify message is still pending
        if ($message->status !== 'pending') {
            Log::warning('Attempted to process non-pending message', [
                'id' => $message->id,
                'status' => $message->status,
            ]);
            return;
        }

        // Update status to processing
        $message->update(['status' => 'processing']);

        DB::beginTransaction();

        try {
            $recipients = $this->getRecipients($message);
            $dispatchedCount = 0;
            $failedCount = 0;
            $errors = [];

            foreach ($recipients as $recipient) {
                try {
                    SendSMSJob::dispatch(
                        $recipient,
                        $message->message,
                        $message->sender_id
                    );
                    $dispatchedCount++;
                } catch (\Exception $e) {
                    $failedCount++;
                    $errors[] = [
                        'number' => $recipient,
                        'error' => $e->getMessage(),
                    ];
                }
            }

            // Update message status
            $message->update([
                'status' => $failedCount > 0 ? 'partially_sent' : 'sent',
                'sent_at' => now(),
                'sent_count' => $dispatchedCount,
                'failed_count' => $failedCount,
                'errors' => $failedCount > 0 ? $errors : null,
            ]);

            DB::commit();

            Log::info('Scheduled message processed', [
                'id' => $message->id,
                'dispatched' => $dispatchedCount,
                'failed' => $failedCount,
            ]);
        } catch (\Exception $e) {
            DB::rollBack();

            $message->update([
                'status' => 'failed',
                'errors' => [['error' => $e->getMessage()]],
            ]);

            Log::error('Failed to process scheduled message', [
                'id' => $message->id,
                'exception' => $e->getMessage(),
            ]);

            throw $e;
        }
    }

    private function getRecipients(ScheduledMessage $message): array
    {
        $recipients = [];
        $recipientData = $message->recipient_data;

        // Direct numbers
        if (!empty($recipientData['numbers'])) {
            foreach ($recipientData['numbers'] as $number) {
                try {
                    $phone = new PhoneNumber($number, 'PH');
                    $recipients[] = $phone->formatE164();
                } catch (\Exception $e) {
                    Log::warning('Invalid phone number in scheduled message', [
                        'number' => $number,
                        'message_id' => $message->id,
                    ]);
                }
            }
        }

        // Groups
        if (!empty($recipientData['groups'])) {
            foreach ($recipientData['groups'] as $groupData) {
                $group = Group::with('contacts')->find($groupData['id']);
                if ($group) {
                    foreach ($group->contacts as $contact) {
                        $recipients[] = $contact->e164_mobile;
                    }
                }
            }
        }

        return array_unique($recipients);
    }
}
```

**Usage:**
```php
use App\Jobs\ProcessScheduledMessage;

ProcessScheduledMessage::dispatch($scheduledMessageId);
```

---

### ContactImportJob

**Location:** `app/Jobs/ContactImportJob.php`

**Description:** Parses and imports contacts from CSV/Excel files into a group.

**Queue:** `imports`

**Retry:** 1 attempt

**Supported Formats:**
- CSV (`,` or `;` delimiter)
- Excel (.xlsx, .xls)

**Expected Columns:**
- `mobile` (required) - Phone number
- `name` (optional) - Contact name
- `tags` (optional) - Comma-separated tags

**Implementation:**
```php
namespace App\Jobs;

use App\Models\Contact;
use App\Models\Group;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Storage;
use League\Csv\Reader;
use Propaganistas\LaravelPhone\PhoneNumber;

class ContactImportJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 1;
    public int $timeout = 600; // 10 minutes for large files

    public function __construct(
        public int $groupId,
        public string $filePath
    ) {}

    public function handle(): void
    {
        $group = Group::findOrFail($this->groupId);
        $fileContent = Storage::get($this->filePath);

        $csv = Reader::createFromString($fileContent);
        $csv->setHeaderOffset(0);

        $imported = 0;
        $failed = 0;
        $errors = [];

        foreach ($csv as $index => $row) {
            try {
                if (empty($row['mobile'])) {
                    $failed++;
                    $errors[] = "Row {$index}: Missing mobile number";
                    continue;
                }

                $phone = new PhoneNumber($row['mobile'], 'PH');
                $contact = Contact::fromPhoneNumber($phone);

                // Set name if provided
                if (!empty($row['name'])) {
                    $contact->setMeta('name', $row['name']);
                }

                // Set tags if provided
                if (!empty($row['tags'])) {
                    $tags = array_map('trim', explode(',', $row['tags']));
                    $contact->setTags($tags);
                }

                $contact->save();

                // Attach to group
                if (!$group->contacts->contains($contact)) {
                    $group->contacts()->attach($contact->id);
                }

                $imported++;
            } catch (\Exception $e) {
                $failed++;
                $errors[] = "Row {$index}: {$e->getMessage()}";
            }
        }

        // Clean up file
        Storage::delete($this->filePath);

        Log::info('Contact import completed', [
            'group_id' => $this->groupId,
            'imported' => $imported,
            'failed' => $failed,
        ]);

        if ($failed > 0) {
            Log::warning('Contact import had errors', [
                'errors' => $errors,
            ]);
        }
    }

    public function failed(\Throwable $exception): void
    {
        Log::error('Contact import job failed', [
            'group_id' => $this->groupId,
            'file_path' => $this->filePath,
            'exception' => $exception->getMessage(),
        ]);

        // Clean up file
        Storage::delete($this->filePath);
    }
}
```

**Usage:**
```php
use App\Jobs\ContactImportJob;

// Upload file first
$path = $request->file('csv')->store('imports');

// Dispatch import job
ContactImportJob::dispatch(
    groupId: 1,
    filePath: $path
)->onQueue('imports');
```

---

## Console Commands

### ProcessScheduledMessages

**Location:** `app/Console/Commands/ProcessScheduledMessages.php`

**Signature:** `messages:process-scheduled`

**Schedule:** Runs every minute

**Description:** Processes scheduled messages that are ready to be sent (scheduled_at <= now()).

**Logic:**
1. Query `scheduled_messages` table for ready messages (status = 'pending', scheduled_at <= now())
2. Dispatch `ProcessScheduledMessage` job for each
3. Job handles status updates and error handling

**Implementation:**
```php
namespace App\Console\Commands;

use App\Jobs\ProcessScheduledMessage;
use App\Models\ScheduledMessage;
use Illuminate\Console\Command;

class ProcessScheduledMessages extends Command
{
    protected $signature = 'messages:process-scheduled';
    protected $description = 'Process scheduled messages that are ready to be sent';

    public function handle(): int
    {
        $readyMessages = ScheduledMessage::ready()->get();

        if ($readyMessages->isEmpty()) {
            $this->info('No scheduled messages ready for processing.');
            return Command::SUCCESS;
        }

        $this->info("Found {$readyMessages->count()} scheduled messages ready for processing...");

        foreach ($readyMessages as $message) {
            ProcessScheduledMessage::dispatch($message->id);
            
            $this->line("→ Dispatched message #{$message->id} ({$message->total_recipients} recipients)");
        }

        $this->info("✓ Dispatched {$readyMessages->count()} scheduled messages");

        return Command::SUCCESS;
    }
}
```

**Register in `app/Console/Kernel.php`:**
```php
namespace App\Console;

use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Foundation\Console\Kernel as ConsoleKernel;

class Kernel extends ConsoleKernel
{
    protected function schedule(Schedule $schedule): void
    {
        $schedule->command('messages:process-scheduled')
            ->everyMinute()
            ->withoutOverlapping()
            ->runInBackground();
    }

    protected function commands(): void
    {
        $this->load(__DIR__.'/Commands');

        require base_path('routes/console.php');
    }
}
```

**Manual Execution:**
```bash
# Process scheduled messages immediately
php artisan messages:process-scheduled

# Run scheduler (includes this command)
php artisan schedule:work
```

---

### SendScheduledBroadcasts

**Location:** `app/Console/Commands/SendScheduledBroadcasts.php`

**Signature:** `broadcasts:send`

**Schedule:** Runs every minute

**Description:** Processes scheduled campaign broadcasts (alternative to ScheduledMessage for multi-group campaigns).

**Logic:**
1. Query campaigns with `scheduled_at <= now()` and `status = 'pending'`
2. Dispatch `BroadcastToGroupJob` for each group in campaign
3. Update campaign status to `sending`

**Implementation:**
```php
namespace App\Console\Commands;

use App\Jobs\BroadcastToGroupJob;
use App\Models\Campaign;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;

class SendScheduledBroadcasts extends Command
{
    protected $signature = 'broadcasts:send';
    protected $description = 'Send scheduled campaign broadcasts';

    public function handle(): int
    {
        $campaigns = Campaign::where('status', 'pending')
            ->where('scheduled_at', '<=', now())
            ->with('groups')
            ->get();

        if ($campaigns->isEmpty()) {
            $this->info('No scheduled broadcasts ready.');
            return Command::SUCCESS;
        }

        $this->info("Found {$campaigns->count()} scheduled campaigns...");

        DB::transaction(function () use ($campaigns) {
            foreach ($campaigns as $campaign) {
                $groupCount = $campaign->groups->count();
                
                foreach ($campaign->groups as $group) {
                    BroadcastToGroupJob::dispatch(
                        $group->id,
                        $campaign->message,
                        $campaign->sender_id ?? config('sms.default_sender_id')
                    );
                }
                
                $campaign->update([
                    'status' => 'sending',
                    'sent_at' => now(),
                ]);

                $this->line("→ Dispatched campaign #{$campaign->id} to {$groupCount} group(s)");
            }
        });

        $this->info("✓ Dispatched {$campaigns->count()} scheduled campaigns");

        return Command::SUCCESS;
    }
}
```

**Register in `app/Console/Kernel.php`:**
```php
protected function schedule(Schedule $schedule): void
{
    // Process scheduled messages
    $schedule->command('messages:process-scheduled')
        ->everyMinute()
        ->withoutOverlapping()
        ->runInBackground();

    // Process scheduled campaigns
    $schedule->command('broadcasts:send')
        ->everyMinute()
        ->withoutOverlapping()
        ->runInBackground();
}
```

**Manual Execution:**
```bash
# Send scheduled broadcasts immediately
php artisan broadcasts:send

# See all scheduled commands
php artisan schedule:list
```

---

## Routes Summary

```php
// routes/api.php

use App\Actions\SendToMultipleRecipients;
use App\Actions\SendToMultipleGroups;
use App\Actions\Groups\{CreateGroup, ListGroups, GetGroup, UpdateGroup, DeleteGroup};
use App\Actions\Contacts\{AddContactToGroup, UpdateContactInGroup, DeleteContactFromGroup, ListGroupContacts};

// SMS Broadcasting
Route::post('/send', SendToMultipleRecipients::class);
Route::post('/groups/send', SendToMultipleGroups::class);

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

// Blacklist Management
Route::get('/blacklist', ListBlacklistedNumbers::class);
Route::post('/blacklist', AddToBlacklist::class);
Route::delete('/blacklist', RemoveFromBlacklist::class);
Route::post('/blacklist/check', CheckIfBlacklisted::class);

// Public opt-out endpoint (no authentication required)
Route::post('/optout', OptOut::class);
```

---

## Related Documentation

- [API Documentation](api-documentation.md) - HTTP API endpoints
- [SMS Integration](sms-integration.md) - SMS packages and drivers
- [Package Architecture](package-architecture.md) - Detailed package structure
- [Development Plan](development-plan.md) - Implementation roadmap
