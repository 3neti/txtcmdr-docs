# Blacklist / No-Send List

This document describes the blacklist feature that prevents SMS from being sent to specific phone numbers.

---

## Table of Contents

- [Overview](#overview)
- [Database Schema](#database-schema)
- [Model](#model)
- [Job Middleware](#job-middleware)
- [Actions](#actions)
- [API Endpoints](#api-endpoints)
- [Frontend Components](#frontend-components)
- [Usage Examples](#usage-examples)

---

## Overview

The blacklist feature uses **Job Middleware** to intercept SMS jobs before execution and check if the recipient's phone number is blacklisted.

### Architecture Decision

**Why Job Middleware?**
- ✅ Centralized checking - Single point of control
- ✅ Early termination - Jobs never execute for blacklisted numbers
- ✅ Consistent enforcement - Applied to all SMS jobs automatically
- ✅ Auditable - Centralized logging of blocked sends
- ✅ Clean separation - Business logic remains unchanged

### Flow

```
SendSMSJob dispatched
    ↓
CheckBlacklist middleware
    ↓
Is number blacklisted?
    ↓
Yes → Skip job, log event
    ↓
No → Execute job
```

---

## Database Schema

### Migration

**File:** `database/migrations/2024_01_20_000000_create_blacklisted_numbers_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('blacklisted_numbers', function (Blueprint $table) {
            $table->id();
            $table->string('mobile')->unique(); // E.164 format
            $table->string('reason')->nullable(); // Why blacklisted
            $table->string('added_by')->nullable(); // User who added
            $table->timestamp('blocked_at')->useCurrent();
            $table->timestamps();
            
            $table->index('mobile');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('blacklisted_numbers');
    }
};
```

### Table Structure

| Column | Type | Description |
|--------|------|-------------|
| `id` | bigint | Primary key |
| `mobile` | string | Phone number in E.164 format (+639171234567) |
| `reason` | string | Reason for blacklisting (opt-out, complaint, invalid) |
| `added_by` | string | User who added the number |
| `blocked_at` | timestamp | When the number was blacklisted |
| `created_at` | timestamp | Record creation time |
| `updated_at` | timestamp | Record update time |

---

## Model

**File:** `app/Models/BlacklistedNumber.php`

```php
<?php

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
        // Normalize to E.164 format
        try {
            $phone = new PhoneNumber($mobile, 'PH');
            $normalized = $phone->formatE164();
        } catch (\Exception $e) {
            return false; // Invalid numbers aren't blacklisted
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

## Job Middleware

**File:** `app/Jobs/Middleware/CheckBlacklist.php`

```php
<?php

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

### Alternative: Throw Exception (Optional)

If you prefer to track blocked attempts as "failed" jobs:

```php
public function handle(object $job, callable $next): void
{
    if (!property_exists($job, 'mobile')) {
        $next($job);
        return;
    }

    if (BlacklistedNumber::isBlacklisted($job->mobile)) {
        throw new \Exception("Number is blacklisted: {$job->mobile}");
    }

    $next($job);
}
```

---

## Applying Middleware to Jobs

### SendSMSJob (Updated)

**File:** `app/Jobs/SendSMSJob.php`

```php
<?php

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

            throw $e;
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

### BroadcastToGroupJob (No Changes Needed)

The middleware automatically applies to `SendSMSJob` dispatched from within `BroadcastToGroupJob`, so no changes needed!

### ProcessScheduledMessage (No Changes Needed)

Same - middleware applies to all `SendSMSJob` dispatches.

---

## Actions

### AddToBlacklist

**File:** `app/Actions/Blacklist/AddToBlacklist.php`

```php
<?php

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

**File:** `app/Actions/Blacklist/RemoveFromBlacklist.php`

```php
<?php

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

**File:** `app/Actions/Blacklist/ListBlacklistedNumbers.php`

```php
<?php

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

**File:** `app/Actions/Blacklist/CheckIfBlacklisted.php`

```php
<?php

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

## API Endpoints

**File:** `routes/api.php`

```php
use App\Actions\Blacklist\{
    AddToBlacklist,
    RemoveFromBlacklist,
    ListBlacklistedNumbers,
    CheckIfBlacklisted
};

// Blacklist Management
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/blacklist', ListBlacklistedNumbers::class);
    Route::post('/blacklist', AddToBlacklist::class);
    Route::delete('/blacklist', RemoveFromBlacklist::class);
    Route::post('/blacklist/check', CheckIfBlacklisted::class);
});
```

### API Examples

**Add to blacklist:**
```bash
POST /api/blacklist
Content-Type: application/json

{
  "mobile": "0917 123 4567",
  "reason": "User opted out"
}
```

**Remove from blacklist:**
```bash
DELETE /api/blacklist
Content-Type: application/json

{
  "mobile": "+639171234567"
}
```

**List blacklisted numbers:**
```bash
GET /api/blacklist?search=0917&reason=opt-out
```

**Check if blacklisted:**
```bash
POST /api/blacklist/check
Content-Type: application/json

{
  "mobile": "0917 123 4567"
}
```

---

## Frontend Components

### BlacklistManager.vue

**File:** `resources/js/pages/blacklist/index.vue`

```vue
<template>
  <app-layout>
    <div class="max-w-6xl mx-auto p-6">
      <div class="flex justify-between items-center mb-6">
        <h1 class="text-2xl font-bold text-gray-900">🚫 Blacklisted Numbers</h1>
        <button
          @click="showAddModal = true"
          class="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
        >
          Add Number
        </button>
      </div>

      <!-- Search & Filter -->
      <div class="mb-6 flex gap-4">
        <input
          v-model="filters.search"
          @input="search"
          type="text"
          placeholder="Search mobile number..."
          class="flex-1 px-4 py-2 border border-gray-300 rounded-lg"
        />
        <select
          v-model="filters.reason"
          @change="search"
          class="px-4 py-2 border border-gray-300 rounded-lg"
        >
          <option value="">All Reasons</option>
          <option value="opt-out">Opt-out</option>
          <option value="complaint">Complaint</option>
          <option value="invalid">Invalid</option>
          <option value="manual">Manual</option>
        </select>
      </div>

      <!-- Table -->
      <div class="bg-white rounded-lg shadow overflow-hidden">
        <table class="min-w-full divide-y divide-gray-200">
          <thead class="bg-gray-50">
            <tr>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                Mobile Number
              </th>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                Reason
              </th>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                Added By
              </th>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                Blocked At
              </th>
              <th class="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase">
                Actions
              </th>
            </tr>
          </thead>
          <tbody class="bg-white divide-y divide-gray-200">
            <tr v-for="item in blacklisted.data" :key="item.id">
              <td class="px-6 py-4 whitespace-nowrap font-mono text-sm">
                {{ item.mobile }}
              </td>
              <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                {{ item.reason }}
              </td>
              <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                {{ item.added_by || '—' }}
              </td>
              <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                {{ formatDate(item.blocked_at) }}
              </td>
              <td class="px-6 py-4 whitespace-nowrap text-right text-sm">
                <button
                  @click="removeNumber(item.mobile)"
                  class="text-red-600 hover:text-red-900"
                >
                  Remove
                </button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Pagination -->
      <div class="mt-4 flex justify-between items-center">
        <div class="text-sm text-gray-700">
          Showing {{ blacklisted.from }} to {{ blacklisted.to }} of {{ blacklisted.total }}
        </div>
        <div class="flex gap-2">
          <button
            v-for="link in blacklisted.links"
            :key="link.label"
            @click="goToPage(link.url)"
            :disabled="!link.url"
            :class="[
              'px-3 py-1 rounded',
              link.active ? 'bg-blue-600 text-white' : 'bg-white border hover:bg-gray-50',
              !link.url && 'opacity-50 cursor-not-allowed'
            ]"
            v-html="link.label"
          />
        </div>
      </div>
    </div>

    <!-- Add Modal -->
    <add-blacklist-modal
      :show="showAddModal"
      @close="showAddModal = false"
      @success="handleAdded"
    />
  </app-layout>
</template>

<script setup lang="ts">
import { ref, reactive } from 'vue'
import { router } from '@inertiajs/vue3'
import AppLayout from '@/layouts/app-layout.vue'
import AddBlacklistModal from '@/components/blacklist/add-blacklist-modal.vue'
import dayjs from 'dayjs'

interface Props {
  blacklisted: {
    data: Array<{
      id: number
      mobile: string
      reason: string
      added_by: string | null
      blocked_at: string
    }>
    from: number
    to: number
    total: number
    links: Array<{ label: string; url: string | null; active: boolean }>
  }
}

const props = defineProps<Props>()

const showAddModal = ref(false)

const filters = reactive({
  search: '',
  reason: '',
})

const search = () => {
  router.get('/blacklist', filters, { preserveState: true })
}

const removeNumber = async (mobile: string) => {
  if (!confirm(`Remove ${mobile} from blacklist?`)) return

  try {
    await axios.delete('/api/blacklist', { data: { mobile } })
    router.reload()
  } catch (error) {
    alert('Failed to remove number')
  }
}

const handleAdded = () => {
  showAddModal.value = false
  router.reload()
}

const goToPage = (url: string | null) => {
  if (url) router.visit(url)
}

const formatDate = (date: string) => {
  return dayjs(date).format('MMM D, YYYY h:mm A')
}
</script>
```

---

### AddBlacklistModal.vue

**File:** `resources/js/components/blacklist/add-blacklist-modal.vue`

```vue
<template>
  <TransitionRoot :show="show" as="template">
    <Dialog as="div" class="relative z-50" @close="close">
      <TransitionChild
        enter="ease-out duration-300"
        enter-from="opacity-0"
        enter-to="opacity-100"
        leave="ease-in duration-200"
        leave-from="opacity-100"
        leave-to="opacity-0"
      >
        <div class="fixed inset-0 bg-black bg-opacity-25" />
      </TransitionChild>

      <div class="fixed inset-0 overflow-y-auto">
        <div class="flex min-h-full items-center justify-center p-4">
          <TransitionChild
            enter="ease-out duration-300"
            enter-from="opacity-0 scale-95"
            enter-to="opacity-100 scale-100"
            leave="ease-in duration-200"
            leave-from="opacity-100 scale-100"
            leave-to="opacity-0 scale-95"
          >
            <DialogPanel class="w-full max-w-md bg-white rounded-lg p-6 shadow-xl">
              <DialogTitle class="text-lg font-bold mb-4">
                Add to Blacklist
              </DialogTitle>

              <form @submit.prevent="submit">
                <!-- Mobile Number -->
                <div class="mb-4">
                  <label class="block text-sm font-medium text-gray-700 mb-2">
                    Mobile Number
                  </label>
                  <input
                    v-model="form.mobile"
                    type="text"
                    required
                    placeholder="0917 123 4567"
                    class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                  />
                </div>

                <!-- Reason -->
                <div class="mb-6">
                  <label class="block text-sm font-medium text-gray-700 mb-2">
                    Reason
                  </label>
                  <select
                    v-model="form.reason"
                    class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                  >
                    <option value="opt-out">User opted out</option>
                    <option value="complaint">Complaint received</option>
                    <option value="invalid">Invalid number</option>
                    <option value="manual">Manual addition</option>
                  </select>
                </div>

                <!-- Actions -->
                <div class="flex gap-3">
                  <button
                    type="button"
                    @click="close"
                    class="flex-1 px-4 py-2 border border-gray-300 text-gray-700 rounded-lg hover:bg-gray-50"
                  >
                    Cancel
                  </button>
                  <button
                    type="submit"
                    :disabled="processing"
                    class="flex-1 px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700 disabled:opacity-50"
                  >
                    {{ processing ? 'Adding...' : 'Add to Blacklist' }}
                  </button>
                </div>
              </form>
            </DialogPanel>
          </TransitionChild>
        </div>
      </div>
    </Dialog>
  </TransitionRoot>
</template>

<script setup lang="ts">
import { ref, reactive } from 'vue'
import { Dialog, DialogPanel, DialogTitle, TransitionRoot, TransitionChild } from '@headlessui/vue'

interface Props {
  show: boolean
}

const props = defineProps<Props>()
const emit = defineEmits(['close', 'success'])

const processing = ref(false)

const form = reactive({
  mobile: '',
  reason: 'opt-out',
})

const submit = async () => {
  processing.value = true

  try {
    await axios.post('/api/blacklist', form)
    emit('success')
    form.mobile = ''
    form.reason = 'opt-out'
  } catch (error) {
    alert('Failed to add number to blacklist')
  } finally {
    processing.value = false
  }
}

const close = () => {
  emit('close')
}
</script>
```

---

## Usage Examples

### 1. Add Number to Blacklist

```php
use App\Actions\Blacklist\AddToBlacklist;

$action = new AddToBlacklist();
$action->handle(
    mobile: '0917 123 4567',
    reason: 'User opted out',
    addedBy: 'Admin'
);
```

### 2. Check Before Sending (Optional Manual Check)

```php
use App\Models\BlacklistedNumber;
use App\Jobs\SendSMSJob;

$mobile = '+639171234567';

if (!BlacklistedNumber::isBlacklisted($mobile)) {
    SendSMSJob::dispatch($mobile, 'Your message', 'TXTCMDR');
} else {
    Log::info('Skipped sending - number is blacklisted', ['mobile' => $mobile]);
}
```

### 3. Bulk Add from CSV

```php
use App\Models\BlacklistedNumber;
use League\Csv\Reader;

$csv = Reader::createFromPath('opt-outs.csv');
$csv->setHeaderOffset(0);

foreach ($csv as $row) {
    BlacklistedNumber::add(
        mobile: $row['mobile'],
        reason: 'CSV import - opt-out',
        addedBy: 'System'
    );
}
```

### 4. Remove Number

```php
use App\Models\BlacklistedNumber;

BlacklistedNumber::remove('0917 123 4567');
```

---

## Testing

### Test Blacklist Middleware

**File:** `tests/Feature/Blacklist/BlacklistMiddlewareTest.php`

```php
<?php

use App\Jobs\SendSMSJob;
use App\Models\BlacklistedNumber;
use Illuminate\Support\Facades\Queue;

beforeEach(function () {
    Queue::fake();
});

it('blocks SMS to blacklisted number', function () {
    $mobile = '+639171234567';
    
    // Add to blacklist
    BlacklistedNumber::add($mobile, 'Test');
    
    // Dispatch job
    SendSMSJob::dispatch($mobile, 'Test message', 'TXTCMDR');
    
    // Job should be deleted by middleware (not executed)
    Queue::assertNotPushed(SendSMSJob::class);
});

it('allows SMS to non-blacklisted number', function () {
    $mobile = '+639171234567';
    
    // Dispatch job (number not blacklisted)
    SendSMSJob::dispatch($mobile, 'Test message', 'TXTCMDR');
    
    // Job should execute normally
    Queue::assertPushed(SendSMSJob::class);
});
```

---

## Related Documentation

- [Backend Services](backend-services.md) - Jobs and Actions
- [Test Scaffolding](test-scaffolding.md) - Testing patterns
- [SMS Integration](sms-integration.md) - SMS sending details
