# Test Scaffolding (Pest)

This document provides complete Pest test scaffolding for all layers of Text Commander, including models, factories, actions, controllers, services, DTOs, and seeders.

---

## Table of Contents

- [Pest Configuration](#pest-configuration)
- [Factories](#factories)
- [Model Tests](#model-tests)
- [Action Tests](#action-tests)
- [Controller Tests](#controller-tests)
- [Service Tests](#service-tests)
- [Seeder Tests](#seeder-tests)
- [Integration Tests](#integration-tests)

---

## Pest Configuration

### `tests/Pest.php`

```php
<?php

use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

uses(TestCase::class, RefreshDatabase::class)->in('Feature', 'Unit');

// Helper functions
function actingAsAdmin()
{
    return test()->actingAs(\App\Models\User::factory()->create([
        'role' => 'admin',
    ]));
}

function createContact(array $attributes = [])
{
    return \App\Models\Contact::factory()->create($attributes);
}

function createGroup(array $attributes = [])
{
    return \App\Models\Group::factory()->create($attributes);
}

function createScheduledMessage(array $attributes = [])
{
    return \App\Models\ScheduledMessage::factory()->create($attributes);
}

function createBlacklistedNumber(array $attributes = [])
{
    return \App\Models\BlacklistedNumber::factory()->create($attributes);
}
```

### `phpunit.xml` (or `pest.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true">
    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory>tests/Feature</directory>
        </testsuite>
    </testsuites>
    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="DB_CONNECTION" value="sqlite"/>
        <env name="DB_DATABASE" value=":memory:"/>
        <env name="CACHE_DRIVER" value="array"/>
        <env name="QUEUE_CONNECTION" value="sync"/>
        <env name="SESSION_DRIVER" value="array"/>
    </php>
</phpunit>
```

---

## Factories

### `database/factories/UserFactory.php`

```php
namespace Database\Factories;

use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;

class UserFactory extends Factory
{
    protected $model = User::class;

    public function definition(): array
    {
        return [
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'email_verified_at' => now(),
            'password' => Hash::make('password'),
            'role' => 'admin',
            'remember_token' => Str::random(10),
        ];
    }

    public function unverified(): static
    {
        return $this->state(fn (array $attributes) => [
            'email_verified_at' => null,
        ]);
    }
}
```

### `database/factories/ContactFactory.php`

```php
namespace Database\Factories;

use App\Models\Contact;
use Illuminate\Database\Eloquent\Factories\Factory;
use Propaganistas\LaravelPhone\PhoneNumber;

class ContactFactory extends Factory
{
    protected $model = Contact::class;

    public function definition(): array
    {
        $mobile = '0917' . fake()->numerify('#######');
        $phone = new PhoneNumber($mobile, 'PH');

        return [
            'mobile' => $phone->formatForMobileDialingInCountry('PH'),
            'country' => 'PH',
            'extra_attributes' => [
                'name' => fake()->name(),
                'address' => fake()->address(),
            ],
        ];
    }

    public function withTags(array $tags): static
    {
        return $this->afterCreating(function (Contact $contact) use ($tags) {
            $contact->setTags($tags);
            $contact->save();
        });
    }

    public function withoutName(): static
    {
        return $this->state(fn (array $attributes) => [
            'extra_attributes' => [],
        ]);
    }
}
```

### `database/factories/GroupFactory.php`

```php
namespace Database\Factories;

use App\Models\Group;
use Illuminate\Database\Eloquent\Factories\Factory;

class GroupFactory extends Factory
{
    protected $model = Group::class;

    public function definition(): array
    {
        return [
            'name' => fake()->words(2, true),
            'description' => fake()->sentence(),
        ];
    }

    public function withContacts(int $count = 5): static
    {
        return $this->afterCreating(function (Group $group) use ($count) {
            $contacts = \App\Models\Contact::factory()->count($count)->create();
            $group->contacts()->attach($contacts->pluck('id'));
        });
    }
}
```

### `database/factories/ScheduledMessageFactory.php`

```php
namespace Database\Factories;

use App\Models\ScheduledMessage;
use Illuminate\Database\Eloquent\Factories\Factory;

class ScheduledMessageFactory extends Factory
{
    protected $model = ScheduledMessage::class;

    public function definition(): array
    {
        return [
            'message' => fake()->sentence(),
            'sender_id' => 'TXTCMDR',
            'recipient_type' => 'numbers',
            'recipient_data' => [
                'numbers' => ['+639171234567', '+639189876543'],
                'groups' => [],
            ],
            'scheduled_at' => now()->addHour(),
            'status' => 'pending',
            'total_recipients' => 2,
            'sent_count' => 0,
            'failed_count' => 0,
        ];
    }

    public function pending(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => 'pending',
            'scheduled_at' => now()->addHour(),
        ]);
    }

    public function sent(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => 'sent',
            'sent_at' => now()->subHour(),
            'sent_count' => $attributes['total_recipients'],
        ]);
    }

    public function failed(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => 'failed',
            'sent_at' => now()->subHour(),
            'failed_count' => $attributes['total_recipients'],
            'errors' => [
                ['number' => '+639171234567', 'error' => 'Invalid number'],
            ],
        ]);
    }

    public function toGroup(): static
    {
        return $this->state(fn (array $attributes) => [
            'recipient_type' => 'group',
            'recipient_data' => [
                'numbers' => [],
                'groups' => [
                    ['id' => 1, 'name' => 'Test Group', 'count' => 10],
                ],
            ],
            'total_recipients' => 10,
        ]);
    }
}
```

### `database/factories/BlacklistedNumberFactory.php`

```php
namespace Database\Factories;

use App\Models\BlacklistedNumber;
use Illuminate\Database\Eloquent\Factories\Factory;
use Propaganistas\LaravelPhone\PhoneNumber;

class BlacklistedNumberFactory extends Factory
{
    protected $model = BlacklistedNumber::class;

    public function definition(): array
    {
        $mobile = '0917' . fake()->numerify('#######');
        $phone = new PhoneNumber($mobile, 'PH');

        return [
            'mobile' => $phone->formatE164(),
            'reason' => fake()->randomElement(['opt-out', 'complaint', 'invalid', 'manual']),
            'added_by' => fake()->name(),
            'blocked_at' => now(),
        ];
    }

    public function optOut(): static
    {
        return $this->state(fn (array $attributes) => [
            'reason' => 'opt-out',
        ]);
    }

    public function complaint(): static
    {
        return $this->state(fn (array $attributes) => [
            'reason' => 'complaint',
        ]);
    }

    public function invalid(): static
    {
        return $this->state(fn (array $attributes) => [
            'reason' => 'invalid',
        ]);
    }
}
```

---

### `database/factories/SenderIDFactory.php`

```php
namespace Database\Factories;

use App\Models\SenderID;
use Illuminate\Database\Eloquent\Factories\Factory;

class SenderIDFactory extends Factory
{
    protected $model = SenderID::class;

    public function definition(): array
    {
        return [
            'name' => strtoupper(fake()->word()),
            'is_default' => false,
        ];
    }

    public function default(): static
    {
        return $this->state(fn (array $attributes) => [
            'is_default' => true,
        ]);
    }
}
```

---

## Model Tests

### `tests/Unit/Models/ContactTest.php`

```php
use App\Models\Contact;
use Propaganistas\LaravelPhone\PhoneNumber;

it('creates a contact from phone number', function () {
    $phone = new PhoneNumber('0917 123 4567', 'PH');
    $contact = Contact::fromPhoneNumber($phone);

    expect($contact)->toBeInstanceOf(Contact::class)
        ->and($contact->mobile)->toBe('09171234567')
        ->and($contact->country)->toBe('PH');
});

it('returns e164 formatted mobile number', function () {
    $contact = Contact::factory()->create([
        'mobile' => '09171234567',
        'country' => 'PH',
    ]);

    expect($contact->e164_mobile)->toBe('+639171234567');
});

it('can set and get tags', function () {
    $contact = Contact::factory()->create();
    $contact->setTags(['leader', 'volunteer']);

    expect($contact->getTags())->toBe(['leader', 'volunteer'])
        ->and($contact->hasTag('leader'))->toBeTrue()
        ->and($contact->hasTag('unknown'))->toBeFalse();
});

it('can set and get meta data', function () {
    $contact = Contact::factory()->create();
    $contact->setMeta('name', 'Juan Dela Cruz');
    $contact->setMeta('address', 'Quezon City');

    expect($contact->getMeta('name'))->toBe('Juan Dela Cruz')
        ->and($contact->getMeta('address'))->toBe('Quezon City');
});

it('stores extra attributes as schemaless', function () {
    $contact = Contact::factory()->create([
        'extra_attributes' => [
            'name' => 'Test User',
            'notes' => 'Important contact',
        ],
    ]);

    expect($contact->extra_attributes->get('name'))->toBe('Test User')
        ->and($contact->extra_attributes->get('notes'))->toBe('Important contact');
});
```

### `tests/Unit/Models/GroupTest.php`

```php
use App\Models\Contact;
use App\Models\Group;

it('can attach contacts to a group', function () {
    $group = Group::factory()->create();
    $contacts = Contact::factory()->count(3)->create();

    $group->contacts()->attach($contacts->pluck('id'));

    expect($group->contacts)->toHaveCount(3)
        ->and($group->contacts_count)->toBe(3);
});

it('can detach contacts from a group', function () {
    $group = Group::factory()->withContacts(5)->create();

    expect($group->contacts)->toHaveCount(5);

    $group->contacts()->detach();

    expect($group->fresh()->contacts)->toHaveCount(0);
});

it('can count contacts in a group', function () {
    $group = Group::factory()->withContacts(10)->create();

    expect($group->contacts()->count())->toBe(10);
});
```

### `tests/Unit/Models/BlacklistedNumberTest.php`

```php
use App\Models\BlacklistedNumber;

it('checks if a number is blacklisted', function () {
    $mobile = '+639171234567';
    
    expect(BlacklistedNumber::isBlacklisted($mobile))->toBeFalse();
    
    BlacklistedNumber::factory()->create(['mobile' => $mobile]);
    
    expect(BlacklistedNumber::isBlacklisted($mobile))->toBeTrue();
});

it('normalizes phone numbers when checking', function () {
    BlacklistedNumber::factory()->create(['mobile' => '+639171234567']);
    
    expect(BlacklistedNumber::isBlacklisted('0917 123 4567'))->toBeTrue()
        ->and(BlacklistedNumber::isBlacklisted('09171234567'))->toBeTrue()
        ->and(BlacklistedNumber::isBlacklisted('+639171234567'))->toBeTrue();
});

it('adds a number to blacklist', function () {
    $mobile = '0917 123 4567';
    
    $blacklisted = BlacklistedNumber::add($mobile, 'Test reason', 'Test User');
    
    expect($blacklisted->mobile)->toBe('+639171234567')
        ->and($blacklisted->reason)->toBe('Test reason')
        ->and($blacklisted->added_by)->toBe('Test User');
});

it('prevents duplicate blacklist entries', function () {
    $mobile = '+639171234567';
    
    BlacklistedNumber::add($mobile, 'First reason');
    BlacklistedNumber::add($mobile, 'Second reason'); // Should not create duplicate
    
    expect(BlacklistedNumber::where('mobile', $mobile)->count())->toBe(1);
});

it('removes a number from blacklist', function () {
    $mobile = '+639171234567';
    BlacklistedNumber::factory()->create(['mobile' => $mobile]);
    
    expect(BlacklistedNumber::isBlacklisted($mobile))->toBeTrue();
    
    $removed = BlacklistedNumber::remove($mobile);
    
    expect($removed)->toBeTrue()
        ->and(BlacklistedNumber::isBlacklisted($mobile))->toBeFalse();
});

it('scopes recent blacklisted numbers', function () {
    BlacklistedNumber::factory()->create(['blocked_at' => now()->subDays(10)]);
    BlacklistedNumber::factory()->create(['blocked_at' => now()->subDays(40)]);
    
    $recent = BlacklistedNumber::recent(30)->get();
    
    expect($recent)->toHaveCount(1);
});

it('scopes by reason', function () {
    BlacklistedNumber::factory()->optOut()->create();
    BlacklistedNumber::factory()->complaint()->create();
    BlacklistedNumber::factory()->invalid()->create();
    
    $optOuts = BlacklistedNumber::byReason('opt-out')->get();
    $complaints = BlacklistedNumber::byReason('complaint')->get();
    
    expect($optOuts)->toHaveCount(1)
        ->and($complaints)->toHaveCount(1);
});
```

---

### `tests/Unit/Models/ScheduledMessageTest.php`

```php
use App\Models\ScheduledMessage;

it('identifies pending messages', function () {
    $message = ScheduledMessage::factory()->pending()->create();

    expect($message->isPending())->toBeTrue()
        ->and($message->status)->toBe('pending');
});

it('identifies sent messages', function () {
    $message = ScheduledMessage::factory()->sent()->create();

    expect($message->isPending())->toBeFalse()
        ->and($message->status)->toBe('sent');
});

it('determines if message is cancellable', function () {
    $pending = ScheduledMessage::factory()->pending()->create();
    $sent = ScheduledMessage::factory()->sent()->create();

    expect($pending->isCancellable())->toBeTrue()
        ->and($sent->isCancellable())->toBeFalse();
});

it('determines if message is editable', function () {
    $futureMessage = ScheduledMessage::factory()->create([
        'status' => 'pending',
        'scheduled_at' => now()->addHour(),
    ]);

    $pastMessage = ScheduledMessage::factory()->create([
        'status' => 'pending',
        'scheduled_at' => now()->subHour(),
    ]);

    expect($futureMessage->isEditable())->toBeTrue()
        ->and($pastMessage->isEditable())->toBeFalse();
});

it('generates recipient summary', function () {
    $message = ScheduledMessage::factory()->create([
        'recipient_type' => 'numbers',
        'recipient_data' => ['numbers' => ['+639171234567', '+639189876543']],
    ]);

    expect($message->recipient_summary)->toContain('2 number(s)');
});

it('scopes ready messages', function () {
    ScheduledMessage::factory()->create([
        'status' => 'pending',
        'scheduled_at' => now()->subMinute(),
    ]);

    ScheduledMessage::factory()->create([
        'status' => 'pending',
        'scheduled_at' => now()->addHour(),
    ]);

    $ready = ScheduledMessage::ready()->get();

    expect($ready)->toHaveCount(1);
});
```

---

## Action Tests

### `tests/Feature/Actions/SendToMultipleRecipientsTest.php`

```php
use App\Actions\SendToMultipleRecipients;
use App\Jobs\SendSMSJob;
use Illuminate\Support\Facades\Queue;

beforeEach(function () {
    Queue::fake();
    actingAsAdmin();
});

it('sends SMS to multiple recipients', function () {
    $action = new SendToMultipleRecipients();

    $result = $action->handle(
        '+639171234567,+639189876543',
        'Test message',
        'TXTCMDR'
    );

    expect($result['status'])->toBe('queued')
        ->and($result['count'])->toBe(2)
        ->and($result['recipients'])->toHaveCount(2);

    Queue::assertPushed(SendSMSJob::class, 2);
});

it('handles invalid phone numbers gracefully', function () {
    $action = new SendToMultipleRecipients();

    $result = $action->handle(
        '+639171234567,invalid-number',
        'Test message',
        'TXTCMDR'
    );

    expect($result['count'])->toBe(1)
        ->and($result['invalid_count'])->toBe(1);
});

it('can be invoked as controller', function () {
    $response = $this->postJson('/api/send', [
        'recipients' => '+639171234567,+639189876543',
        'message' => 'Test message',
        'sender_id' => 'TXTCMDR',
    ]);

    $response->assertStatus(200)
        ->assertJson([
            'status' => 'queued',
            'count' => 2,
        ]);

    Queue::assertPushed(SendSMSJob::class, 2);
});

it('validates required fields', function () {
    $response = $this->postJson('/api/send', [
        'message' => 'Test message',
    ]);

    $response->assertStatus(422)
        ->assertJsonValidationErrors(['recipients']);
});
```

### `tests/Feature/Actions/ScheduleMessageTest.php`

```php
use App\Actions\ScheduleMessage;
use App\Models\ScheduledMessage;
use Carbon\Carbon;

beforeEach(fn () => actingAsAdmin());

it('schedules a message for future delivery', function () {
    $action = new ScheduleMessage();
    $scheduledAt = Carbon::now()->addHour();

    $message = $action->handle(
        '+639171234567',
        'Test message',
        $scheduledAt,
        'TXTCMDR'
    );

    expect($message)->toBeInstanceOf(ScheduledMessage::class)
        ->and($message->status)->toBe('pending')
        ->and($message->scheduled_at->format('Y-m-d H:i'))->toBe($scheduledAt->format('Y-m-d H:i'));
});

it('parses recipients correctly', function () {
    $group = createGroup();
    $group->contacts()->attach(createContact()->id);

    $action = new ScheduleMessage();
    $message = $action->handle(
        $group->name,
        'Test message',
        Carbon::now()->addHour()
    );

    expect($message->recipient_type)->toBe('group')
        ->and($message->total_recipients)->toBeGreaterThan(0);
});

it('validates scheduled_at is in the future', function () {
    $response = $this->postJson('/api/send/schedule', [
        'recipients' => '+639171234567',
        'message' => 'Test message',
        'scheduled_at' => Carbon::now()->subHour()->toIso8601String(),
    ]);

    $response->assertStatus(422)
        ->assertJsonValidationErrors(['scheduled_at']);
});
```

### `tests/Feature/Actions/Groups/CreateGroupTest.php`

```php
use App\Actions\Groups\CreateGroup;
use App\Models\Group;

beforeEach(fn () => actingAsAdmin());

it('creates a new group', function () {
    $action = new CreateGroup();

    $group = $action->handle('Test Group', 'Test Description');

    expect($group)->toBeInstanceOf(Group::class)
        ->and($group->name)->toBe('Test Group')
        ->and($group->description)->toBe('Test Description');
});

it('can be invoked as controller', function () {
    $response = $this->postJson('/api/groups', [
        'name' => 'New Group',
        'description' => 'Group Description',
    ]);

    $response->assertStatus(201)
        ->assertJson([
            'name' => 'New Group',
            'description' => 'Group Description',
        ]);

    $this->assertDatabaseHas('groups', [
        'name' => 'New Group',
    ]);
});

it('validates unique group names', function () {
    createGroup(['name' => 'Existing Group']);

    $response = $this->postJson('/api/groups', [
        'name' => 'Existing Group',
    ]);

    $response->assertStatus(422)
        ->assertJsonValidationErrors(['name']);
});
```

### `tests/Feature/Actions/Blacklist/AddToBlacklistTest.php`

```php
use App\Actions\Blacklist\AddToBlacklist;
use App\Models\BlacklistedNumber;

beforeEach(fn () => actingAsAdmin());

it('adds a number to blacklist', function () {
    $action = new AddToBlacklist();
    
    $blacklisted = $action->handle(
        '0917 123 4567',
        'User opted out',
        'Admin User'
    );
    
    expect($blacklisted)->toBeInstanceOf(BlacklistedNumber::class)
        ->and($blacklisted->mobile)->toBe('+639171234567')
        ->and($blacklisted->reason)->toBe('User opted out')
        ->and($blacklisted->added_by)->toBe('Admin User');
});

it('can be invoked as controller', function () {
    $response = $this->postJson('/api/blacklist', [
        'mobile' => '0917 123 4567',
        'reason' => 'User complaint',
    ]);
    
    $response->assertStatus(201)
        ->assertJson([
            'message' => 'Number added to blacklist',
        ]);
    
    $this->assertDatabaseHas('blacklisted_numbers', [
        'mobile' => '+639171234567',
        'reason' => 'User complaint',
    ]);
});

it('validates phone number format', function () {
    $response = $this->postJson('/api/blacklist', [
        'mobile' => 'invalid-phone',
        'reason' => 'Test',
    ]);
    
    $response->assertStatus(422)
        ->assertJsonValidationErrors(['mobile']);
});

it('prevents duplicate entries', function () {
    BlacklistedNumber::factory()->create(['mobile' => '+639171234567']);
    
    $response = $this->postJson('/api/blacklist', [
        'mobile' => '0917 123 4567',
        'reason' => 'Duplicate test',
    ]);
    
    $response->assertStatus(201); // Should still succeed
    
    expect(BlacklistedNumber::where('mobile', '+639171234567')->count())->toBe(1);
});
```

---

### `tests/Feature/Actions/Blacklist/RemoveFromBlacklistTest.php`

```php
use App\Actions\Blacklist\RemoveFromBlacklist;
use App\Models\BlacklistedNumber;

beforeEach(fn () => actingAsAdmin());

it('removes a number from blacklist', function () {
    $mobile = '+639171234567';
    BlacklistedNumber::factory()->create(['mobile' => $mobile]);
    
    $action = new RemoveFromBlacklist();
    $result = $action->handle($mobile);
    
    expect($result)->toBeTrue()
        ->and(BlacklistedNumber::isBlacklisted($mobile))->toBeFalse();
});

it('returns false for non-existent numbers', function () {
    $action = new RemoveFromBlacklist();
    $result = $action->handle('+639171234567');
    
    expect($result)->toBeFalse();
});

it('can be invoked as controller', function () {
    $mobile = '+639171234567';
    BlacklistedNumber::factory()->create(['mobile' => $mobile]);
    
    $response = $this->deleteJson('/api/blacklist', [
        'mobile' => '0917 123 4567',
    ]);
    
    $response->assertStatus(200)
        ->assertJson([
            'message' => 'Number removed from blacklist',
        ]);
    
    $this->assertDatabaseMissing('blacklisted_numbers', [
        'mobile' => $mobile,
    ]);
});

it('returns 404 for non-existent numbers via controller', function () {
    $response = $this->deleteJson('/api/blacklist', [
        'mobile' => '0917 123 4567',
    ]);
    
    $response->assertStatus(404)
        ->assertJson([
            'message' => 'Number not found in blacklist',
        ]);
});
```

---

### `tests/Feature/Actions/Blacklist/ListBlacklistedNumbersTest.php`

```php
use App\Actions\Blacklist\ListBlacklistedNumbers;
use App\Models\BlacklistedNumber;

beforeEach(fn () => actingAsAdmin());

it('lists all blacklisted numbers', function () {
    BlacklistedNumber::factory()->count(5)->create();
    
    $action = new ListBlacklistedNumbers();
    $result = $action->handle();
    
    expect($result->total())->toBe(5);
});

it('filters by search term', function () {
    BlacklistedNumber::factory()->create(['mobile' => '+639171234567']);
    BlacklistedNumber::factory()->create(['mobile' => '+639189876543']);
    
    $action = new ListBlacklistedNumbers();
    $result = $action->handle('0917');
    
    expect($result->total())->toBe(1);
});

it('filters by reason', function () {
    BlacklistedNumber::factory()->optOut()->count(3)->create();
    BlacklistedNumber::factory()->complaint()->count(2)->create();
    
    $action = new ListBlacklistedNumbers();
    $result = $action->handle(null, 'opt-out');
    
    expect($result->total())->toBe(3);
});

it('can be invoked as controller', function () {
    BlacklistedNumber::factory()->count(10)->create();
    
    $response = $this->getJson('/api/blacklist');
    
    $response->assertStatus(200)
        ->assertJsonStructure([
            'data',
            'current_page',
            'total',
        ])
        ->assertJsonCount(10, 'data');
});
```

---

### `tests/Feature/Actions/Blacklist/CheckIfBlacklistedTest.php`

```php
use App\Actions\Blacklist\CheckIfBlacklisted;
use App\Models\BlacklistedNumber;

beforeEach(fn () => actingAsAdmin());

it('checks if number is blacklisted', function () {
    $mobile = '+639171234567';
    $blacklisted = BlacklistedNumber::factory()->create(['mobile' => $mobile]);
    
    $action = new CheckIfBlacklisted();
    $result = $action->handle($mobile);
    
    expect($result['is_blacklisted'])->toBeTrue()
        ->and($result['mobile'])->toBe($mobile)
        ->and($result['record']->id)->toBe($blacklisted->id);
});

it('returns false for non-blacklisted numbers', function () {
    $mobile = '+639171234567';
    
    $action = new CheckIfBlacklisted();
    $result = $action->handle($mobile);
    
    expect($result['is_blacklisted'])->toBeFalse()
        ->and($result['mobile'])->toBe($mobile)
        ->and($result['record'])->toBeNull();
});

it('can be invoked as controller', function () {
    $mobile = '+639171234567';
    BlacklistedNumber::factory()->create(['mobile' => $mobile]);
    
    $response = $this->postJson('/api/blacklist/check', [
        'mobile' => '0917 123 4567',
    ]);
    
    $response->assertStatus(200)
        ->assertJson([
            'is_blacklisted' => true,
            'mobile' => $mobile,
        ])
        ->assertJsonStructure([
            'record' => ['id', 'mobile', 'reason'],
        ]);
});
```

---

### `tests/Feature/Actions/Contacts/AddContactToGroupTest.php`

```php
use App\Actions\Contacts\AddContactToGroup;
use App\Models\Contact;

beforeEach(fn () => actingAsAdmin());

it('adds a contact to a group', function () {
    $group = createGroup();
    $action = new AddContactToGroup();

    $contact = $action->handle(
        $group->id,
        '0917 123 4567',
        'Juan Dela Cruz',
        ['leader', 'volunteer']
    );

    expect($contact)->toBeInstanceOf(Contact::class)
        ->and($contact->getMeta('name'))->toBe('Juan Dela Cruz')
        ->and($contact->getTags())->toContain('leader', 'volunteer')
        ->and($group->contacts->contains($contact))->toBeTrue();
});

it('validates phone number format', function () {
    $group = createGroup();

    $response = $this->postJson("/api/groups/{$group->id}/contacts", [
        'mobile' => 'invalid-phone',
        'name' => 'Test User',
    ]);

    $response->assertStatus(422)
        ->assertJsonValidationErrors(['mobile']);
});
```

---

## Controller Tests

### `tests/Feature/Controllers/DashboardControllerTest.php`

```php
use App\Models\Contact;
use App\Models\Group;
use App\Models\SenderID;

beforeEach(fn () => actingAsAdmin());

it('shows dashboard to authenticated users', function () {
    Contact::factory()->count(5)->create();
    Group::factory()->count(3)->create();
    SenderID::factory()->default()->create();

    $response = $this->get('/');

    $response->assertStatus(200)
        ->assertInertia(fn ($page) =>
            $page->component('dashboard')
                ->has('contacts')
                ->has('groups')
                ->has('senderIds')
        );
});

it('redirects guests to login', function () {
    auth()->logout();

    $response = $this->get('/');

    $response->assertRedirect('/login');
});

it('pre-fills recipient when passed via query string', function () {
    $response = $this->get('/?recipient=Juan');

    $response->assertStatus(200)
        ->assertInertia(fn ($page) =>
            $page->has('prefilledRecipient', 'Juan')
        );
});
```

### `tests/Feature/Controllers/Auth/LoginControllerTest.php`

```php
use App\Models\User;
use Illuminate\Support\Facades\Hash;

it('shows login form to guests', function () {
    $response = $this->get('/login');

    $response->assertStatus(200)
        ->assertInertia(fn ($page) => $page->component('auth/login'));
});

it('authenticates user with valid credentials', function () {
    $user = User::factory()->create([
        'email' => 'admin@txtcmdr.local',
        'password' => Hash::make('password'),
    ]);

    $response = $this->post('/login', [
        'email' => 'admin@txtcmdr.local',
        'password' => 'password',
    ]);

    $response->assertRedirect('/')
        ->assertSessionHasNoErrors();

    expect(auth()->user())->toBeInstanceOf(User::class)
        ->and(auth()->id())->toBe($user->id);
});

it('rejects invalid credentials', function () {
    User::factory()->create([
        'email' => 'admin@txtcmdr.local',
        'password' => Hash::make('password'),
    ]);

    $response = $this->post('/login', [
        'email' => 'admin@txtcmdr.local',
        'password' => 'wrong-password',
    ]);

    $response->assertSessionHasErrors('email');
    expect(auth()->check())->toBeFalse();
});

it('logs out authenticated user', function () {
    actingAsAdmin();

    $response = $this->post('/logout');

    $response->assertRedirect('/login');
    expect(auth()->check())->toBeFalse();
});

it('remembers user when remember flag is set', function () {
    $user = User::factory()->create([
        'email' => 'admin@txtcmdr.local',
        'password' => Hash::make('password'),
    ]);

    $response = $this->post('/login', [
        'email' => 'admin@txtcmdr.local',
        'password' => 'password',
        'remember' => true,
    ]);

    $response->assertRedirect('/');
    expect(auth()->viaRemember())->toBeTrue();
});
```

### `tests/Feature/Controllers/ContactControllerTest.php`

```php
use App\Models\Contact;
use App\Models\Group;

beforeEach(fn () => actingAsAdmin());

it('shows contacts index page', function () {
    Contact::factory()->count(10)->create();
    Group::factory()->count(3)->create();

    $response = $this->get('/contacts');

    $response->assertStatus(200)
        ->assertInertia(fn ($page) =>
            $page->component('contacts/index')
                ->has('contacts')
                ->has('groups')
        );
});

it('filters contacts by search query', function () {
    Contact::factory()->create([
        'extra_attributes' => ['name' => 'Juan Dela Cruz'],
    ]);

    Contact::factory()->create([
        'extra_attributes' => ['name' => 'Maria Santos'],
    ]);

    $response = $this->get('/contacts?search=Juan');

    $response->assertStatus(200)
        ->assertInertia(fn ($page) =>
            $page->has('contacts.data', 1)
                ->where('search', 'Juan')
        );
});

it('shows contact edit page', function () {
    $contact = createContact();

    $response = $this->get("/contacts/{$contact->id}/edit");

    $response->assertStatus(200)
        ->assertInertia(fn ($page) =>
            $page->component('contacts/edit')
                ->has('contact')
                ->has('groups')
        );
});
```

### `tests/Feature/Controllers/LogControllerTest.php`

```php
use App\Models\ScheduledMessage;

beforeEach(fn () => actingAsAdmin());

it('shows logs index page', function () {
    ScheduledMessage::factory()->count(10)->create();

    $response = $this->get('/logs');

    $response->assertStatus(200)
        ->assertInertia(fn ($page) =>
            $page->component('logs/index')
                ->has('messages')
                ->has('filters')
        );
});

it('filters messages by status', function () {
    ScheduledMessage::factory()->pending()->create();
    ScheduledMessage::factory()->sent()->count(2)->create();

    $response = $this->get('/logs?status=sent');

    $response->assertStatus(200)
        ->assertInertia(fn ($page) =>
            $page->has('messages.data', 2)
                ->where('filters.status', 'sent')
        );
});

it('filters messages by date', function () {
    ScheduledMessage::factory()->create([
        'scheduled_at' => now(),
    ]);

    ScheduledMessage::factory()->create([
        'scheduled_at' => now()->subWeek(),
    ]);

    $response = $this->get('/logs?date=today');

    $response->assertStatus(200)
        ->assertInertia(fn ($page) =>
            $page->has('messages.data', 1)
        );
});
```

### `tests/Feature/Controllers/SettingsControllerTest.php`

```php
use App\Models\SenderID;

beforeEach(fn () => actingAsAdmin());

it('shows settings page', function () {
    SenderID::factory()->count(3)->create();

    $response = $this->get('/settings');

    $response->assertStatus(200)
        ->assertInertia(fn ($page) =>
            $page->component('settings/index')
                ->has('user')
                ->has('senderIds')
                ->has('gateway')
        );
});

it('includes gateway information', function () {
    $response = $this->get('/settings');

    $response->assertInertia(fn ($page) =>
        $page->where('gateway.provider', 'engagespark')
            ->has('gateway.balance')
    );
});
```

---

## Service Tests

### `tests/Unit/Services/SMSServiceTest.php`

```php
use App\Services\SMSService;
use Illuminate\Support\Facades\Log;
use LBHurtado\SMS\Facades\SMS;

beforeEach(function () {
    SMS::fake();
});

it('sends SMS successfully', function () {
    $service = new SMSService();

    $result = $service->send('+639171234567', 'Test message', 'TXTCMDR');

    expect($result)->toBeTrue();
    SMS::assertSent('+639171234567');
});

it('logs SMS sending', function () {
    Log::spy();

    $service = new SMSService();
    $service->send('+639171234567', 'Test message', 'TXTCMDR');

    Log::shouldHaveReceived('info')
        ->with('SMS sent', [
            'mobile' => '+639171234567',
            'sender_id' => 'TXTCMDR',
        ]);
});

it('handles SMS sending errors', function () {
    SMS::shouldReceive('channel->from->to->content->send')
        ->andThrow(new \Exception('API Error'));

    $service = new SMSService();
    $result = $service->send('+639171234567', 'Test message', 'TXTCMDR');

    expect($result)->toBeFalse();
});

it('sends SMS with airtime topup', function () {
    $service = new SMSService();

    $result = $service->sendWithTopup(
        '+639171234567',
        'Test message',
        'TXTCMDR',
        25
    );

    expect($result)->toBeTrue();
});
```

---

## Seeder Tests

### `tests/Feature/Seeders/AdminUserSeederTest.php`

```php
use App\Models\User;
use Database\Seeders\AdminUserSeeder;
use Illuminate\Support\Facades\Hash;

it('creates admin user', function () {
    $seeder = new AdminUserSeeder();
    $seeder->run();

    $admin = User::where('email', 'admin@txtcmdr.local')->first();

    expect($admin)->not->toBeNull()
        ->and($admin->name)->toBe('Admin')
        ->and($admin->role)->toBe('admin')
        ->and(Hash::check('password', $admin->password))->toBeTrue();
});

it('does not duplicate admin user if already exists', function () {
    User::factory()->create([
        'email' => 'admin@txtcmdr.local',
        'name' => 'Existing Admin',
    ]);

    $seeder = new AdminUserSeeder();
    $seeder->run();

    expect(User::where('email', 'admin@txtcmdr.local')->count())->toBe(1);

    $admin = User::where('email', 'admin@txtcmdr.local')->first();
    expect($admin->name)->toBe('Existing Admin'); // Should not be overwritten
});

it('can authenticate with seeded admin credentials', function () {
    $seeder = new AdminUserSeeder();
    $seeder->run();

    $response = $this->post('/login', [
        'email' => 'admin@txtcmdr.local',
        'password' => 'password',
    ]);

    $response->assertRedirect('/');
    expect(auth()->check())->toBeTrue();
});
```

---

## Integration Tests

### `tests/Feature/SMS/SMSWorkflowTest.php`

```php
use App\Actions\ScheduleMessage;
use App\Jobs\ProcessScheduledMessage;
use App\Jobs\SendSMSJob;
use App\Models\ScheduledMessage;
use Carbon\Carbon;
use Illuminate\Support\Facades\Queue;

beforeEach(function () {
    Queue::fake();
    actingAsAdmin();
});

it('completes full SMS scheduling workflow', function () {
    // Schedule a message
    $action = new ScheduleMessage();
    $message = $action->handle(
        '+639171234567,+639189876543',
        'Test message',
        Carbon::now()->addMinute()
    );

    expect($message->status)->toBe('pending');

    // Fast-forward time
    $this->travel(2)->minutes();

    // Process scheduled message
    ProcessScheduledMessage::dispatch($message->id);

    Queue::assertPushed(SendSMSJob::class, 2);
});

it('handles group broadcasting workflow', function () {
    $group = createGroup();
    $group->contacts()->attach(
        createContact(['mobile' => '09171234567'])->id
    );
    $group->contacts()->attach(
        createContact(['mobile' => '09189876543'])->id
    );

    $action = new ScheduleMessage();
    $message = $action->handle(
        $group->name,
        'Group message',
        Carbon::now()->addMinute()
    );

    expect($message->recipient_type)->toBe('group')
        ->and($message->total_recipients)->toBe(2);

    $this->travel(2)->minutes();

    ProcessScheduledMessage::dispatch($message->id);

    Queue::assertPushed(SendSMSJob::class, 2);

    $message->refresh();
    expect($message->status)->toBe('sent')
        ->and($message->sent_count)->toBe(2);
});
```

### `tests/Feature/Blacklist/BlacklistMiddlewareTest.php`

```php
use App\Jobs\SendSMSJob;
use App\Models\BlacklistedNumber;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Log;

beforeEach(function () {
    Queue::fake();
    Log::fake();
});

it('blocks SMS to blacklisted number', function () {
    $mobile = '+639171234567';
    BlacklistedNumber::factory()->create(['mobile' => $mobile]);
    
    SendSMSJob::dispatch($mobile, 'Test message', 'TXTCMDR');
    
    // Job should be intercepted and not executed
    Queue::assertNothingPushed();
    
    Log::assertLogged('warning', function ($message, $context) use ($mobile) {
        return $context['mobile'] === $mobile && 
               str_contains($message, 'SMS blocked - number is blacklisted');
    });
});

it('allows SMS to non-blacklisted number', function () {
    $mobile = '+639171234567';
    
    SendSMSJob::dispatch($mobile, 'Test message', 'TXTCMDR');
    
    Queue::assertPushed(SendSMSJob::class, function ($job) use ($mobile) {
        return $job->mobile === $mobile;
    });
});

it('blocks group broadcast to blacklisted numbers', function () {
    $group = createGroup();
    
    $contact1 = createContact(['mobile' => '09171234567']);
    $contact2 = createContact(['mobile' => '09189876543']);
    
    $group->contacts()->attach([$contact1->id, $contact2->id]);
    
    // Blacklist one number
    BlacklistedNumber::factory()->create(['mobile' => $contact1->e164_mobile]);
    
    BroadcastToGroupJob::dispatch($group->id, 'Test message', 'TXTCMDR');
    
    // Should only dispatch for non-blacklisted number
    Queue::assertPushed(SendSMSJob::class, 1);
    
    Queue::assertPushed(SendSMSJob::class, function ($job) use ($contact2) {
        return $job->mobile === $contact2->e164_mobile;
    });
});

it('handles middleware for jobs without mobile property', function () {
    // Create a job without mobile property - should pass through
    $job = new class implements \Illuminate\Contracts\Queue\ShouldQueue {
        use \Illuminate\Foundation\Bus\Dispatchable;
        use \Illuminate\Queue\InteractsWithQueue;
        use \Illuminate\Queue\SerializesModels;
        
        public function handle() {
            // Do nothing
        }
    };
    
    dispatch($job);
    
    // Should not cause any errors
    expect(true)->toBeTrue();
});
```

---

### `tests/Feature/Contact/ContactManagementTest.php`

```php
use App\Actions\Contacts\AddContactToGroup;
use App\Actions\Contacts\UpdateContactInGroup;
use App\Actions\Contacts\DeleteContactFromGroup;

beforeEach(fn () => actingAsAdmin());

it('manages contact lifecycle in a group', function () {
    $group = createGroup(['name' => 'Test Group']);

    // Add contact
    $addAction = new AddContactToGroup();
    $contact = $addAction->handle(
        $group->id,
        '0917 123 4567',
        'Juan Dela Cruz',
        ['leader']
    );

    expect($group->contacts->contains($contact))->toBeTrue();

    // Update contact
    $updateAction = new UpdateContactInGroup();
    $updated = $updateAction->handle(
        $group->id,
        $contact->id,
        '0918 765 4321',
        'Juan Updated',
        ['leader', 'volunteer']
    );

    expect($updated->getMeta('name'))->toBe('Juan Updated')
        ->and($updated->getTags())->toContain('volunteer');

    // Delete contact
    $deleteAction = new DeleteContactFromGroup();
    $deleteAction->handle($group->id, $contact->id);

    expect($group->fresh()->contacts->contains($contact))->toBeFalse();
});
```

---

## Running Tests

### Run all tests
```bash
php artisan test
# or
vendor/bin/pest
```

### Run specific test file
```bash
php artisan test tests/Feature/Actions/SendToMultipleRecipientsTest.php
# or
vendor/bin/pest tests/Feature/Actions/SendToMultipleRecipientsTest.php
```

### Run tests with coverage
```bash
php artisan test --coverage
# or
vendor/bin/pest --coverage
```

### Run tests in parallel
```bash
php artisan test --parallel
# or
vendor/bin/pest --parallel
```

### Run specific test by name
```bash
php artisan test --filter="it sends SMS to multiple recipients"
```

---

## Test Organization

```
tests/
├── Feature/
│   ├── Actions/
│   │   ├── SendToMultipleRecipientsTest.php
│   │   ├── SendToMultipleGroupsTest.php
│   │   ├── ScheduleMessageTest.php
│   │   ├── Groups/
│   │   │   ├── CreateGroupTest.php
│   │   │   ├── UpdateGroupTest.php
│   │   │   └── DeleteGroupTest.php
│   │   ├── Contacts/
│   │   │   ├── AddContactToGroupTest.php
│   │   │   ├── UpdateContactInGroupTest.php
│   │   │   └── DeleteContactFromGroupTest.php
│   │   └── Blacklist/
│   │       ├── AddToBlacklistTest.php
│   │       ├── RemoveFromBlacklistTest.php
│   │       ├── ListBlacklistedNumbersTest.php
│   │       └── CheckIfBlacklistedTest.php
│   ├── Controllers/
│   │   ├── DashboardControllerTest.php
│   │   ├── Auth/
│   │   │   └── LoginControllerTest.php
│   │   ├── ContactControllerTest.php
│   │   ├── LogControllerTest.php
│   │   └── SettingsControllerTest.php
│   ├── Seeders/
│   │   └── AdminUserSeederTest.php
│   ├── SMS/
│   │   └── SMSWorkflowTest.php
│   ├── Contact/
│   │   └── ContactManagementTest.php
│   └── Blacklist/
│       └── BlacklistMiddlewareTest.php
├── Unit/
│   ├── Models/
│   │   ├── ContactTest.php
│   │   ├── GroupTest.php
│   │   ├── ScheduledMessageTest.php
│   │   └── BlacklistedNumberTest.php
│   └── Services/
│       └── SMSServiceTest.php
└── Pest.php
```

---

## Related Documentation

- [Backend Services](backend-services.md) - Actions being tested
- [Controller Scaffolding](controller-scaffolding.md) - Controllers being tested
- [Frontend Scaffolding](frontend-scaffolding.md) - TypeScript types for API responses
- [Development Plan](development-plan.md) - Implementation roadmap
